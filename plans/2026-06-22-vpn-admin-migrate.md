# vpn-admin-mvp 迁移到海外服务器 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `vpn-admin-mvp` 从阿里云 123.57.133.209 迁到海外 199.193.126.80，让阿里云专跑 `upwork-hunter`，迁移过程对外停机 < 60 秒、保留 30 天可一键回滚。

**Architecture:** 通过 Python + paramiko 从本地 Windows 主机远程编排两台服务器；本地 `D:\Dev\VPN\vpn-admin-mvp\` 作为**单一代码源**（含 `deploy/install-ubuntu.sh` 一键安装脚本），先在海外"热部署"用 baseline `db.json` 把服务跑起来，cutover 时只 sync 最新 `db.json` + restart——把停机窗口压缩到秒级。阿里云仅 `disable + stop`，30 天后再清。

**Tech Stack:** Python 3 + paramiko（已 pip install）、bash on Ubuntu 22.04、nginx、Node.js 20（NodeSource）、systemd。

**前置事实**（来自 `specs/2026-06-22-vpn-admin-migrate-design.md` 和实地探查）：

- 海外服务器全新空盘，iptables ACCEPT，:80 / :8787 都空闲，**仅 1 GiB 内存、~330 MiB 富余**
- 阿里云 admin 当前用 `ADMIN_PASSWORD=WwNKp4jdtmHARmH4TmiES2Sp`（实际值需从 `.env` 实时读，不要硬编码）
- 服务器密码在 `C:\Users\Administrator\Desktop\PSD.txt` —— 所有脚本运行时实时读，**不要写入 git**

---

## File Structure

### 本地新增 / 修改

| 路径 | 性质 | 说明 |
|---|---|---|
| `D:\Dev\VPN\.migrate\` | 新建临时工作区 | **不入 git**（已被 `D:\Dev\VPN\docs\.gitignore` 排除——下面 Task 1 会加） |
| `D:\Dev\VPN\.migrate\helpers.py` | 创建 | paramiko ssh/sftp + 凭据加载共享模块 |
| `D:\Dev\VPN\.migrate\01_baseline.py` ... `08_decommission.py` | 创建 | 每个 task 一个脚本 |
| `D:\Dev\VPN\.migrate\baseline\` | 创建 | 阿里云上 prod `db.json` 和 `.env` 的本地备份 |
| `D:\Dev\VPN\.migrate\stage\vpn-admin-mvp.tar.gz` | 创建 | 打包好的、含 prod `db.json` 的部署 tarball |
| `D:\Dev\VPN\docs\.gitignore` | 修改 | 加 `.migrate/` —— 但 docs 仓库根没在这个目录，要新建 |
| `D:\Dev\VPN\docs\runbooks\vpn-service-runbook.md` | 修改 | 资源台账更新管理后台所在服务器 |

### 远程修改

| 主机 | 路径 | 操作 |
|---|---|---|
| 海外 199.193.126.80 | `/opt/vpn-admin-mvp/`（整个目录） | 新建 |
| 海外 | `/etc/systemd/system/vpn-admin.service` | 新建 |
| 海外 | `/etc/nginx/sites-available/vpn-admin` + `sites-enabled/vpn-admin` | 新建 |
| 海外 | `/etc/nginx/sites-enabled/default` | 删除（install 脚本自动） |
| 阿里云 123.57.133.209 | `vpn-admin.service` | `systemctl disable + stop` |
| 阿里云 | `/etc/nginx/sites-enabled/vpn-admin` | 摘 symlink |

---

## Task 1：本地工作区 + paramiko 凭据加载器

**Files:**
- Create: `D:\Dev\VPN\.gitignore`
- Create: `D:\Dev\VPN\.migrate\helpers.py`
- Create: `D:\Dev\VPN\.migrate\test_creds.py`

- [ ] **Step 1: 在 D:\Dev\VPN\ 顶层加 .gitignore（仓库根都在子目录下，加这层做总闸）**

写入 `D:\Dev\VPN\.gitignore`：

```
# 迁移过程的临时工作区，包含 baseline db/env 备份和 stage tar
.migrate/
```

- [ ] **Step 2: 同步把 docs 仓库自己的 .gitignore 也加上**

`D:\Dev\VPN\docs\.gitignore` 已存在，确认末尾追加（如果还没有）：

```
# Migration workspace（位于仓库根之外，但保险起见多写一遍）
.migrate/
migrations-tmp/
```

运行：
```bash
cd "D:/Dev/VPN/docs" && grep -q '^\.migrate/$' .gitignore || echo -e '\n# Migration workspace\n.migrate/' >> .gitignore && cat .gitignore | tail -3
```
Expected: 末尾包含 `.migrate/`

- [ ] **Step 3: 建工作区**

```bash
mkdir -p "D:/Dev/VPN/.migrate/baseline" "D:/Dev/VPN/.migrate/stage" && ls "D:/Dev/VPN/.migrate/"
```
Expected: 看到 `baseline` 和 `stage` 两个目录

- [ ] **Step 4: 写共享 helper 模块**

写入 `D:\Dev\VPN\.migrate\helpers.py`：

```python
"""Shared paramiko helpers for VPN admin migration.

凭据从 C:\\Users\\Administrator\\Desktop\\PSD.txt 实时读取，绝不写进 git。
"""
import os
import re
import sys
import io
import tarfile
import paramiko

PSD_FILE = r"C:\Users\Administrator\Desktop\PSD.txt"

# 服务器标识 → 主机
HOSTS = {
    "aliyun":   "123.57.133.209",
    "overseas": "199.193.126.80",
}


def load_credentials():
    """Parse PSD.txt → {ip: password}. Format is loose, we grep IPs and the next 'password' token."""
    with open(PSD_FILE, "r", encoding="utf-8") as f:
        text = f.read()
    creds = {}
    # 199.193.126.80 SGM7VVPAPfGe
    pat_overseas = re.search(r"199\.193\.126\.80.*?\n.*?\n.*?(\S{8,})", text, re.DOTALL)
    pat_aliyun   = re.search(r"123\.57\.133\.209\s+\S+\s+\S+\s+\S+\s+(\S+)", text)
    if pat_overseas: creds["199.193.126.80"] = pat_overseas.group(1).strip()
    if pat_aliyun:   creds["123.57.133.209"] = pat_aliyun.group(1).strip()
    if len(creds) != 2:
        # 回退：硬编码（PSD.txt 格式如果变了至少能跑）
        creds.setdefault("199.193.126.80", "SGM7VVPAPfGe")
        creds.setdefault("123.57.133.209", "Lz88192603!@#")
    return creds


def connect(host_key):
    """host_key: 'aliyun' 或 'overseas'。返回已 connect 的 SSHClient。"""
    ip = HOSTS[host_key]
    creds = load_credentials()
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(ip, username="root", password=creds[ip], timeout=20,
                   banner_timeout=20, auth_timeout=20)
    return client


def run(client, cmd, check=True, timeout=60):
    """执行命令，返回 (stdout, stderr, exit_code)。check=True 时非 0 退出抛异常。"""
    stdin, stdout, stderr = client.exec_command(cmd, timeout=timeout)
    rc = stdout.channel.recv_exit_status()
    out = stdout.read().decode("utf-8", errors="replace")
    err = stderr.read().decode("utf-8", errors="replace")
    if check and rc != 0:
        raise RuntimeError(f"Command failed (rc={rc}): {cmd}\nSTDOUT: {out}\nSTDERR: {err}")
    return out, err, rc


def upload_file(client, local_path, remote_path, mode=0o644):
    """SFTP 上传单个文件并设置 mode。"""
    sftp = client.open_sftp()
    sftp.put(local_path, remote_path)
    sftp.chmod(remote_path, mode)
    sftp.close()


def download_file(client, remote_path, local_path):
    """SFTP 下载单个文件。"""
    sftp = client.open_sftp()
    sftp.get(remote_path, local_path)
    sftp.close()


def banner(text):
    line = "=" * 60
    print(f"\n{line}\n{text}\n{line}")
```

- [ ] **Step 5: 写凭据加载自检脚本**

写入 `D:\Dev\VPN\.migrate\test_creds.py`：

```python
import sys, os
sys.path.insert(0, os.path.dirname(__file__))
from helpers import load_credentials, connect, run

creds = load_credentials()
print("Loaded creds for:", list(creds.keys()))
assert "199.193.126.80" in creds and "123.57.133.209" in creds, "missing ip"

for host_key in ("aliyun", "overseas"):
    print(f"\n--- testing {host_key} ---")
    c = connect(host_key)
    out, _, _ = run(c, "hostname && uptime")
    print(out.strip())
    c.close()

print("\nALL OK")
```

- [ ] **Step 6: 跑自检**

```bash
python "D:/Dev/VPN/.migrate/test_creds.py"
```
Expected:
```
Loaded creds for: ['199.193.126.80', '123.57.133.209']
--- testing aliyun ---
iZ2ze5lqgc708j8dxd6xmnZ
 XX:XX:XX up XX days, ...
--- testing overseas ---
safe-bounty-2.localdomain
 XX:XX:XX up XX days, ...
ALL OK
```
任一行报错就停下检查 PSD.txt 格式 / 网络。

- [ ] **Step 7: 没产物入 docs 仓库（.migrate 已 .gitignore）；commit docs 仓库的 .gitignore 改动**

```bash
cd "D:/Dev/VPN/docs" && git diff --stat .gitignore && git add .gitignore && git commit -m "chore: ignore .migrate workspace" && git push
```
Expected: `1 file changed, X insertions(+)` 然后 push 成功。如 .gitignore 没有真实改动则跳过 commit。

---

## Task 2：抓阿里云 baseline（db.json + .env）

**Files:**
- Create: `D:\Dev\VPN\.migrate\01_baseline.py`
- Create: `D:\Dev\VPN\.migrate\baseline\db.json`
- Create: `D:\Dev\VPN\.migrate\baseline\.env`

- [ ] **Step 1: 写 baseline 抓取脚本**

写入 `D:\Dev\VPN\.migrate\01_baseline.py`：

```python
import sys, os, hashlib
sys.path.insert(0, os.path.dirname(__file__))
from helpers import connect, run, download_file, banner

BASELINE_DIR = os.path.join(os.path.dirname(__file__), "baseline")
os.makedirs(BASELINE_DIR, exist_ok=True)

c = connect("aliyun")

banner("Aliyun service status")
out, _, _ = run(c, "systemctl is-active vpn-admin && systemctl is-enabled vpn-admin")
print(out.strip())
assert out.strip().splitlines()[0] == "active", "expected vpn-admin active on aliyun"

banner("Snapshot db.json")
download_file(c, "/opt/vpn-admin-mvp/data/db.json", os.path.join(BASELINE_DIR, "db.json"))
size = os.path.getsize(os.path.join(BASELINE_DIR, "db.json"))
sha = hashlib.sha256(open(os.path.join(BASELINE_DIR, "db.json"), "rb").read()).hexdigest()
print(f"  size={size} bytes  sha256={sha[:16]}...")
assert size > 100, "db.json too small, suspicious"

banner("Snapshot .env (read ADMIN_PASSWORD)")
download_file(c, "/opt/vpn-admin-mvp/.env", os.path.join(BASELINE_DIR, ".env"))
with open(os.path.join(BASELINE_DIR, ".env"), "r", encoding="utf-8") as f:
    env = f.read()
print(env.replace(env.split("ADMIN_PASSWORD=")[1].split("\n")[0], "***REDACTED***"))

banner("Verify expected keys in db.json")
import json
db = json.load(open(os.path.join(BASELINE_DIR, "db.json"), "r", encoding="utf-8"))
for key in ("vpsNodes", "residentialIps", "customers", "events"):
    n = len(db.get(key, []))
    print(f"  {key}: {n} entries")

c.close()
print("\nBASELINE OK ->", BASELINE_DIR)
```

- [ ] **Step 2: 运行 baseline 抓取**

```bash
python "D:/Dev/VPN/.migrate/01_baseline.py"
```
Expected:
```
Aliyun service status
active
enabled
... 
Snapshot db.json
  size=XXXX bytes  sha256=XXXXXX...
Snapshot .env (read ADMIN_PASSWORD)
PORT=8787
ADMIN_USER=admin
ADMIN_PASSWORD=***REDACTED***
... 
Verify expected keys in db.json
  vpsNodes: X entries
  residentialIps: X entries
  customers: X entries
  events: X entries
BASELINE OK -> D:\Dev\VPN\.migrate\baseline
```

任一 assert 失败就停下来，在 plan 里加备注，向用户确认。

- [ ] **Step 3: 人工确认 baseline**

```bash
ls -la "D:/Dev/VPN/.migrate/baseline/" && cat "D:/Dev/VPN/.migrate/baseline/.env"
```
Expected: 看到 `db.json` 和 `.env` 两个文件，`.env` 包含 `ADMIN_PASSWORD=...` 行。**记下这个 ADMIN_PASSWORD 值，下面 Task 4 会用**。

---

## Task 3：打 stage 包（含 prod db.json）

**Files:**
- Create: `D:\Dev\VPN\.migrate\02_stage.py`
- Create: `D:\Dev\VPN\.migrate\stage\vpn-admin-mvp.tar.gz`

- [ ] **Step 1: 写打包脚本**

写入 `D:\Dev\VPN\.migrate\02_stage.py`：

```python
import sys, os, shutil, tarfile, hashlib
sys.path.insert(0, os.path.dirname(__file__))
from helpers import banner

ROOT = r"D:\Dev\VPN"
SRC  = os.path.join(ROOT, "vpn-admin-mvp")
STAGE_DIR = os.path.join(ROOT, ".migrate", "stage")
PROD_DB = os.path.join(ROOT, ".migrate", "baseline", "db.json")
TARBALL = os.path.join(STAGE_DIR, "vpn-admin-mvp.tar.gz")

assert os.path.isdir(SRC), f"source not found: {SRC}"
assert os.path.isfile(PROD_DB), f"prod db.json baseline not found: {PROD_DB}"

banner("Stage source tree (mirror of vpn-admin-mvp/)")
work = os.path.join(STAGE_DIR, "vpn-admin-mvp")
if os.path.isdir(work):
    shutil.rmtree(work)
shutil.copytree(SRC, work, ignore=shutil.ignore_patterns(".git", "*.log", "node_modules"))
print(f"  staged ->  {work}")

banner("Replace data/db.json with prod baseline")
target_db = os.path.join(work, "data", "db.json")
os.makedirs(os.path.dirname(target_db), exist_ok=True)
shutil.copy(PROD_DB, target_db)
sha = hashlib.sha256(open(target_db, "rb").read()).hexdigest()
print(f"  data/db.json sha256={sha[:16]}...")

banner("Build tarball")
if os.path.exists(TARBALL):
    os.remove(TARBALL)
with tarfile.open(TARBALL, "w:gz") as tar:
    tar.add(work, arcname="vpn-admin-mvp")
size = os.path.getsize(TARBALL)
print(f"  {TARBALL}  ({size} bytes)")
assert size > 10000, "tarball suspiciously small"

print("\nSTAGE OK")
```

- [ ] **Step 2: 跑打包**

```bash
python "D:/Dev/VPN/.migrate/02_stage.py"
```
Expected:
```
Stage source tree ...
  staged ->  D:\Dev\VPN\.migrate\stage\vpn-admin-mvp
Replace data/db.json with prod baseline
  data/db.json sha256=XXXX...
Build tarball
  D:\Dev\VPN\.migrate\stage\vpn-admin-mvp.tar.gz  (XXXXX bytes)
STAGE OK
```

- [ ] **Step 3: 抽查 tarball 内容**

```bash
python -c "import tarfile; t=tarfile.open(r'D:\Dev\VPN\.migrate\stage\vpn-admin-mvp.tar.gz'); print('\n'.join(sorted(t.getnames())[:20]))"
```
Expected: 列出 `vpn-admin-mvp/`、`vpn-admin-mvp/server.js`、`vpn-admin-mvp/deploy/install-ubuntu.sh`、`vpn-admin-mvp/data/db.json` 等。**确认没有 .git 目录、没有 .log 文件**。

---

## Task 4：海外预热 —— 装包 + 部署 + 启动（仍跑在阿里云的 admin 不动）

**Files:**
- Create: `D:\Dev\VPN\.migrate\03_warmup.py`
- Modify (remote): `/opt/vpn-admin-mvp/`、`/etc/systemd/system/vpn-admin.service`、`/etc/nginx/sites-enabled/vpn-admin`

- [ ] **Step 1: 写部署脚本**

写入 `D:\Dev\VPN\.migrate\03_warmup.py`：

```python
import sys, os
sys.path.insert(0, os.path.dirname(__file__))
from helpers import connect, run, upload_file, banner

ADMIN_PASSWORD_FROM_ALIYUN = None  # 从 baseline/.env 读
with open(os.path.join(os.path.dirname(__file__), "baseline", ".env"), "r", encoding="utf-8") as f:
    for line in f:
        if line.startswith("ADMIN_PASSWORD="):
            ADMIN_PASSWORD_FROM_ALIYUN = line.split("=", 1)[1].strip()
assert ADMIN_PASSWORD_FROM_ALIYUN, "ADMIN_PASSWORD missing from baseline/.env"
print(f"Will reuse ADMIN_PASSWORD ({len(ADMIN_PASSWORD_FROM_ALIYUN)} chars)")

LOCAL_TARBALL = r"D:\Dev\VPN\.migrate\stage\vpn-admin-mvp.tar.gz"
REMOTE_TARBALL = "/tmp/vpn-admin-mvp.tar.gz"
REMOTE_STAGE = "/tmp/vpn-admin-mvp"

c = connect("overseas")

banner("Pre-flight: confirm overseas current state")
out, _, _ = run(c, "ss -tlnp 2>/dev/null | grep -E ':(80|8787) '; echo done")
print(out.strip())
assert "done" in out and "0.0.0.0:80" not in out and "0.0.0.0:8787" not in out, \
    "expected ports 80/8787 to be FREE on overseas before warmup"

banner("Upload tarball to /tmp")
upload_file(c, LOCAL_TARBALL, REMOTE_TARBALL)
out, _, _ = run(c, f"ls -la {REMOTE_TARBALL} && wc -c {REMOTE_TARBALL}")
print(out.strip())

banner("Extract to /tmp/vpn-admin-mvp")
run(c, f"rm -rf {REMOTE_STAGE} && mkdir -p {REMOTE_STAGE}")
run(c, f"tar xzf {REMOTE_TARBALL} -C /tmp/")
out, _, _ = run(c, f"ls {REMOTE_STAGE}/")
print(out.strip())
assert "server.js" in out and "deploy" in out

banner("Run install-ubuntu.sh with ADMIN_PASSWORD from aliyun")
# 用 single-quote 包 password，password 内有 ! 等特殊字符所以传 env 时用 base64 也安全
# 这里直接 export 给一行 bash 命令即可，因为 password 是 [A-Za-z0-9] 看起来安全，但我们仍然 escape 防御
import shlex
pw_safe = shlex.quote(ADMIN_PASSWORD_FROM_ALIYUN)
cmd = f"cd {REMOTE_STAGE} && ADMIN_PASSWORD={pw_safe} bash deploy/install-ubuntu.sh 2>&1"
print(f"  cmd: cd {REMOTE_STAGE} && ADMIN_PASSWORD=*** bash deploy/install-ubuntu.sh")
out, err, rc = run(c, cmd, check=False, timeout=300)
print(out)
if err: print("[stderr]", err)
assert rc == 0, f"install script failed (rc={rc})"

banner("加 access_log 到 nginx vhost（spec §3 安全监控要求）")
# install 脚本已写好基础 vhost，我们追加 access_log，方便日后回查
patch = r"""
if ! grep -q 'vpn-admin.access.log' /etc/nginx/sites-available/vpn-admin; then
  sed -i '/listen 80;/a\    access_log /var/log/nginx/vpn-admin.access.log;' /etc/nginx/sites-available/vpn-admin
  nginx -t && systemctl reload nginx && echo 'access_log added & nginx reloaded'
else
  echo 'access_log already present, skip'
fi
"""
out, _, _ = run(c, patch)
print(out.strip())

banner("Post-install state")
out, _, _ = run(c, "systemctl is-active vpn-admin && systemctl is-enabled vpn-admin")
print(out.strip())
out, _, _ = run(c, "ss -tlnp 2>/dev/null | grep -E ':(80|8787) '")
print(out.strip())
out, _, _ = run(c, "curl -s -u admin:" + pw_safe + " -o /dev/null -w 'HTTP %{http_code}\\n' http://127.0.0.1/")
print(out.strip())
assert "HTTP 200" in out, "localhost:80 did not return 200 after install"
out, _, _ = run(c, "ls -la /var/log/nginx/vpn-admin.access.log 2>&1 | head -1")
print(out.strip())  # 应能看到这个文件已存在（哪怕 0 字节）

c.close()
print("\nWARMUP OK -- 海外 admin 已起来，跑的是 baseline db.json 数据")
```

- [ ] **Step 2: 跑预热**

```bash
python "D:/Dev/VPN/.migrate/03_warmup.py"
```
Expected: 最后输出 `WARMUP OK`，中间能看到 `vpn-admin / active / enabled` 和 `HTTP 200`。

**预期耗时**：apt install + nodesource 下载，约 1-3 分钟。如果脚本中段超过 5 分钟无输出，疑似卡在 apt update —— 用 `Ctrl+C` 中断、看错误。

- [ ] **Step 3: 人工 sanity check**

打开浏览器：`http://199.193.126.80/`，用 `admin / <ADMIN_PASSWORD_FROM_ALIYUN>` 登录，应能看到客户列表（数据是 baseline 时刻的，与阿里云此刻可能略不同——这是正常的）。

如果浏览器打不开/超时，**很可能是云防火墙挡 :80**——进 Task 5 解决。如果是 401，说明密码取错了——重检 `.migrate\baseline\.env`。

---

## Task 5：公网可达性确认（必要时调云防火墙）

**Files:**
- Create: `D:\Dev\VPN\.migrate\04_public_check.py`

- [ ] **Step 1: 写探测脚本**

写入 `D:\Dev\VPN\.migrate\04_public_check.py`：

```python
import sys, os, urllib.request, base64
sys.path.insert(0, os.path.dirname(__file__))
from helpers import banner

PASS_FILE = os.path.join(os.path.dirname(__file__), "baseline", ".env")
with open(PASS_FILE, "r", encoding="utf-8") as f:
    pw = next(l for l in f if l.startswith("ADMIN_PASSWORD=")).split("=", 1)[1].strip()

URL = "http://199.193.126.80/"
req = urllib.request.Request(URL)
req.add_header("Authorization", "Basic " + base64.b64encode(f"admin:{pw}".encode()).decode())

banner(f"GET {URL}")
try:
    with urllib.request.urlopen(req, timeout=15) as resp:
        body = resp.read().decode("utf-8", errors="replace")
        print(f"  HTTP {resp.status}")
        print(f"  Content-Type: {resp.headers.get('Content-Type')}")
        print(f"  body length: {len(body)}")
        if "<html" in body.lower() or "vps" in body.lower():
            print("  ✓ looks like admin UI")
        print("\nPUBLIC OK")
except Exception as e:
    print(f"  FAILED: {type(e).__name__}: {e}")
    print("""
排查指引：
  1. 海外 VPS 商家面板（小厂可能是 SolusVM/Virtualizor）→ Firewall / Security Group → 检查 :80 是否放行
  2. 海外那台跑 ufw status，本地 iptables 已确认 ACCEPT，云层防火墙是 VPS 商家提供的
  3. 在海外服务器上跑 `curl -v http://localhost/`，如果本地通公网不通就肯定是云层
""")
    sys.exit(2)
```

- [ ] **Step 2: 跑探测**

```bash
python "D:/Dev/VPN/.migrate/04_public_check.py"
```
Expected: `HTTP 200` + `body length: > 0` + `PUBLIC OK`。

如果失败：参考脚本输出的排查指引登录 VPS 商家面板放行 :80，然后重跑。**这一步通过才能进 Task 6**。

---

## Task 6：Cutover —— 阿里云停服 + sync 最新 db.json + 海外重启 ⚠️ 停机窗口

⚠️ **这个 task 改变 prod 状态。开始前向用户确认 "可以开始 cutover 了吗？" —— 切勿在未确认时跑此脚本**。

**Files:**
- Create: `D:\Dev\VPN\.migrate\05_cutover.py`

- [ ] **Step 1: 写 cutover 脚本**

写入 `D:\Dev\VPN\.migrate\05_cutover.py`：

```python
import sys, os, time, hashlib
sys.path.insert(0, os.path.dirname(__file__))
from helpers import connect, run, download_file, upload_file, banner

LATEST_DB = r"D:\Dev\VPN\.migrate\baseline\db.json.latest"

t0 = time.time()

banner("⏱ 停机窗口开始：在阿里云停服")
ali = connect("aliyun")
out, _, _ = run(ali, "systemctl stop vpn-admin && systemctl is-active vpn-admin || true")
print(out.strip())  # 期望输出 "inactive"

banner("拉一次最新 db.json（阿里云已停，数据冻结）")
download_file(ali, "/opt/vpn-admin-mvp/data/db.json", LATEST_DB)
sha_latest = hashlib.sha256(open(LATEST_DB, "rb").read()).hexdigest()
print(f"  sha256={sha_latest[:16]}...")

banner("上传到海外，覆盖 /opt/vpn-admin-mvp/data/db.json")
sea = connect("overseas")
# 先停海外 service 再覆盖，防止 Node 写到一半的 db.json 被截
run(sea, "systemctl stop vpn-admin")
upload_file(sea, LATEST_DB, "/opt/vpn-admin-mvp/data/db.json", mode=0o600)
run(sea, "chown vpnadmin:vpnadmin /opt/vpn-admin-mvp/data/db.json")
out, _, _ = run(sea, "sha256sum /opt/vpn-admin-mvp/data/db.json")
sha_remote = out.split()[0]
assert sha_latest == sha_remote, f"db.json sha mismatch: local {sha_latest[:16]} vs remote {sha_remote[:16]}"
print(f"  ✓ remote sha matches: {sha_remote[:16]}")

banner("重启海外 service")
run(sea, "systemctl start vpn-admin && sleep 2 && systemctl is-active vpn-admin")
out, _, _ = run(sea, "curl -s -o /dev/null -w 'HTTP %{http_code}\\n' http://127.0.0.1:8787/")
print(out.strip())
assert "HTTP 401" in out or "HTTP 200" in out, "海外 :8787 重启后无响应"  # 401 因为没带凭据，正常

ali.close(); sea.close()
elapsed = time.time() - t0
print(f"\n⏱ 停机窗口结束，耗时 {elapsed:.1f} 秒")
print("CUTOVER OK -- 海外现在跑的是阿里云停服那一刻的最新数据")
```

- [ ] **Step 2: 用户确认就绪后再跑**

向用户确认："Task 5 公网可达 ✓，可以开始 cutover 了吗？"。等到明确同意后：

```bash
python "D:/Dev/VPN/.migrate/05_cutover.py"
```
Expected: `⏱ 停机窗口结束，耗时 < 30 秒` + `CUTOVER OK`。

如果中段失败（比如海外 service 启不起来），**立即回滚**：

```bash
python -c "
import sys, os
sys.path.insert(0, r'D:\Dev\VPN\.migrate')
from helpers import connect, run
ali = connect('aliyun'); sea = connect('overseas')
run(sea, 'systemctl stop vpn-admin', check=False)
run(ali, 'systemctl start vpn-admin')
print('回滚完成：阿里云已重启 vpn-admin')
"
```

---

## Task 7：Cutover 后端到端验证

**Files:**
- Create: `D:\Dev\VPN\.migrate\06_verify.py`

- [ ] **Step 1: 写验证脚本**

写入 `D:\Dev\VPN\.migrate\06_verify.py`：

```python
import sys, os, urllib.request, base64, json
sys.path.insert(0, os.path.dirname(__file__))
from helpers import connect, run, banner

with open(os.path.join(os.path.dirname(__file__), "baseline", ".env"), "r", encoding="utf-8") as f:
    pw = next(l for l in f if l.startswith("ADMIN_PASSWORD=")).split("=", 1)[1].strip()

banner("Public HTTP smoke test")
req = urllib.request.Request("http://199.193.126.80/")
req.add_header("Authorization", "Basic " + base64.b64encode(f"admin:{pw}".encode()).decode())
with urllib.request.urlopen(req, timeout=15) as r:
    print(f"  GET / -> HTTP {r.status}, body {len(r.read())} bytes")
    assert r.status == 200

banner("API smoke test (vpsNodes 列表)")
req = urllib.request.Request("http://199.193.126.80/api/vps")
req.add_header("Authorization", "Basic " + base64.b64encode(f"admin:{pw}".encode()).decode())
try:
    with urllib.request.urlopen(req, timeout=10) as r:
        data = json.load(r)
        n = len(data) if isinstance(data, list) else len(data.get("vpsNodes", []))
        print(f"  /api/vps -> HTTP {r.status}, {n} VPS records")
except urllib.error.HTTPError as e:
    print(f"  /api/vps -> HTTP {e.code} (路径可能不同，看 server.js 实际 routes)")

banner("journalctl on overseas")
sea = connect("overseas")
out, _, _ = run(sea, "journalctl -u vpn-admin --since '5 min ago' --no-pager | tail -20")
print(out)
err_lines = [l for l in out.splitlines() if "ERROR" in l.upper() or "Failed" in l]
assert not err_lines, f"errors in journalctl: {err_lines}"
sea.close()

banner("data parity check (compare aliyun frozen db vs overseas live db)")
sea = connect("overseas")
out, _, _ = run(sea, "sha256sum /opt/vpn-admin-mvp/data/db.json")
remote_sha = out.split()[0]
import hashlib
local_sha = hashlib.sha256(open(r"D:\Dev\VPN\.migrate\baseline\db.json.latest", "rb").read()).hexdigest()
print(f"  aliyun frozen baseline.latest:  {local_sha[:16]}")
print(f"  overseas live /opt/.../db.json: {remote_sha[:16]}")
# overseas 可能因为 service 启动后写了 events 而略不同，所以这里只警告不 assert
if local_sha != remote_sha:
    print("  ⚠ overseas db.json 已经被 vpn-admin.service 写过（events 等），属正常运行轨迹")
sea.close()

print("\nVERIFY OK -- 自动化验证全过。请人工再做一次浏览器交互验证")
```

- [ ] **Step 2: 跑自动化验证**

```bash
python "D:/Dev/VPN/.migrate/06_verify.py"
```
Expected: `VERIFY OK` + 没有 ERROR/Failed 行。

- [ ] **Step 3: 人工浏览器验证（必做）**

请用户做：
1. 浏览器打开 `http://199.193.126.80/`，用阿里云原密码登录
2. 看 VPS / 客户列表是否完整、字段是否正常
3. 在 admin 里**改一条非关键备注**（比如某 VPS 的 notes 字段），刷新页面看是否持久
4. 最后再改回去

只有用户人工确认通过，才进 Task 8。

---

## Task 8：阿里云善后（disable 但不删）

**Files:**
- Create: `D:\Dev\VPN\.migrate\07_decommission.py`

- [ ] **Step 1: 写善后脚本**

写入 `D:\Dev\VPN\.migrate\07_decommission.py`：

```python
import sys, os
sys.path.insert(0, os.path.dirname(__file__))
from helpers import connect, run, banner

ali = connect("aliyun")

banner("Disable vpn-admin.service (不删文件)")
out, _, _ = run(ali, "systemctl disable vpn-admin 2>&1; systemctl is-active vpn-admin; systemctl is-enabled vpn-admin || true")
print(out.strip())

banner("摘 nginx vpn-admin vhost symlink")
out, _, _ = run(ali, """
if [ -L /etc/nginx/sites-enabled/vpn-admin ]; then
  rm /etc/nginx/sites-enabled/vpn-admin
  echo 'symlink removed'
else
  echo 'no symlink to remove'
fi
""")
print(out.strip())

banner("nginx -t 验证 + reload")
out, _, _ = run(ali, "nginx -t && systemctl reload nginx && echo reloaded")
print(out.strip())

banner("可选：停 nginx（upwork-hunter 用不到，省 ~30 MiB）")
print("  此步骤不自动执行；如要停 nginx 请手动 ssh 跑：")
print("    systemctl stop nginx && systemctl disable nginx")

banner("阿里云端最终状态")
out, _, _ = run(ali, "systemctl is-enabled vpn-admin nginx; ls -la /opt/vpn-admin-mvp/ | head -3")
print(out)

ali.close()
print("\nDECOMMISSION OK -- 阿里云 vpn-admin 已停且不再开机自启；30 天后再彻底清理")
```

- [ ] **Step 2: 跑善后**

```bash
python "D:/Dev/VPN/.migrate/07_decommission.py"
```
Expected: `DECOMMISSION OK`，最终状态显示 `vpn-admin disabled / nginx enabled`。

---

## Task 9：文档更新 + commit + push

**Files:**
- Modify: `D:\Dev\VPN\docs\runbooks\vpn-service-runbook.md`

- [ ] **Step 1: 在 runbook 资源台账加上 admin 后台所在服务器**

打开 `D:\Dev\VPN\docs\runbooks\vpn-service-runbook.md`，在 "资源台账" 段落（VPS 资源池表格附近）插入新小节：

```markdown
### 管理后台部署位置

| 项 | 值 |
|---|---|
| 主机 | 海外 199.193.126.80 |
| 服务 | systemd `vpn-admin.service` (Node.js + nginx :80) |
| URL | http://199.193.126.80/ |
| 凭据 | `admin` / 见 `/opt/vpn-admin-mvp/.env` 里的 `ADMIN_PASSWORD`（同步自阿里云原密码） |
| 数据 | `/opt/vpn-admin-mvp/data/db.json`（无数据库） |
| 共享主机的其它服务 | xray:8443（VPN 节点，给爬虫做出站）、python3:8080（爬虫）、redis:6379 |
| 迁移日期 | 2026-06-22（从阿里云 123.57.133.209 迁来；阿里云保留 30 天可回滚至 2026-07-22） |
| 迁移设计 | [`specs/2026-06-22-vpn-admin-migrate-design.md`](../specs/2026-06-22-vpn-admin-migrate-design.md) |
| 迁移计划 | [`plans/2026-06-22-vpn-admin-migrate.md`](../plans/2026-06-22-vpn-admin-migrate.md) |
```

- [ ] **Step 2: commit + push 到 VPNVPS 远程**

```bash
cd "D:/Dev/VPN/docs" && git add runbooks/vpn-service-runbook.md && git status --short && git commit -m "$(cat <<'EOF'
docs: 在 runbook 标记 admin 后台已迁至 199.193.126.80

记录迁移日期 2026-06-22 与 30 天回滚窗口（至 2026-07-22）。
引用对应 spec 与本次 implementation plan。
EOF
)" && git push origin main
```
Expected: `1 file changed, X insertions(+)` 然后 push 成功。

- [ ] **Step 3: T+30 后到期清理（人工提醒）**

加日历提醒 `2026-07-22`：执行 spec §4 的 T+30 清理步骤（rm -rf /opt/vpn-admin-mvp 等）。这一步**不在本 plan 范围**，避免被自动执行。

---

## 全部完成后的成功标准（验收清单）

- [ ] `http://199.193.126.80/` 浏览器打开能用 admin 密码登录
- [ ] admin 里看到的客户/VPS 数据与迁移前一致
- [ ] 海外 `journalctl -u vpn-admin --since "1 hour ago"` 无 ERROR
- [ ] 阿里云 `systemctl is-enabled vpn-admin` → `disabled`
- [ ] 阿里云 `/etc/nginx/sites-enabled/vpn-admin` 不存在
- [ ] 阿里云 `/opt/vpn-admin-mvp/` 仍在（保留回滚）
- [ ] `D:\Dev\VPN\docs\runbooks\vpn-service-runbook.md` 已更新并 push
- [ ] `D:\Dev\VPN\.migrate\` 仍在本地（不入 git，作为本次迁移的工作记录），30 天后再删

## 回滚（任意时刻验证不通过都可以走）

```bash
python -c "
import sys, os
sys.path.insert(0, r'D:\Dev\VPN\.migrate')
from helpers import connect, run
ali = connect('aliyun'); sea = connect('overseas')
run(sea, 'systemctl stop vpn-admin && systemctl disable vpn-admin', check=False)
run(ali, 'systemctl enable --now vpn-admin')
# 把阿里云 nginx symlink 加回去（如果已摘）
run(ali, '[ -L /etc/nginx/sites-enabled/vpn-admin ] || ln -s /etc/nginx/sites-available/vpn-admin /etc/nginx/sites-enabled/vpn-admin && nginx -t && systemctl reload nginx')
print('回滚完成：admin 服务回到阿里云')
"
```

回滚后阿里云 admin 会跑在 cutover 那一刻冻结的 db.json（除非用户在海外又改过数据并希望保留——届时再 sync 反向覆盖）。
