# PikPak rclone配置完整教程：如何用rclone挂载PikPak网盘？命令行操作、Emby/Jellyfin影视库搭建、常见问题全解析

有一类人，在发现 PikPak 的那一刻就知道自己找到了宝贝。

磁力链接一丢进去，十几秒后文件就躺在云盘里——不用等，不用挂机，你睡着了它还在跑。对于玩 NAS、折腾家庭影院的人来说，这简直就是梦寐以求的配置：云端离线下载 + 本地播放，完美。

但光有 PikPak 还不够。你总不能每次都手动下载文件到本地吧？这时候就轮到 **rclone** 出场了——这个命令行老将专门干「让云盘像本地磁盘一样用」这件事，而且从 v1.62 版本开始就原生支持 PikPak。

本文从零开始讲清楚：PikPak rclone 怎么配置、怎么挂载、怎么配合 Emby/Jellyfin 搭起家庭影视库，还有几个常见踩坑点。

---

## 先搞清楚这套组合在干什么

简单说就三件事：

1. **PikPak 在云端帮你下载**：把磁力链接、BT 种子丢进去，PikPak 用它自己的服务器秒速完成，你不需要开着电脑等。
2. **rclone 把 PikPak 挂载成本地磁盘**：配置好之后，PikPak 里的文件就像 `/mnt/pikpak` 目录里的文件一样，你可以直接操作。
3. **Emby/Jellyfin 读取挂载目录作为媒体库**：影视库自动刮削海报、字幕，完美收入囊中。

这套组合的好处是省硬盘——文件存在 PikPak 的 10TB 空间里，本地不需要存太多东西，流媒体读取时走直链播放，流量和延迟都很友好。

---

## 第一步：注册 PikPak 账号

先得有个 PikPak 账号。

新用户通过邀请码注册，有机会获得免费 Premium 会员体验。

👉 [点击这里用邀请码 74098243 注册 PikPak](https://mypikpak.com?invitation-code=74098243)

注册好以后，记住你的登录邮箱（或手机号）和密码——后面 rclone 配置要用。

> **注意**：PikPak 的「用户名」是昵称，不能用于登录。登录 rclone 时要用注册时的邮箱或手机号（手机号需要加区号，比如中国用户是 `+86XXXXXXXXXXX`）。

---

## 第二步：安装 rclone

### Linux / macOS

官方一键安装脚本，直接跑：

bash
sudo -v ; curl https://rclone.org/install.sh | sudo bash


验证是否安装成功：

bash
rclone version


看到版本号输出就对了，确保版本在 **v1.62 或更高**，否则没有 PikPak 支持。

### Windows

去 [rclone 官网下载页](https://rclone.org/downloads/) 下载对应的 zip 包，解压后把 `rclone.exe` 放到 PATH 里就好，或者直接在解压目录用命令行操作。

---

## 第三步：配置 PikPak 远程存储

运行配置向导：

bash
rclone config


跟着下面的步骤走：

### 1. 创建新的远程配置


No remotes found, make a new one?
n/s/q> n


输入 `n`，回车。

### 2. 给这个远程起个名字


Enter name for new remote.
name> pikpak


名字随意，建议用 `pikpak` 或 `my-pikpak`，方便后续命令里识别。

### 3. 选择存储类型

这一步会列出 rclone 支持的所有云存储，找到 **PikPak** 对应的编号输入即可。由于 rclone 版本不同编号会变，找到 `PikPak` 那行输入对应数字就行。


Storage> XX   # XX 是 PikPak 对应的编号


### 4. 输入账号密码


user> your@email.com    # 注册用的邮箱或手机号（含区号）


密码输入时不显示内容，正常输入然后回车：


y/g> y
Enter the password:
password:
Confirm the password:
password:


### 5. 高级配置


Edit advanced config?
y/n>


直接回车（默认 No），普通使用不需要改高级选项。

### 6. 确认保存


Keep this "pikpak" remote?
y/e/d> y


输入 `y` 保存，然后 `q` 退出配置。

配置完成后，来验证一下是否连上了：

bash
rclone lsd pikpak:


如果看到你 PikPak 网盘里的文件夹列表，恭喜，配置成功。

---

## 第四步：常用 rclone 命令操作 PikPak

配置好之后，PikPak 就是 rclone 的一个普通「远程」，所有标准命令都能用。

### 列出文件

bash
# 列出根目录下的文件夹
rclone lsd pikpak:

# 列出某个目录的所有文件
rclone ls pikpak:"My Pack"


### 下载文件

bash
# 下载单个文件到当前目录
rclone copy -P pikpak:"My Pack/movie.mkv" .

# 下载整个目录，8 线程并发
rclone copy pikpak:"My Pack/电影" /local/movies --transfers=8 --progress


`--transfers=8` 指定并发线程数，配合 PikPak 的高速服务器，能把本地带宽跑满。

### 上传文件

bash
rclone copy /local/file.zip pikpak:"My Pack/" --progress


### 同步目录

bash
# 将本地目录同步到 PikPak（以 PikPak 为准，本地多余文件不动）
rclone sync /local/backup pikpak:"Backup/" --progress


---

## 第五步：把 PikPak 挂载为本地目录

这是最实用的玩法——让 PikPak 的内容像本地文件夹一样存在。

### 前提：安装 FUSE

Linux 需要先装 FUSE：

bash
sudo apt install -y fuse3


macOS 需要安装 macFUSE（去官网下载 `.pkg` 安装包）。

### 挂载命令

bash
# 创建挂载点
mkdir -p /mnt/pikpak

# 挂载（后台运行）
rclone mount pikpak: /mnt/pikpak \
  --daemon \
  --allow-other \
  --vfs-cache-mode writes \
  --dir-cache-time 1000h


参数解释：

| 参数 | 作用 |
|---|---|
| `--daemon` | 后台运行，关闭终端不断开 |
| `--allow-other` | 允许其他用户（如 Jellyfin 服务）访问挂载点 |
| `--vfs-cache-mode writes` | 写入时缓存到本地，提高稳定性 |
| `--dir-cache-time 1000h` | 目录缓存时间，减少 API 请求 |

挂载成功后，`ls /mnt/pikpak` 应该能看到你 PikPak 里的内容。

### 开机自动挂载

创建 systemd 服务文件：

bash
sudo nano /etc/systemd/system/rclone-pikpak.service


写入：

ini
[Unit]
Description=RClone PikPak Mount
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/rclone mount pikpak: /mnt/pikpak \
  --allow-other \
  --vfs-cache-mode writes \
  --dir-cache-time 1000h
Restart=on-failure

[Install]
WantedBy=multi-user.target


启用并启动：

bash
sudo systemctl enable rclone-pikpak
sudo systemctl start rclone-pikpak


---

## 第六步：配合 Emby / Jellyfin 搭建家庭影院

挂载好了，配合 Jellyfin 搭影视库就很直接了。

### 方式一：直接运行在系统上

在 Jellyfin 控制台里添加媒体库，选择 `/mnt/pikpak/电影` 这样的路径，它就会把 PikPak 里的内容当本地文件刮削。

### 方式二：Docker 运行 Jellyfin

把挂载目录映射进容器：

bash
docker run -d \
  --name jellyfin \
  -v /root/jellyfin/config:/config \
  -v /root/jellyfin/cache:/cache \
  -v /mnt/pikpak:/media \
  -p 8096:8096 \
  --restart unless-stopped \
  jellyfin/jellyfin


这样容器内的 `/media` 就是 PikPak 的内容，在 Jellyfin 里添加媒体库时选 `/media` 目录即可。

> **小提示**：如果遇到 Jellyfin 没有权限访问挂载目录，检查 `/etc/fuse.conf` 里是否开启了 `user_allow_other` 选项（取消这一行的注释即可）。

---

## PikPak 套餐一览

rclone 配合 PikPak 使用，会员和免费用户在速度和空间上差别比较明显。下面是 PikPak 主要套餐对比：

| 套餐 | 存储空间 | 下载速度 | 离线下载 | 参考价格 | 购买 |
|---|---|---|---|---|---|
| 免费版 | 6GB | 标准速度 | 支持（有限制） | 免费 |  [注册体验](https://mypikpak.com?invitation-code=74098243) |
| Premium 月付 | 10TB | 高速不限速 | 无限制 | 约 $5.79–$10/月 |  [立即订阅](https://mypikpak.com?invitation-code=74098243) |
| Premium 年付 | 10TB | 高速不限速 | 无限制 | 约 $57.59–$110/年 |  [年付更划算](https://mypikpak.com?invitation-code=74098243) |

> 注意：PikPak 会根据你的 IP 地址、设备语言等自动判断所在区域，提供对应区域的定价。最终价格以购买页面显示为准。通过邀请码 **74098243** 注册，有机会获得免费 Premium 体验天数。

对于自建影视库的场景，Premium 会员是必须的——免费版的速度和离线下载限制会让整个体验大打折扣。年付的性价比明显比月付高，如果确定要长期用，年付更划算。

👉 [使用邀请码注册 PikPak，有机会获赠 Premium 试用](https://mypikpak.com?invitation-code=74098243)

---

## 常见问题与排错

### Q：挂载后读取文件很慢，怎么办？

rclone 的 VFS 缓存模式对播放体验影响很大。对于媒体服务器场景，可以尝试：

bash
rclone mount pikpak: /mnt/pikpak \
  --daemon \
  --allow-other \
  --vfs-cache-mode full \
  --vfs-cache-max-size 10G \
  --buffer-size 256M


`--vfs-cache-mode full` 会把访问过的文件缓存在本地，大幅提升二次访问速度。但需要本地有足够的磁盘空间。

### Q：rclone config 配置后提示认证失败？

- 检查用户名是否是注册邮箱或带区号的手机号，不是昵称
- 密码是否正确（特别是有特殊字符时）
- 如果账号开启了两步验证，目前 rclone 的 PikPak 驱动可能不支持，建议临时关闭

### Q：挂载命令报错 `fuse: failed to exec fusermount`？

安装 fuse 依赖：

bash
sudo apt install fuse3
# 或者
sudo apt install fuse


然后检查 `/etc/fuse.conf`，确保有这一行且没有被注释：


user_allow_other


### Q：Jellyfin 刮削很慢，或者显示文件但无法播放？

这通常是网络问题。可以：

1. 确保 VPS 或 NAS 能正常访问 PikPak 的 API
2. 增大 `--buffer-size` 参数
3. 在 Jellyfin 中启用「直接播放」而不是转码，减少服务器压力

### Q：离线下载任务怎么用 rclone 触发？

rclone 的 PikPak 驱动支持 `addurl` 后端命令，可以直接从命令行添加离线下载任务：

bash
rclone backend addurl pikpak:"My Pack" "magnet:?xt=urn:btih:..."


指定文件名：

bash
rclone backend addurl pikpak:"My Pack" "https://example.com/file.zip" -o name=myfile.zip


这个功能对于脚本化批量下载非常有用，比如定时抓取 RSS 源里的磁力链接然后自动丢进 PikPak。

---

## 写在最后

PikPak + rclone 这套组合，本质上是在用云端的算力和带宽替代本地的资源。PikPak 负责搞定「怎么下」，rclone 负责搞定「怎么用」，两者分工清晰。

对于预算有限但又想玩转家庭影院的人来说，这个方案的性价比相当高——不需要买大容量硬盘，不需要担心本地下载速度，只需要一台能运行 Jellyfin 的设备和一个 PikPak Premium 账号就够了。

如果你还没有 PikPak 账号，现在注册可以通过邀请码获得免费会员体验：

👉 [注册 PikPak（邀请码：74098243）](https://mypikpak.com?invitation-code=74098243)

试试这套组合，你大概率会觉得以前的下载方式白走了很多弯路。
