# algo-practice

我的第一个代码仓库，用来练习 C++ 和熟悉 Git 工作流。

## 环境

- 系统：Windows 11 + WSL2（Ubuntu 24.04）
- 编辑器：VS Code（通过 WSL 扩展连接）
- 编译器：g++
- 版本管理：Git，通过 SSH 推送到 GitHub

## 文件

| 文件 | 说明 |
|---|---|
| `hello.cpp` | 第一个 C++ 程序，打印一行文字 |
| `.gitignore` | 忽略编译产生的可执行文件 |

## 怎么编译运行

```bash
g++ hello.cpp -o hello
./hello
```

---

## 我踩过的坑

第一次把代码推上 GitHub 花了几个小时。记录一下过程和最终的解法，
免得以后重装系统再来一遍。

### 1. `wsl --install` 只装好了一半

**现象**：在管理员 PowerShell 里敲 `wsl --install`，重启电脑后什么也没有变化。

**排查**：

```bash
wsl --version          # 正常，WSL 内核是装好了的（2.7.14.0）
wsl --list --online    # 能列出 20 多个发行版，组件没问题
```

但开始菜单里**搜不到 Ubuntu**——说明发行版没装上。

**原因**：`wsl --install` 做了两件事（装内核、下载发行版），
**下载 Ubuntu 那一步依赖 Microsoft Store，国内网络下失败了**。

**解法**：

```powershell
wsl --install -d Ubuntu-24.04 --no-launch
```

- 用 `Ubuntu-24.04`（LTS）而不是默认的 `Ubuntu`（指向最新版 26.04），
  因为 **LTS 版本网上的教程和报错对照资料最多**
- `--no-launch` 让它只下载安装、不自动启动，
  这样初始化窗口不会在没准备好的时候弹出来又错过

### 2. WSL 的起始目录跟着 PowerShell 走

**现象**：从 PowerShell 敲 `wsl` 进去后，找不到自己刚才创建的文件。

**原因**：WSL **会跟着你 PowerShell 当前所在的目录启动**。

| 在 PowerShell 里的位置 | 进 WSL 后落在哪 |
|---|---|
| `PS C:\Users\Shang>` | `/mnt/c/Users/Shang`（Windows 的 C 盘） |
| `PS C:\>` | `/mnt/c` |

**关键概念**：`/mnt/c/...` 是 **Windows 的 C 盘**，`~` 是 **Linux 家目录**，
这两个是完全不同的地方。

**习惯**：进 WSL 后第一件事就是 `cd ~`。

### 3. GitHub 连不上（两个问题叠加）

修好第一个，才暴露出第二个。

#### 问题一：`github.com` 被 hosts 文件劫持到 `127.0.0.1`

**现象**：

```
Failed to connect to github.com port 443 after 3 ms: Couldn't connect to server
```

**关键线索**：**3 毫秒**就失败——说明连接根本没发出去，不是网络慢。

**诊断**：

```bash
getent hosts github.com
```

结果：

```
127.0.0.1       github.com
```

被解析到自己电脑了。

**原因**：Windows 的 `C:\Windows\System32\drivers\etc\hosts` 里
有一段 GitHub 相关条目全部指向 `127.0.0.1`（第 41–70 行），
包括 `github.com`、`api.github.com`、`raw.githubusercontent.com` 等。
这些条目来源不明（可能是某个加速工具留下的），但它们**正在阻止连接**。

**解法**：用管理员权限编辑该 hosts 文件，删掉（或注释）那些行，然后：

```powershell
ipconfig /flushdns
```

修好后：

```
20.205.243.166  github.com
```

> 注意：刚修完立刻查可能仍显示 `127.0.0.1`，那是 WSL 的 DNS 缓存，多查几次即可。

#### 问题二：HTTPS 的 TLS 握手被重置

hosts 修好后，仍然报：

```
fatal: unable to access '...': GnuTLS recv error (-110):
The TLS connection was non-properly terminated.
```

连接能建立，但 **TLS 握手被中途掐断**。

**解法：改用 SSH，并让 SSH 走 443 端口**

```bash
# 1. 生成密钥（连按 3 次回车，用默认路径、不设密码短语）
ssh-keygen -t ed25519 -C "zhideng@cau.edu.cn"

# 2. 复制公钥
cat ~/.ssh/id_ed25519.pub
```

把打印出来的整行加到 GitHub：
**Settings → SSH and GPG keys → New SSH key → Add SSH key**

```bash
# 3. 配置 SSH 走 443 端口
nano ~/.ssh/config
```

内容：

```
Host github.com
  Hostname ssh.github.com
  Port 443
  User git
```

```bash
# 4. 测试（第一次问 yes/no 时敲 yes）
ssh -T git@github.com
```

成功的样子：

```
Hi zhideng-shang! You've successfully authenticated,
but GitHub does not provide shell access.
```

> 这句话看着像报错，其实**是成功**。`does not provide shell access` 是 GitHub 的正常提示，
> 重点是前面那句 `You've successfully authenticated`。

```bash
# 5. 把仓库地址改成 SSH 并推送
git remote set-url origin git@github.com:zhideng-shang/algo-practice.git
git remote -v
git push -u origin main
```

**结论**：在国内网络下，**SSH + 443 比 HTTPS 稳定得多**。
以后配新机器直接用这个方式，不要在 HTTPS 上浪费时间。

### 4. 其他几个小教训

| 现象 | 真实含义 |
|---|---|
| `destination path 'xxx' already exists` | **不是错误**——已经克隆过了，直接 `cd` 进去 |
| `Your branch is based on 'origin/main', but the upstream is gone` | **不是错误**——远端还没有这个分支（因为还没 push 成功过） |
| `nothing added to commit but untracked files present` | 文件没有变化，没有新东西要提交 |

另外两条：

- **一次粘贴多行命令会掩盖真正的错误**。第一条失败了，第二条还会跟着跑，
  屏幕上出现一堆 `fatal`，很难分辨哪个才是真问题。**一条一条敲。**
- **终端里的 Tab 是自动补全，不是缩进**。写代码时用空格缩进，或者用 VS Code 编辑文件。
