# Arch 安装手册

<div style="opacity:0;height:0;">cSpell:disable</div>

记录了安装 Arch Linux 的步骤和注意事项。

## 术语速查

| 缩写  | 全称                                  | 描述                       | 特点                             |
| ----- | ------------------------------------- | -------------------------- | -------------------------------- |
| NTFS  | New Technology File System            | Windows 常用文件系统       |
| FAT   | File Allocation Table                 | 兼容性强的文件系统         |
| exFAT | Extended File Allocation Table        | 适用于大文件存储的文件系统 |
| ext4  | Fourth Extended Filesystem            | Linux 常用文件系统         |
| btrfs | B-tree File System                    | 现代 Linux 文件系统        | 支持**快照**、**子卷**的文件系统 |
| swap  | Swap Space                            | Linux 交换分区             |
| vfat  | Virtual File Allocation Table         | FAT 文件系统的变体         |
| EFI   | Extensible Firmware Interface         | 现代计算机的固件接口       |
| UEFI  | Unified Extensible Firmware Interface | EFI 的升级版               |
| ESP   | EFI System Partition                  | EFI 系统分区               |

---

| 缩写     | 全称                                 | 描述               | 特点                               |
| -------- | ------------------------------------ | ------------------ | ---------------------------------- |
| Wayland  | Wayland Display Server Protocol      | **显示服务器协议** | 现代化的显示服务器协议             |
| X11      | X Window System                      | **显示服务器协议** | 历史悠久、兼容性强的显示服务器协议 |
| Hyprland | Hyprland Wayland Compositor          | **窗口管理器**     | 基于 Wayland 的动态平铺窗口管理器  |
| GNOME    | GNU Network Object Model Environment | **桌面环境**       | 现代化、易用的桌面环境             |
| KDE      | K Desktop Environment                | **桌面环境**       | 功能丰富、可定制性强的桌面环境     |

```
应用程序
↓
显示服务器协议
↓
窗口管理器
```

## 参考资料

- [「Archlinux 究极指南 2025」从手动安装到显卡直通，最后删除 Linux](https://www.bilibili.com/video/BV1L2gxzVEgs)
- [从「Linuxmint入门」到「ArchLinux安装详解」桌面端Linux入门的最佳路径](https://www.bilibili.com/video/BV19DBqB4EY4)

## 配置文件

```bash
# 从远程服务器下载配置文件到本地
scp kuuhaku@syerii:~/.zshrc .zshrc                            # .zshrc 配置文件
scp kuuhaku@syerii:~/.p10k.zsh .p10k.zsh                      # Powerlevel10k 主题配置文件
scp kuuhaku@syerii:~/.config/hypr/hyprland.conf hyprland.conf # Hyprland 配置文件
scp kuuhaku@syerii:~/.config/ghostty/config ghostty.conf      # Ghostty 终端配置文件
```

```bash
# 把本地配置文件上传到远程服务器
scp .zshrc          kuuhaku@syerii:~/.zshrc
scp .p10k.zsh       kuuhaku@syerii:~/.p10k.zsh
scp hyprland.conf   kuuhaku@syerii:~/.config/hypr/hyprland.conf
scp ghostty.conf    kuuhaku@syerii:~/.config/ghostty/config
scp hyprlock.conf   kuuhaku@syerii:~/.config/hypr/hyprlock.conf # Hyprland 锁屏配置文件

# 把整个文件夹上传到远程服务器
scp -r pictures kuuhaku@syerii:~/Pictures
```
