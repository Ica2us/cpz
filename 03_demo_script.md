# 现场演示脚本 / Live Demo Runbook

> 全部命令从 **项目根目录** `C:\Users\wanxy\Desktop\project` 运行。
> Run all commands from the **project root**.
> 演示时长约 4–6 分钟 / ~4–6 minutes.

---

## 0. 赛前检查 / Pre-flight checklist（答辩前一晚务必跑一遍）

- [ ] `python --version` ≥ 3.8（本机为 3.11.2）。
- [ ] 终端用 UTF-8：先执行 `set PYTHONIOENCODING=utf-8`（cmd）/ `$env:PYTHONIOENCODING="utf-8"`（PowerShell），否则中文可能乱码。
- [ ] 生成演示文件：`python defense/demo/prepare_demo.py`。
- [ ] **完整走一遍下面 1–7 步**，确认无报错。
- [ ] 备份：把本文件的“预期输出”截图存好，万一现场环境出问题可直接展示截图（见 §FALLBACK）。
- [ ] GUI 能打开：`python archive_gui.py`。
- [ ] （可选）若要演示 AES-GCM：`pip install cryptography`，并预先验证一次。

---

## 1. 准备演示文件 / Prepare sample files

```bash
python defense/demo/prepare_demo.py
```
预期 / Expected:
```
生成演示文件 / Generating demo files into: ...\defense\demo
  生成 / wrote  sample_report.txt           179,200 bytes
  生成 / wrote  secret.txt                       54 bytes
  生成 / wrote  big_sample.bin            3,407,872 bytes
```
**怎么说 / Say:** “我准备了一个高度重复的报告文本、一个小机密文件，和一个 ~3.4 MB 的二进制文件用于演示流式处理。”

---

## 2. 加密并展示压缩率 / Encrypt + show compression

```bash
python debug.py encrypt -i defense/demo/sample_report.txt -o defense/demo/report.cpz -p Defense@2026
python -c "import os;a=os.path.getsize('defense/demo/sample_report.txt');b=os.path.getsize('defense/demo/report.cpz');print(f'original={a:,}  archive={b:,}  ratio={b/a:.1%}')"
```
预期 / Expected:
```
正在压缩加密: ... -> defense/demo/report.cpz
  格式标识: CPZ1
  压缩算法: lzma
  加密算法: custom
  PBKDF2迭代: 100000
完成。
original=179,200  archive=428  ratio=0.2%
```
**怎么说 / Say:** “先压缩后加密：179 KB 的重复文本压到了 428 字节。压缩在加密前完成，因为加密后的数据接近随机、不可再压。”

---

## 3. 归档看起来是随机的 / The archive looks random

```bash
python -c "d=open('defense/demo/report.cpz','rb').read();print('明文头部 magic+version:',d[:6]);print('密文头部(hex):',d[44:72].hex())"
```
预期 / Expected:
```
明文头部 magic+version: b'CPZ1\x02\x01'
密文头部(hex): a74b7d8c56f14624...   (每次随机不同 / random each run)
```
**怎么说 / Say:** “文件头是明文的——`CPZ1`、版本 2、flags=0x01（lzma+custom）；这些是 **自描述** 信息，并且 **被 HMAC 认证**，改不了。从 salt/nonce 之后就是密文，呈随机分布。”

---

## 4. 解密并验证完全一致 / Decrypt + verify identical

```bash
python debug.py decrypt -i defense/demo/report.cpz -o defense/demo/report.out -p Defense@2026
python -c "print('往返一致 round-trip identical:',open('defense/demo/sample_report.txt','rb').read()==open('defense/demo/report.out','rb').read())"
```
预期 / Expected:
```
正在解密解压: ... 
  压缩算法: lzma (来自文件头)
  加密算法: custom (来自文件头)
  PBKDF2迭代: 100000 (来自文件头)
完成。
往返一致 round-trip identical: True
```
**怎么说 / Say:** “注意解密 **没有指定算法和迭代次数**——它们从文件头自动读取。还原结果与原文逐字节一致。”

---

## 5. 错误口令：安全失败、不留输出 / Wrong password fails safely

```bash
python debug.py decrypt -i defense/demo/report.cpz -o defense/demo/HACK.out -p WrongPass
python -c "import os;print('是否生成 HACK.out:',os.path.exists('defense/demo/HACK.out'))"
```
预期 / Expected:
```
...
操作失败: HMAC 校验失败：密码错误或文件已损坏
是否生成 HACK.out: False
```
**怎么说 / Say:** “错误口令在 **认证阶段** 就失败——而且关键在于：**没有产生任何输出文件**。这是‘先验证后解密’：验证不通过，根本不解密、不落地任何明文。”

---

## 6. 篡改检测 / Tamper detection

```bash
python -c "import shutil;shutil.copy('defense/demo/report.cpz','defense/demo/tampered.cpz');f=open('defense/demo/tampered.cpz','r+b');f.seek(60);b=f.read(1);f.seek(60);f.write(bytes([b[0]^1]));f.close();print('已翻转第60字节的1个bit')"
python debug.py decrypt -i defense/demo/tampered.cpz -o defense/demo/tampered.out -p Defense@2026
```
预期 / Expected:
```
已翻转第60字节的1个bit
...
操作失败: HMAC 校验失败：密码错误或文件已损坏
```
**怎么说 / Say:** “我只翻转了密文里的 **1 个 bit**，HMAC 立刻检测到——这就是完整性保护。注意错误口令和篡改给出的是 **同样** 的提示，攻击者无法区分。”

---

## 7. 图形界面 / GUI demo

```bash
python archive_gui.py
```
操作 / Steps:
1. 模式选「加密」，输入文件选 `defense/demo/big_sample.bin`（~3.4 MB，能看到 **进度条** 走动）。
2. 设口令并确认，点「运行」→ 弹出成功对话框。
3. 模式切「解密」，选刚才的 `.cpz`，**界面自动识别** 压缩/加密算法（来自文件头）。
4. 用对的口令解密成功；故意输错口令 → 弹出错误框、不生成文件。

**怎么说 / Say:** “GUI 复用同一个核心类；加解密在 **后台线程** 执行，进度条实时更新，界面不卡死。解密时算法是从文件头 **自动识别** 的。”

---

## 8.（可选）AES-GCM 工业级后端 / Optional AES-GCM backend

> 需先 `pip install cryptography`。
```bash
python debug.py encrypt -i defense/demo/secret.txt -o defense/demo/secret.cpz -p Defense@2026 --cipher aesgcm
python debug.py decrypt -i defense/demo/secret.cpz -o defense/demo/secret.out -p Defense@2026
```
**怎么说 / Say:** “同一个格式、同一套流程，只是把加密后端切换成经过审计的 AES-256-GCM——这是真实敏感数据应使用的路径。”

---

## FALLBACK：现场出问题怎么办 / If something breaks live

1. **乱码** → 先 `set PYTHONIOENCODING=utf-8` 再重试；或直接演示 GUI（Tk 不受控制台编码影响）。
2. **命令打错/路径错** → 用本文件准备好的命令 **复制粘贴**，不要现场手敲。
3. **环境彻底崩** → 打开预先准备的 **截图/录屏**（赛前按本脚本截好每一步输出）。
4. **时间不够** → 只演示第 2、4、5 步（压缩率 + 往返一致 + 错误口令安全失败），这三步覆盖压缩、机密性、完整性三大卖点。
5. 备一个 **预先生成好的 `report.cpz`**，万一加密步骤出问题也能直接演示解密。

---

## 一键清理 / Cleanup after rehearsal
```bash
python -c "import glob,os;[os.remove(p) for p in glob.glob('defense/demo/*.cpz')+glob.glob('defense/demo/*.out')]"
```
（保留 `prepare_demo.py` 与生成的样本文件 / keeps the generator and sample inputs.）
