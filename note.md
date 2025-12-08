第一难

启动系统

主机密码: pi141592

root 管理员(JELEE)密码:

pi141592j

pi141592pass

# 硬盘分区

```bash
# 查看硬盘分区情况
lsblk -pf

# 分区
cfdisk /dev/sda

# 格式化分区
# -F32 指定 FAT32 文件系统
mkfs.vfat -F32 /dev/sda1      # EFI 分区 (启动分区)
mkfs.btrfs /dev/sda2          # 根分区，用 btrfs 文件系统

# 创建 btrfs 分区的子卷

# 挂载根分区
# -t 指定文件系统类型
# -o 指定挂载选项
# compress=zstd 表示使用 zstd 透明压缩算法
mount -t btrfs -o compress=zstd /dev/sda2 /mnt

# 创建子卷
# 创建了三个子卷: `@`, `@home`, `@swap`
# @ 表示根目录
btrfs subvolume create /mnt/@          # 根子卷
btrfs subvolume create /mnt/@home      # home 子卷
btrfs subvolume create /mnt/@swap      # swap 子卷

# 卸载根分区
umount /mnt

# 正式挂载
# 挂载根目录
# -t btrfs 指定文件系统类型为 btrfs
# -o 指定挂载选项
# subvol=@ 指定挂载 @ 子卷
mount -t btrfs -o compress=zstd,subvol=@ /dev/sda2 /mnt                   # 挂载根子卷
mount --mkdir -t btrfs -o compress=zstd,subvol=@home /dev/sda2 /mnt/home  # 挂载 home 子卷
mount --mkdir -t btrfs -o compress=zstd,subvol=@swap /dev/sda2 /mnt/swap  # 如果需要 swap 分区的话
mount --mkdir /dev/sda1 /mnt/boot                                         # 挂载 EFI 分区，即启动目录

# 查看挂载情况
df -h

# 查看 fstab 文件内容
cat /etc/fstab
```

# 更新系统

```bash

# 设置镜像源
vim /etc/pacman.d/mirrorlist
# 更换为清华大学开源软件镜像站: Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch

# 更新密钥
pacman -Sy archlinux-keyring

# 安装系统
# pacstrap 命令用于安装 Arch Linux 基础系统到指定的挂载点
# -K 表示复制主机的密钥环
# base -> 基础系统包
# base-devel -> 基础开发包
# linux ->  Linux 内核包
# linux-firmware ->  Linux 固件包
# btrfs-progs ->  btrfs 文件系统工具包
pacstrap -K /mnt base base-devel linux linux-firmware btrfs-progs

# 安装必要的功能性软件包
# networkmanager -> 网络管理工具
# vim -> 文本编辑器
# sudo -> 提升权限工具
# intel-ucode -> Intel CPU 微码更新
pacstrap -K /mnt networkmanager vim sudo intel-ucode

# 网络管理服务启动并设置开机自启
# enable -> 设置开机自启服务
# --now 表示立即启动服务
systemctl enable --now NetworkManager

# 创建 swap 文件
# btrfs 文件系统创建 swap 文件的命令
# --size 指定 swap 文件大小
# --uuid 表示 swap 文件的 UUID
# clear 表示清除文件内容
# /mnt/swap/swapfile 指定 swap 文件的路径
btrfs filesystem mkswapfile --size 8G --uuid clear /mnt/swap/swapfile
# 启用 swap 文件
swapon /mnt/swap/swapfile
# 生成 fstab 文件，让机器启动时自动挂载分区和启用 swap 文件
# -U 表示使用 UUID 方式生成 fstab 文件
genfstab -U /mnt > /mnt/etc/fstab


# 进入新系统环境
arch-chroot /mnt

# root 密码设置
passwd

# 设置主机名
echo "Aris" > /etc/hostname

# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc # 同步硬件时钟

# 本地化设置
vim /etc/locale.gen
# 取消注释: en_US.UTF-8 UTF-8 和 zh_CN.UTF-8 UTF-8
locale-gen # 生成本地化文件

# 设置语言环境
echo "LANG=en_US.UTF-8" > /etc/locale.conf


# 安装引导程序

# grub -> 引导加载程序
# efibootmgr -> EFI 启动管理工具
pacman -S grub efibootmgr

# 安装 GRUB 到 EFI 分区
# --target 指定目标架构
# --efi-directory 指定 EFI 分区挂载点
# --bootloader-id 指定引导加载程序标识
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=BootAris

# 编辑 GRUB 源文件
vim /etc/default/grub

# 修改以下内容:
#
# GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
# ↓
# GRUB_CMDLINE_LINUX_DEFAULT="loglevel=5 nowatchdog modprobe.blacklist=iTCO_wdt"
#
# 说明:
# 删除 quiet -> 取消静默模式，显示更多启动信息
# loglevel=5 -> 设置内核日志级别为 5，显示更多的信息，便于调试
# nowatchdog -> 禁用内核看门狗，防止系统自动重启
# modprobe.blacklist=iTCO_wdt -> 禁用 Intel TCO 看门狗驱动，防止系统自动重启
#
# 添加: GRUB_DISABLE_OS_PROBER=false
# 用操作系统探测器，允许 GRUB 引导其他操作系统

# 生成 GRUB 配置文件
grub-mkconfig -o /boot/grub/grub.cfg

# 重启
exit   # 退出 chroot 环境
reboot # 重启

```

# SSH 远程登录设置

配置 SSH 服务，允许通过 `<用户名>@<主机名或 IP 地址>` 远程登录系统

```bash

# 安装 OpenSSH 服务器
pacman -S openssh
# 启动并设置开机自启 SSH 服务
systemctl enable --now sshd

# 默认无法使用 root 用户通过 SSH 登录
# 需要先修改配置文件
vim /etc/ssh/sshd_config

# 修改以下内容:
# PermitRootLogin prohibit-password
# ↓
# PermitRootLogin yes

# 重启 SSH 服务或者重启系统使配置生效
systemctl restart sshd

# 安装 mDNS 支持（avahi）
# 让主机名可以自动通过 .local 被发现
pacman -S avahi nss-mdns
systemctl enable --now avahi-daemon
# 编辑 /etc/nsswitch.conf 文件:
#
# hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
# ↓
# hosts: files mdns_minimal [NOTFOUND=return] resolve mymachines myhostname dns
#
# 说明：
# files：优先查 /etc/hosts
# mdns_minimal：允许 .local 主机名通过 mDNS（Avahi）解析
# [NOTFOUND=return]：若 mDNS 成功解析则不继续向后查找
# resolve：systemd-resolved
# mymachines / myhostname：本地 systemd 名称服务
# dns：普通 DNS 解析
```

# 配置系统

```bash
sudo pacman -Syu

# 安装 fastfetch lolcat cmatrix
# fastfetch -> 系统信息显示工具
# lolcat -> 彩虹文字显示工具
# cmatrix -> 矩阵雨屏幕特效
pacman -S fastfetch lolcat q

# 安装 htop 交互式进程查看器
pacman -S htop

# 安装组件
# gnome-desktop -> GNOME 桌面环境基础组件
# gdm -> GNOME 显示管理器
# ghostty -> 终端仿真器
# gnome-control-center -> GNOME 控制中心
# gnome-software -> GNOME 软件管理器
# flatpak -> 应用程序打包和分发系统
# xdg-desktop-portal-gnome -> GNOME 桌面门户服务
pacman -S gnome-desktop gdm ghostty gnome-control-center gnome-software flatpak xdg-desktop-portal-gnome

# 安装密码和密钥管理相关组件
# gnome-keyring -> GNOME 密钥环，用于存储密码和密钥
# libsecret -> 用于访问密码和密钥的库
sudo pacman -S gnome-keyring libsecret
# 启动并设置开机自启 GNOME Keyring 服务
systemctl --user enable --now gnome-keyring-daemon.service

# 启动并设置开机自启 GDM 服务
systemctl enable --now gdm
# 关闭图形界面登录
systemctl disable --now gdm
# 单次关闭
systemctl stop gdm

# 安装 Xorg 相关组件
sudo pacman -S xorg-server xorg-apps xorg-xinit
# 对于 GNOME on Xorg 会话，在 ~/.xinitrc 中添加
# export XDG_SESSION_TYPE=x11
# export GDK_BACKEND=x11
# exec gnome-session

# 删除不需要的软件包
pacman -Rns xorg-server xorg-apps xorg-xinit

# 安装字体
pacman -S ttf-sarasa-gothic # 思源黑体
pacman -S noto-fonts-emoji # 谷歌 Noto Emoji 字体

# Noto CJK 字体（支持中日韩字符）
pacman -S noto-fonts-cjk

# MesloLGS Nerd Font
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
fc-cache -fv

```

# 通过 pacman 安装桌面软件

```bash
# 安装 Firefox 浏览器
sudo pacman -S firefox

```

# 配置宿主机代理

确保命令行和依赖环境变量的程序能走代理

影响所有用户的命令行工具和许多依赖 Shell 环境变量的程序

但是不影响一些图形界面程序，特别是那些使用 NetworkManager 管理网络设置的程序。

1. 临时配置（当前终端会话有效，关闭终端失效）

```bash
export proxy_url="http://192.168.11.1"                          # 代理服务器地址
export proxy_port_http="10811"                                  # HTTP 代理端口
export proxy_port_socks5="10810"                                # SOCKS5 代理端口
export http_proxy="${proxy_url}:${proxy_port_http}"             # HTTP 代理
export https_proxy="${proxy_url}:${proxy_port_http}"            # HTTPS 代理
export all_proxy="socks5://${proxy_url}:${proxy_port_socks5}"   # SOCKS5 代理
export no_proxy="localhost,127.0.0.1,*.local,"                  # 不使用代理的地址列表
# 说明:
# 192.168.11.1 是代理服务器的 IP 地址，10811 和 10810 是局域网代理端口
#
# 取消代理设置
unset http_proxy
unset https_proxy
unset all_proxy
unset no_proxy
unset http_proxy https_proxy all_proxy no_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
# 因为是临时配置，关闭当前终端会话代理就会自动失效

```

2. 永久配置（对所有用户生效）

在 Arch 中设置系统级代理（对所有用户生效）

创建或编辑全局环境变量文件 `/etc/profile.d/proxy.sh`

写入以下内容:

**注意大小写**

`/etc/profile.d/proxy.sh`

```sh
# --- /etc/profile.d/proxy.sh: 全局代理配置 ---

# 配置代理地址
proxy_url="192.168.11.1"                        # 代理服务器地址
proxy_port_http="10811"                         # HTTP 代理端口
proxy_port_socks5="10810"                       # SOCKS5 代理端口
no_proxy_list="localhost,127.0.0.1,*.local,"    # 排除地址列表

# 代理设置（小写变量）
export http_proxy="http://${proxy_url}:${proxy_port_http}"
export https_proxy="http://${proxy_url}:${proxy_port_http}"
export all_proxy="socks5://${proxy_url}:${proxy_port_socks5}"
export no_proxy="${no_proxy_list}"

# 代理设置（大写变量）(兼容某些程序)
export HTTP_PROXY="http://${proxy_url}:${proxy_port_http}"
export HTTPS_PROXY="http://${proxy_url}:${proxy_port_http}"
export ALL_PROXY="socks5://${proxy_url}:${proxy_port_socks5}"
export NO_PROXY="${no_proxy_list}"

# -----------------------------------------------
```

应用配置:

`bash`

```bash
# 确保脚本文件具有执行权限
chmod +x /etc/profile.d/proxy.sh
# chmod (Change Mode): 改变权限的命令
# 在 Linux 中，每个文件都有三组权限：用户 (Owner)、群组 (Group)、其他 (Others)
# 权限包括读 (r)、写 (w)、执行 (x)
# 例如，chmod +x 表示添加执行权限
# 只有为 proxy.sh 脚本添加了 +x 权限后，系统在用户登录时才会运行它
#
# /etc/profile.d/proxy.sh: 一个特殊的系统目录
# 在大多数现代 Linux 发行版中，/etc/profile 文件（系统 Shell 启动脚本）会检查这个目录下的 所有可执行脚本
#
# 当用户登录时，系统会运行 /etc/profile
# /etc/profile 文件的一个主要任务是遍历 /etc/profile.d/ 目录，并 执行 (source) 目录下的 所有可执行脚本（通常是 .sh 文件）

# 立即生效
source /etc/profile.d/proxy.sh

# 测试连接
curl -I https://www.google.com
```

# 彩色终端

如何把终端变得更漂亮？

1. zsh (Z Shell): 一种比 bash 功能更强的 Shell
2. oh-my-zsh: 一个流行的 zsh 配置框架，提供了许多主题和插件，让 zsh 开箱即用、好看、好用、可扩展
3. Powerlevel10k: 一个功能强大且高度可定制的 oh-my-zsh 主题，提供了丰富的信息显示和美观的外观

```bash
# https://www.haoyep.com/posts/zsh-config-oh-my-zsh/

# 使用 zsh + oh-my-zsh（更漂亮的提示符）
pacman -S zsh       # 安装 zsh
chsh -s /bin/zsh    # 修改默认 shell 为 zsh
exit                # 重新登录终端即可看到效果

# 安装 oh-my-zsh
# 从 GitHub 下载 oh-my-zsh 的安装脚本，输出到标准输出（屏幕），并通过 sh 执行该脚本
# -f -> 失败时不输出 HTML 错误页
# -s -> silent，安静模式，不显示进度条
# -S -> 出错时显示错误提示（配合 -s 使用）
# -L -> 跟随跳转（如果 URL 有重定向）
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# "$( ... )" 表示将 curl 下载的脚本内容插入为字符串。
# 也就是把下载到的脚本内容塞进 sh -c 执行
# -c 表示执行后面的字符串作为命令
#
# 执行 oh-my-zsh 官方安装脚本:
# 在 ~/.oh-my-zsh/ 下载 oh-my-zsh
# 生成 ~/.zshrc
# 如果当前 shell 是 bash，会提示是否切换到 zsh

# 安装 Powerlevel10k 主题
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/.oh-my-zsh/custom/themes/powerlevel10k
# 把克隆出来的仓库放在 oh-my-zsh 的 主题目录 下
# --depth=1 表示只克隆最新的提交，节省空间和时间

# 修改 ~/.zshrc，设置 Powerlevel10k 为默认主题
#
# ZSH_THEME="robbyrussell"
# ↓
# ZSH_THEME="powerlevel10k/powerlevel10k"
#
# 重新加载 zsh 配置文件，使更改生效
source ~/.zshrc

# 重新配置 Powerlevel10k
p10k configure

# 修改配置文件 ~/.p10k.zsh，根据个人喜好调整外观
# 应用配置
source ~/.p10k.zsh


# 命令自动上色（语法高亮）
# 安装 zsh-syntax-highlighting 插件
# 克隆 zsh-syntax-highlighting 仓库到 oh-my-zsh 的自定义插件目录下
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
# 编辑 ~/.zshrc
# 添加 plugins=( [plugins...] zsh-syntax-highlighting)

# 命令提示插件，当你输入命令时，会自动推测你可能需要输入的命令，按下右键可以快速采用建议
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
# 编辑 ~/.zshrc
# 添加 plugins=( [plugins...] zsh-autosuggestions)


# 给常用命令起别名
# 编辑 ~/.zshrc
# alias ll='ls -la'
# 查看所有别名
alias
# 查看某个别名的具体命令
alias ll

# 让 Zsh 的历史记录自动去重（输入相同命令时只保留最新的一条，旧的自动删除）
# ~/.zshrc
#
# # =========================
# # 历史记录去重（保留最新）
# # =========================
# setopt HIST_IGNORE_ALL_DUPS      # 输入命令时，如果历史中有旧的，立即删除旧的
# setopt HIST_SAVE_NO_DUPS         # 保存到文件时不写重复条目
# setopt HIST_IGNORE_DUPS          # 连续重复时不写重复（可选）
# setopt HIST_FIND_NO_DUPS         # 搜索历史时不显示重复（可选）
#

# 立即生效
source ~/.zshrc


# Ghostty 终端配置
# 配置文件路径: ~/.config/ghostty/config
# 按下 ctrl+shift+, 立即刷新
```

# 创建、管理用户

```bash
# 创建普通用户
# -m -> 创建用户的同时创建 home 目录
# -G -> 指定用户所属的附加组，这里是 wheel 组（管理员组）
# -s -> 指定用户的默认 shell
useradd -m -G wheel -s /bin/zsh jElee
# 设置用户密码
passwd jElee
# 切换到新用户
su - jElee

# 允许 wheel 组用户使用 sudo 提升权限
# 编辑 sudoers 文件: /etc/sudoers
# 找到以下行并取消注释
# %wheel ALL=(ALL) ALL

# 列出所有用户
cat /etc/passwd

# 查看用户组
groups 用户名
```

# 在 Windows 资源管理器中远程访问 Arch Linux 文件系统

用 Samba 共享，需要启动的服务：

1. smb 服务：管理 SAMBA 服务器共享什么目录、文件、打印机
2. nmb 服务：管理群组和 netbios name 解析

```bash
# 安装 Samba
pacman -S samba
# 配置 /etc/samba/smb.conf
# 加入你要共享的目录，例如共享 home：
#
# [global]
#    workgroup = WORKGROUP
#    server string = Arch Samba Server
#    security = user
#    map to guest = Bad User
#
# [home]
#    path = /home/你的用户名
#    read only = no
#    browsable = yes
#
# 说明:
# [home] = 共享名（Windows 里会显示为这个）
# path = 共享的实际路径
# read only = no -> 允许写入

# 为 Samba 设置登录用户密码
smbpasswd -a Username
# 启用
smbpasswd -e Username
# 禁用
smbpasswd -d Username
# 删除 Samba 用户
smbpasswd -x Username

# 启动并设置开机自启 Samba 服务
systemctl enable --now smb nmb
systemctl restart smb nmb

# 查看 Samba 服务状态
systemctl status smb nmb
```

# VMware 虚拟机相关工具

```bash
# https://wiki.archlinuxcn.org/wiki/VMware/%E5%AE%89%E8%A3%85_Arch_Linux_%E4%B8%BA%E8%99%9A%E6%8B%9F%E6%9C%BA
# 终端复制粘贴支持
# 这些功能与 gtkmm3 有着未指明的依赖关系，并会导致这些功能静默失败
# 安装依赖
pacman -S open-vm-tools gtkmm3
# 加载 vmware 相关内核模块
# 编辑 /etc/mkinitcpio.conf 文件
# MODULES=(... vmw_balloon vmw_pvscsi vsock vmw_vsock_vmci_transport ...)
# 启动并设置开机自启 open-vm-tools 服务
# vmtoolsd -> VMware Tools 守护进程
# vmware-vmblock-fuse -> 允许主机和虚拟机之间的文件拖放和复制粘贴功能
systemctl enable --now vmtoolsd vmware-vmblock-fuse
```

# NVIDIA 显卡驱动安装

```bash
# 安装头文件
pacman -S linux-headers
# ...
```

# 修改文件权限

```bash
# 修改文件或目录的所有者
# chown (Change Owner): 改变文件或目录的所有者
chown 用户名:用户组 文件或目录路径
```

# Yay - AUR 助手

Yay 是一个流行的 AUR 助手，用于简化从 Arch User Repository (AUR) 安装和管理软件包的过程

```bash
# 安装 Yay
# 安装依赖
pacman -S git base-devel
# 克隆 Yay 仓库
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
# 安装完成后，可以使用 yay 命令来安装 AUR 软件包
# 使用 Yay 安装 AUR 软件包
yay -S 包名

# 例如，安装 Google Chrome 浏览器
yay -S google-chrome
```

# Docker

## 安装 Docker Engine 并配置代理

```bash
# 安装 Docker
pacman -S docker
# 启动并设置开机自启 Docker 服务
systemctl enable --now docker
# 或者在第一次启动Docker时启动
systemctl enable --now docker.socket

# 给 Docker Daemon 配置代理
# 编辑 /etc/systemd/system/docker.service.d/http-proxy.conf 文件
[Service]
Environment="HTTP_PROXY=http://192.168.11.1:10811"
Environment="HTTPS_PROXY=http://192.168.11.1:10811"
Environment="ALL_PROXY=socks5://192.168.11.1:10810"
Environment="NO_PROXY=localhost,127.0.0.1,::1"

# 重新加载 systemd 配置
systemctl daemon-reload
# 重启 Docker 服务使配置生效
systemctl restart docker
# 查看 Docker 服务状态
docker info

# 测试 Docker 是否安装成功
docker run -it --rm archlinux bash -c "echo hello world"


# 安装 Docker Desktop
# 下载 Docker Desktop for Linux 包
# -q -> 静默模式，不显示下载进度
# -O- -> 将下载内容输出到标准输出（屏幕）
# tar -> 解压 tar 包
# x -> 解压
# v -> 显示解压过程
# f -> 使用文件（file），此处的 - 表示从标准输入中读取压缩包内容
# z -> 处理 gzip 压缩格式
# docker/docker -> 指定 tar 解压时从包内路径 docker/docker 开始解压，docker 目录内的内容会被提取出来。
# --strip-components=1 -> 表示去掉解压后路径的第一部分目录结构。即如果压缩包内部有 docker/docker，解压后只保留 docker 文件，不保留它的父目录。
wget https://download.docker.com/linux/static/stable/x86_64/docker-29.1.2.tgz -qO- | tar xvfz - docker/docker --strip-components=1
# -r -> 递归复制目录及其内容
# -p -> 保留文件属性（权限、时间戳等）
sudo cp -rp ./docker /usr/local/bin/ && rm -r ./docker
# 安装
# -U -> 升级或安装本地包
sudo pacman -U ./docker-desktop-x86_64.pkg.tar.zst

# 安装 AppIndicator 扩展（系统托盘图标支持）
pacman -S gnome-shell-extension-appindicator

# 卸载 Docker Desktop
sudo pacman -Rns docker-desktop
# 删除残留文件
sudo rm -rf ~/.docker/desktop

```

## 关于 Docker Desktop 和 Docker Engine

```bash

# 二者独立
# https://docs.docker.com/desktop/setup/install/linux/#general-system-requirements
# 查看计算机上可用的上下文
docker context ls
# 切换到 Docker Desktop 上下文
docker context use docker-desktop
```

## 以非 root 用户身份管理 Docker

可以让 vsudo 插件访问 Docker，而不需要每次都使用 sudo 提升权限

```bash
# 创建 docker 群组
groupadd docker
# 将当前用户添加到 docker 组
# -a -> 追加用户到组
# -G -> 指定组
usermod -aG docker $USER
# 激活对组的更改
newgrp docker
# 验证
docker run hello-world
```

# 启用 KVM 硬件虚拟化

```bash
# 检查 KVM 硬件支持
LC_ALL=C.UTF-8 lscpu | grep Virtualization
# 检查内核中是否已包含必要的模块（kvm 以及 kvm_amd 和 kvm_intel 中的一个）
zgrep CONFIG_KVM /proc/config.gz
# 确认这些内核模块已自动加载
lsmod | grep kvm
```

# 安装 pass 一个密码管理工具

```bash
# 安装 pass
pacman -S pass
# 设置 GPG 密钥
gpg --full-generate-key

# 查看密钥
gpg --list-secret-keys
# 初始化密码存储库
pass init < GPG-KEY-ID >
```

# 开发工具

## Python

```bash
# pyenv -> Python 虚拟环境管理工具
pacman -S pyenv
# 编辑 ~/.zshrc 添加以下内容以配置 pyenv 环境变量
export PATH="$HOME/.pyenv/bin:$PATH"    # 将 pyenv 可执行文件路径添加到 PATH 环境变量
eval "$(pyenv init --path)"             # 初始化 pyenv 环境变量

# 安装 Python 3.11
pyenv install 3.11.9
# 设置全局 Python 版本
pyenv global 3.11.9
# 列出已安装的 Python 版本
pyenv versions
# 查看当前 Python 版本
pyenv version
```

## Java

```bash
# jenv -> Java 版本管理工具
pacman -S jenv
# 编辑 ~/.zshrc 添加以下内容以配置 jenv 环境变量
export PATH="$HOME/.jenv/bin:$PATH"    # 将 jenv 可执行文件路径添加到 PATH 环境变量
eval "$(jenv init -)"                  # 初始化 jenv 环境变量

# 下载 JDK 25
pacman -S jdk25-openjdk
# 将 JDK 25 添加到 jenv 管理
jenv add /usr/lib/jvm/java-25-openjdk
# 设置全局 Java 版本
jenv global 25
# 列出已添加的 Java 版本
jenv versions
# 查看当前 Java 版本
jenv version
```
