# 幻灯片大纲 / Slides Outline

> 约 15 页，~15–20 分钟。每页给出：**标题** + **要点 (bullets)** + **讲稿提示 (speaker notes)**。
> ~15 slides. Each: title + bullets + speaker notes. 双语标题，正文可按需精简。

---

### Slide 1 — 封面 / Title
- 设计与实现一个压缩加密归档工具 / Design and Implementation of a Compression-Encryption Archive Tool
- 姓名 / 导师 / 日期 / 学院
- **讲稿：** 一句话定位——“一个自描述、带完整性保护的文件归档格式，支持从教学实现到工业级 AEAD 的可切换加密。”

### Slide 2 — 背景与动机 / Motivation
- 文件常需 **既压缩又加密**（备份、传输、归档）。
- 通用容器（如传统 ZIP 加密）常 **完整性弱**、参数不透明、加密实现良莠不齐。
- 想 **完整掌控** 压缩 + 加密 + 完整性这条链路，并理解其中的密码学。
- **讲稿：** 强调“理解与掌控”，这是课题的学习目标，而非重复造轮子。

### Slide 3 — 目标与贡献 / Goals & Contributions
- 机密性、完整性/真实性、流式、原子性、零依赖默认、自描述带版本。
- **贡献：** 自描述格式 + 正确的认证加密工程组合 + 可切换的工业级 AEAD。
- **讲稿：** 明确“贡献不是发明新密码，而是正确地组合已知原语”。

### Slide 4 — 系统总览 / System Overview
- 数据流图：`明文 → 压缩 → 加密 → 认证 → 归档`（放技术文档 §3 的流程图）。
- 两个后端：`custom`（零依赖、教学）/ `aesgcm`（审计库、工业级）。
- **讲稿：** 顺着箭头讲一遍，点出“先压缩后加密”的原因。

### Slide 5 — 文件格式 / File Format (v2)
- 放 §4 的字节表：magic·version·flags·iterations·salt·nonce·ciphertext·trailer。
- 亮点：自描述（算法写在 flags）、带版本、**头部受认证**、无需存长度。
- **讲稿：** “解密端不需要记住用了什么算法——文件自己说明。”

### Slide 6 — 密钥派生 / Key Derivation (PBKDF2)
- `PBKDF2-HMAC-SHA256(pw, salt, iters) → 64B → enc_key | mac_key`。
- 每文件随机盐；**密钥分离**（加密/认证不同密钥）。
- 现状 100k，建议 600k（OWASP 2025）/ Argon2id（改进方向）。
- **讲稿：** 主动说出“100k 偏低，我会调到 600k”，把潜在质疑变成主动展示。

### Slide 7 — 加密：两个后端 / Encryption: Two Backends
- custom：`KS_i = SHA256(key‖nonce‖i)`，计数器模式流密码（XOR）。
- aesgcm：AES-256-GCM（NIST 标准 AEAD，借助 AES-NI）。
- **诚实定位：** custom 为教学实现；真实用 aesgcm。
- **讲稿：** 这页就埋下“黄金法则”的伏笔，主动框定 custom 的性质。

### Slide 8 — 完整性 / Integrity: Encrypt-then-MAC
- `tag = HMAC-SHA256(mac_key, header‖ciphertext)`；头部一并认证。
- EtM（Bellare–Namprempre 2000）：通用地达到 IND-CCA + INT-CTXT。
- **先验证后解密**：错误口令/篡改绝不落地明文；常数时间比较。
- **讲稿：** 这是最能体现“做对了”的一页，重点讲。

### Slide 9 — 流式与原子写入 / Streaming & Atomic Writes
- 1 MiB 缓冲、内存恒定、支持任意大小；进度回调驱动 GUI。
- 原子写入：临时文件 → `os.replace`；崩溃/失败不留脏数据。
- **讲稿：** 把“工程健壮性”作为卖点。

### Slide 10 — 威胁模型 / Threat Model
- 保护：机密性、完整性、错误口令检测（对抗离线持密文的攻击者）。
- 不保护：弱口令、侧信道、元数据（大小/可压缩性）、内存密钥、DoS。
- **讲稿：** 清晰的边界说明是加分项，体现安全思维成熟。

### Slide 11 — 实现与测试 / Implementation & Testing
- Python 3，标准库为主；CLI + Tkinter GUI 复用同一核心类。
- 修复的 4 个缺陷：迭代次数被忽略、算法未存、HMAC 占用内存、未认证明文落地。
- 测试：往返、空/大文件、非默认迭代、自动识别、错误口令、篡改、原子性——全通过。
- **讲稿：** 用“修复的真实 bug”证明你深入理解了系统。

### Slide 12 — 与现有方案对比 / Comparison
- 放 §9 对比表（custom/aesgcm vs ZIP/7-Zip/gpg/age）。
- 定位：教学 + 自描述格式 + 可切换 AEAD + 正确的 AE 组合。
- **讲稿：** 诚实对比，不贬低标准工具。

### Slide 13 — 局限与展望 / Limitations & Future Work
- custom 非标准 → 改用 HMAC-SHA256 密钥流 / ChaCha20。
- PBKDF2 → 600k / Argon2id；常数时间；内存密钥擦除；第三方审计。
- **讲稿：** 主动列局限 = 成熟；每条都配改进方向。

### Slide 14 — 现场演示 / Live Demo
- 压缩率（179 KB→428 B）→ 往返一致 → 错误口令安全失败 → 篡改检测 →（GUI 进度条）。
- **讲稿：** 按 `03_demo_script.md` 走；时间紧只演示压缩率 + 往返 + 错误口令三步。

### Slide 15 — 结论 / Conclusion + Q&A
- 一句话总结贡献：**正确地工程化组合密码学原语**，提供从教学到工业级的可切换归档方案。
- “谢谢，请提问。 / Thank you — questions?”
- **讲稿：** 收尾用问答文档 E3 的金句。

---

## 制作提示 / Production tips
- 图优于字：流程图（Slide 4）、字节表（Slide 5）、对比表（Slide 12）是三张主图。
- 每页 ≤ 5 个要点；代码片段只留关键 2–3 行。
- 备一页 **隐藏附录页**：`SHA256(key‖·)` 的长度扩展与 HMAC 改进（被追问时直接跳转，见问答 A3）。
- 配色朴素；密文 hex、压缩率数字这类“证据”用等宽字体。
