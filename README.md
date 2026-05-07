Cursor 远程开发服务器完整部署文档（解决Mac内存爆满卡顿）
一、方案整体说明
1.1 解决的核心问题
- 本地 16G Mac 长期被 Cursor AI、代码索引、服务进程占满内存，导致整机卡顿、风扇狂转
- 本地起测试服务、后台服务、编译、运行，叠加 AI 占用，内存直接爆
1.2 最终架构（最优生产方案）
- 本地 Mac：仅运行 Cursor 客户端界面（极低内存占用 2–3G）
- 远程 Ubuntu 24.04 Server（无桌面）：承担所有计算、AI 会话、代码索引、编译、服务部署、后台运行
- 连接方式：Remote-SSH 官方协议，无缝无感开发，和本地一模一样
1.3 重要结论
服务器绝对不需要桌面环境，纯命令行即可，Cursor 远程开发完全不依赖系统桌面，装桌面只会浪费内存、降低稳定性。

---
二、服务器初始环境准备（Ubuntu 24.04 Server）
2.1 系统要求
- 系统：Ubuntu 24.04 Server 最小化安装
- 必须开启：SSH 服务（默认安装版自带）
- 必须联网：能访问外网（拉取 Cursor Server、Node、AI 接口）
2.2 基础依赖安装
登录服务器终端，执行全套初始化命令：
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget zip unzip build-essential tmux openssh-server
2.3 安装 Node.js 20.x（Cursor Server 强制依赖）
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
输出 v20.x.x 即为正常。
2.4 配置 SSH 稳定连接（核心优化）
修改服务器 SSH 配置，防止断线、提升稳定性：
sudo vim /etc/ssh/sshd_config
添加/修改以下参数：
ClientAliveInterval 30
ClientAliveCountMax 3
重启 SSH 服务：
sudo systemctl restart sshd

---
三、本地 Mac 免密登录配置（必做）
3.1 本地 Mac 生成密钥
本地终端执行：
ssh-keygen -t ed25519
全程回车，无需密码。
3.2 推送公钥到远程服务器
ssh-copy-id 服务器用户名@服务器IP
3.3 本地 SSH 永久配置（一劳永逸）
编辑本地 Mac 配置文件：
vim ~/.ssh/config
写入以下内容（直接复制替换，改 IP 和用户名即可）：
Host cursor-dev
  HostName 你的服务器公网IP
  User 你的服务器用户名
  Port 22
  IdentityFile ~/.ssh/id_ed25519
  ForwardAgent yes
  ServerAliveInterval 30
  ServerAliveCountMax 3
之后只需输入 ssh cursor-dev 即可一键登录。

---
四、Cursor 远程连接完整部署（核心步骤）
4.1 本地 Cursor 准备
- 确保 Cursor 为最新版 3.x
- 安装扩展：Remote - SSH（微软官方）
4.2 连接远程服务器
1. 打开 Cursor，按下 Cmd + Shift + P
2. 输入：Remote-SSH: Connect to Host
3. 选择刚才配置的cursor-dev
4. 选择 Linux 系统
4.3 自动安装 Cursor 服务端
首次连接，服务器会 自动下载、安装、启动 Cursor Server，无需手动干预。
服务端安装目录：~/.cursor-server
等待 1–2 分钟，左下角出现 SSH: cursor-dev 即部署成功。
4.4 验证运行环境
在 Cursor 内置终端输入：
uname -a
显示 Ubuntu 即为远程环境，所有代码、AI、运行、编译全部跑在服务器上。

---
五、关键优化：彻底解决 Mac 内存爆满
5.1 所有资源全部远程化
- 代码文件：全部放在远程服务器，本地无项目文件
- 代码索引、AI 缓存、会话记录：全部生成在服务器
- 测试服务、后端服务、脚本、编译进程：全部远程运行
5.2 Cursor 远程专属设置
远程连接后，打开设置，开启：
- 关闭本地文件索引
- 关闭本地后台分析
- AI 模型选择：Claude Opus / Auto（全部远程调用）
此时 Mac 内存常驻从 10G+ 降到 2–3G。

---
六、远程开发必备工具配置
6.1 Tmux 服务保活（断开SSH不中断服务）
常用命令：
# 新建会话
tmux new -s dev

# 退出会话（后台运行）
Ctrl+b 松开后按 d

# 恢复会话
tmux a -t dev

# 查看所有会话
tmux ls
6.2 本地访问远程服务（端口转发）
如需访问远程 3000/8080 等服务，本地执行：
ssh -L 3000:127.0.0.1:3000 cursor-dev
本地浏览器打开 localhost:3000 即可访问远程服务。

---
七、常见问题说明
7.1 要不要手动安装 cursor-server？
不需要。Remote-SSH 模式下全部自动安装，手动装反而容易版本不匹配、连接失败。
7.2 无桌面版 Ubuntu 会不会功能缺失？
完全不会。GUI 全部由你本地 Mac 的 Cursor 提供，服务器只做计算和运行，桌面无任何作用。
7.3 Claude Opus 能用吗？
完全可用，模型调用走远程服务器网络，稳定、不占本地资源。

---
八、最终效果
- ✅ Mac 彻底告别内存爆满、卡顿、发热
- ✅ 所有重任务：AI 分析、代码重构、编译、服务部署全在远程
- ✅ 开发体验和本地 100% 一致
- ✅ 服务可 7*24 小时后台挂跑