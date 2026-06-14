# 技术文档 / Technical Documentation
### 压缩加密归档工具 / Compression-Encryption Archive Tool

> 配套代码 / Source: `debug.py` (核心 + 命令行 CLI), `archive_gui.py` (图形界面 GUI)
> 格式版本 / Format version: **v2**

---

## 1. 概述 / Overview

本工具将任意文件 **压缩 (compress)** → **加密 (encrypt)** → **认证 (authenticate)**，
封装为一个自描述的 `.cpz` 归档文件。它有两个加密后端：

| 后端 / Backend | 加密 / Cipher | 完整性 / Integrity | 依赖 / Dependency |
|---|---|---|---|
| `custom` (默认) | SHA-256 计数器模式流密码 (SHA-256 counter-mode stream cipher) | HMAC-SHA256（加密后认证 Encrypt-then-MAC） | 无 / none (纯标准库) |
| `aesgcm` (可选) | AES-256-GCM（认证加密 AEAD） | GCM 标签 (built-in tag) | `cryptography` |

设计哲学 / Design philosophy：
- **零依赖默认**（教学价值）：`custom` 后端完全基于 Python 标准库，便于讲解密码学原语如何组合。
- **可升级到工业级**：`aesgcm` 后端使用经过审计的 AES-256-GCM，用于真实敏感数据。
- **自描述格式**：压缩算法、加密算法、迭代次数全部写入文件头，解密时自动读取。

---

## 2. 设计目标 / Design Goals

1. **机密性 (Confidentiality)** — 没有口令无法恢复明文。
2. **完整性与真实性 (Integrity & Authenticity)** — 任何篡改/损坏/错误口令都能被检测。
3. **流式处理 (Streaming)** — 支持任意大小文件，内存占用恒定（缓冲区 1 MiB）。
4. **原子性 (Atomicity)** — 进程中途崩溃不会留下损坏的输出或未认证的明文。
5. **可移植与零依赖 (Portability)** — 默认后端仅用标准库。
6. **自描述与可演进 (Self-describing & versioned)** — 文件头含版本号，便于格式升级。

---

## 3. 体系结构与数据流 / Architecture & Data Flow

```
加密 / ENCRYPT
  口令 password ──PBKDF2(salt, iters)──► enc_key (+ mac_key)
                                              │
  明文 plaintext ──[compress zlib/lzma]──► 压缩流 ──[encrypt]──► 密文 ciphertext
                                                                    │
                              认证 authenticate(header ‖ ciphertext)─┘
                                              │
  归档 = header ‖ ciphertext ‖ trailer        ▼
        (trailer = 32B HMAC  或  16B GCM tag)

解密 / DECRYPT  (custom：先验证后解密 verify-before-decrypt)
  归档 ──[解析文件头 parse header]──► (salt, nonce, 算法, 迭代次数)
  第一遍 pass-1：重算 HMAC(header‖ciphertext) 是否等于 trailer？
        ─ 否 no ─► 拒绝 reject（口令错误 / 已被篡改），不产生任何输出
        ─ 是 yes ─►
  第二遍 pass-2：ciphertext ─[decrypt]─► 压缩流 ─[decompress]─► 明文
                 (先写临时文件 → 验证通过后原子改名 os.replace)
```

**关键顺序 / Key ordering：先压缩后加密 (compress-then-encrypt)。**
压缩在加密之前，因为加密后的数据接近随机、不可再压缩。其安全影响见 §7 与问答文档。

---

## 4. 文件格式规范 / File Format Specification (v2)

固定头部 (fixed prefix) 打包格式 `>4sBBIBB`（大端 big-endian），共 12 字节：

| 偏移 Offset | 长度 Size | 字段 Field   | 说明 / Description |
|---|---|---|---|
| 0  | 4 | `magic`       | 4 字节格式标识，默认 `CPZ1`，可自定义 |
| 4  | 1 | `version`     | 格式版本 = `2` |
| 5  | 1 | `flags`       | bit0–1 压缩算法 (0=zlib,1=lzma)；bit2–3 加密算法 (0=custom,1=aesgcm) |
| 6  | 4 | `iterations`  | PBKDF2 迭代次数 (uint32) |
| 10 | 1 | `salt_len`    | 盐长度 = 16 |
| 11 | 1 | `nonce_len`   | nonce 长度：custom=16, aesgcm=12 |
| 12 | `salt_len`  | `salt`  | 每文件随机盐 (random salt) |
| 12+S | `nonce_len` | `nonce` | 每文件随机 nonce / IV |
| … | C | `ciphertext`  | 压缩后加密的数据流 |
| 末尾-T | T | `trailer` | custom: 32 B HMAC-SHA256；aesgcm: 16 B GCM tag |

派生量 / Derived:
- `header_size = 12 + salt_len + nonce_len`（custom=44, aesgcm=40）
- `ciphertext_len = filesize − header_size − trailer_len` → **无需在文件中存长度字段**，
  消除了旧版“先写占位、回填长度”的 seek 操作。

设计要点 / Notes:
- **版本字节**使格式可演进；遇到未知版本立即给出清晰错误。
- **flags 自描述**：解密端无需用户指定压缩/加密算法，杜绝“记错算法”导致的失败。
- **头部受认证保护**（见 §5.4），防止攻击者篡改算法标志或迭代次数。

---

## 5. 密码学设计 / Cryptographic Design

### 5.1 密钥派生 / Key Derivation — PBKDF2

```
dk = PBKDF2-HMAC-SHA256(password, salt, iterations, dklen=64)
enc_key = dk[0:32]   # 加密密钥
mac_key = dk[32:64]  # HMAC 密钥（仅 custom 后端使用；aesgcm 仅用 enc_key）
```

- 每文件 **16 字节随机盐**，抵御彩虹表与跨文件批量攻击。
- 迭代次数可配置并写入文件头。**当前默认 100,000**。
- ⚠️ **现状与建议**：OWASP（2025）对 PBKDF2-HMAC-SHA256 的推荐为 **≥ 600,000 次**；
  更优选择是内存硬 (memory-hard) 的 **Argon2id**（需第三方库）。建议在答辩前把默认值
  提高到 600,000（演示仍为亚秒级），并在论文中说明 Argon2id 为后续改进方向。详见问答文档 A4。

### 5.2 自带流密码 / Custom Stream Cipher（SHA-256 计数器模式）

构造 / Construction：
```
KS_i = SHA256( enc_key ‖ nonce ‖ uint64_be(i) )     i = 0, 1, 2, …
ciphertext = plaintext  XOR  (KS_0 ‖ KS_1 ‖ …)
```
即把 SHA-256 当作分组函数、以计数器 `i` 生成密钥流的 **计数器模式 (CTR-like)** 同步流密码。
每个 32 字节明文块对应一个 SHA-256 输出块。

安全论证 / Security argument：
- 若 `SHA256(enc_key ‖ ·)` 表现为伪随机函数 (PRF)，则密钥流与随机不可区分，方案达到 **IND-CPA**。
- 每文件 16 字节随机 nonce，使 `(enc_key, nonce)` 复用概率在生日界 ~2⁶⁴ 文件级别可忽略。

**诚实的局限 / Honest caveats（务必主动说明，见问答 A2/A3）：**
1. `SHA256(key ‖ msg)` 并不是从 SHA-256 构造 PRF 的“教科书正确”方式——它在 MAC 场景下有
   **长度扩展 (length-extension)** 弱点。虽然这不直接破坏“密钥流”的机密性，但属于设计气味。
   规范做法是用 **HMAC-SHA256** 作为 PRF。
2. 该构造 **未经标准化、未经审计**，不应宣称为生产级。
3. SHA-256 软件实现比 AES-NI / ChaCha20 慢。

**推荐加固 / Recommended hardening（一行级改动，强烈建议）：**
```
KS_i = HMAC-SHA256( enc_key, nonce ‖ uint64_be(i) )
```
HMAC 是公认的 PRF（PBKDF2、HKDF、NIST SP 800-108 计数器模式 KDF 均如此使用）。
这把“误用了 SHA-256”变成“按 NIST SP 800-108 计数器模式实例化密钥流”，极大增强可辩护性。
（可请我直接实现此改动。）

### 5.3 AES-256-GCM（可选，工业级 / production-grade）

- 经过审计、标准化 (NIST SP 800-38D) 的 **认证加密 (AEAD)**。
- 12 字节随机 nonce；文件头作为 **附加认证数据 (AAD)** 绑定到密文。
- 16 字节认证标签置于文件末尾。
- 通过 `cryptography` 库的流式 `Cipher(AES, GCM)` 接口实现，支持大文件。

### 5.4 完整性 / Integrity — 加密后认证 (Encrypt-then-MAC)

custom 后端采用 **Encrypt-then-MAC (EtM)**：
```
tag = HMAC-SHA256( mac_key, header ‖ ciphertext )
```
- EtM 是 Bellare–Namprempre (2000) 证明能 **通用地** 同时提供 IND-CCA 与 INT-CTXT 的组合顺序。
- **头部一并认证**：算法标志、迭代次数、salt、nonce 都被 MAC 覆盖，攻击者无法降级或篡改参数。
- **先验证后解密 (verify-before-decrypt)**：custom 解密分两遍——先核验 HMAC，
  通过后才解密解压。因此错误口令/篡改 **绝不会** 把可疑数据喂给解压器，也绝不在目标路径落地明文。
- 比较使用 **常数时间** `hmac.compare_digest`，避免计时侧信道。
- aesgcm 后端由 GCM 标签在 `finalize()` 时验证（标签不符抛 `InvalidTag`）。

---

## 6. 流式处理与原子写入 / Streaming & Atomic Writes

- **流式**：以 1 MiB 缓冲区分块读取、压缩、加密、写出，内存占用与文件大小无关。
- **进度回调**：核心方法接受可选 `progress(done, total)` 回调，驱动 GUI 进度条。
- **原子写入**：输出先写到同目录下的临时文件，全部成功后用 `os.replace` 原子改名；
  任何异常都会删除临时文件。因此：
  - 进程崩溃不会留下半截损坏的归档；
  - 错误口令/篡改不会在目标路径留下未认证明文（修复了旧版的安全缺陷）。
- **权衡说明**：custom 解密为“先验证后解密”而读取文件两遍（一遍算 HMAC、一遍解密）。
  这是以额外读 I/O 换取“绝不解压未认证数据”的安全性，对本地文件可接受。

---

## 7. 威胁模型 / Threat Model

**假设的攻击者 / Assumed adversary：** 能读取并任意修改静止的归档文件（离线持有密文），
但无法观测用户主机的内存、键盘输入或运行过程。

**提供的保证 / Guarantees：**
- 机密性：无口令时，除“文件大小”和“大致可压缩性”外无法获知明文。
- 完整性 / 真实性：对密文或头部的任何改动都会被 MAC / GCM 标签检测。
- 错误口令检测：错误口令在认证阶段即失败，不产生输出。

**不在保护范围 / Out of scope：**
- 弱口令 / 可猜口令：PBKDF2 只能 **减缓** 暴力破解，不能阻止。
- 侧信道：除 `compare_digest` 外，Python 实现不保证常数时间。
- 元数据泄露：文件大小、近似熵/可压缩性、以及明文头部中的 magic/版本/算法标志。
- 内存中的密钥/口令未被擦除（Python 限制）。
- “先压缩后加密”泄露明文的近似大小/可压缩性（对静态单文件，CRIME/BREACH 类攻击不适用，见问答 A5）。
- 拒绝服务、密钥管理 / 公钥分发、多用户场景。

---

## 8. 局限性与改进方向 / Limitations & Future Work

主动承认局限是成熟度的体现 / Owning limitations signals maturity:

| 局限 / Limitation | 改进方向 / Future work |
|---|---|
| custom 密码为非标准、未审计构造 | 密钥流改用 HMAC-SHA256（§5.2）；或直接用 ChaCha20 / AES-CTR |
| PBKDF2 默认迭代偏低 (100k) | 提升至 ≥600k；引入 Argon2id（内存硬） |
| Python 实现非常数时间、密钥不擦除 | 关键路径用经审计的本地库；尽量减少口令在内存中的留存 |
| custom 解密两遍读盘 | 大文件可改为单遍 + 解密到临时文件后验证（如 aesgcm 路径） |
| 未做形式化验证 / 第三方审计 | 引入测试向量、与标准实现交叉验证、安全审计 |

---

## 9. 与现有方案对比 / Comparison

| 维度 / Dimension | 本工具 custom | 本工具 aesgcm | ZIP (AES) | 7-Zip | gpg | age |
|---|---|---|---|---|---|---|
| 加密 Cipher | SHA-256 CTR (自研) | AES-256-GCM | AES-CTR | AES-256-CBC | 多种 | ChaCha20-Poly1305 |
| 完整性 Integrity | HMAC (EtM) | GCM (AEAD) | 部分/弱 | 可选 | 签名/MDC | Poly1305 (AEAD) |
| KDF | PBKDF2 | PBKDF2 | PBKDF2 | SHA-256 派生 | s2k | scrypt |
| 流式 Streaming | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| 依赖 Dependency | 无 | cryptography | 库 | 程序 | 程序 | 程序 |
| 标准化 Standardized | ✘（教学） | ✔ | ✔ | ✔ | ✔ | ✔ |

**本工具的贡献定位 / Positioning：** 一个 **自描述、带版本** 的归档格式 + **从零实现** 的
密码学组合（教学价值）+ **零依赖默认** + **可切换的工业级 AEAD** + 正确的认证加密组合（EtM、
先验证后解密、头部认证）。不以“发明新密码”为卖点，而以“正确地工程化组合已知原语”为贡献。

---

## 附录：术语对照 / Glossary

| 中文 | English | 说明 |
|---|---|---|
| 密钥派生 | KDF (Key Derivation Function) | 由口令生成密钥 |
| 盐 | salt | 每文件随机、防彩虹表 |
| 流密码 | stream cipher | 生成密钥流与明文异或 |
| 计数器模式 | CTR mode | 用递增计数器生成密钥流 |
| 认证加密 | AEAD | 同时保证机密性与完整性 |
| 加密后认证 | Encrypt-then-MAC (EtM) | 先加密、再对密文算 MAC |
| 附加认证数据 | AAD | 被认证但不加密的数据（这里指文件头） |
| 伪随机函数 | PRF | 输出与随机不可区分的密钥相关函数 |
| 长度扩展 | length extension | `SHA256(key‖m)` 类构造的弱点 |
