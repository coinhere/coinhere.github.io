---
title: 移动硬盘安装配置Archlinux+Hyprland
date: 2024-10-13 14:03:45
tags: 
 - Archlinux
 - Hyprland
 - 移动硬盘
---

之前一直使用Archlinux+KDE，也花费了一定时间来美化，自觉效果还不错，也用了较长一段时间，但KDE在使用中时常出现些奇奇怪怪的问题，十分折磨。

前几天看了几个Hyprland的视频，美观程度和流畅感简直甩了KDE两条街，当时就想投奔Hyprland，只是考虑到自己从头配置加美化要花不少时间，加上不能保证Hyprland就不会出各种各样的毛病，所以一直没有转Hyprland。

这几天拆机空出了一块硬盘，又淘了个硬盘盒，正好组成一个移动硬盘，加上GitHub上有许多美观的Hyprland的配置，直接免去了配置加美化的一环，正好试一试平铺式窗口管理的感觉如何。

配置：

- Lenovo Xiaoxin pro 14
- CPU AMD Ryzen 7 7840HS with Radeon 780M Graps
- 1TB RC20固态硬盘+硬盘盒

## Archlinux安装

我参考的是[archlinux 简明指南](https://arch.icekylin.online/)和官方的[Archlinux Installation guide](https://wiki.archlinux.org/title/Installation_guide)，[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)，[Install Arch Linux from existing Linux](https://wiki.archlinux.org/title/Install_Arch_Linux_from_existing_Linux)，以及[GRUB](https://wiki.archlinux.org/title/GRUB)。

### 安装前准备

因为是在已有的Archlinux上安装Archlinux，因此事先不需要配置网络、时钟、镜像源那些东西。

首先安装[arch-install-scripts](https://archlinux.org/packages/?name=arch-install-scripts)这个包，里面是一些安装Archlinux要用到的命令，以及[dosfstools](https://archlinux.org/packages/core/x86_64/dosfstools/)，包含格式化分区的命令。

连上硬盘，切换到Root。

### 分区格式化

之后用`cfdisk`分区，这里这里按官方的提示，增加了一个128G大小的windows的NTFS分区，专门用于跨平台存储数据，并作为第一个分区，其它按安装指南即可。

这里按简明指南使用Btrfs文件系统，因此`/home/`和`/`都在一个分区上。

{% asset_img cfdisk-example.png 分区示例 %}

格式化分区，注意将各个分区名替换成你对应的分区：

```bash
mkfs.ntfs -Q /dev/sda1 # 格式化NTFS分区
mkfs.fat -F 32 /dev/sda2 # 格式化EFI分区，双系统注意不要运行这步
mkswap /dev/sda3 # 格式化Swap分区
mkfs.btrfs -L myArch /dev/sda4 # 格式化Btrfs分区
mount -t btrfs -o compress=zstd /dev/sda4 /mnt # 挂载Btrfs分区
btrfs subvolume create /mnt/@ # 创建 / 目录子卷
btrfs subvolume create /mnt/@home # 创建 /home 目录子卷
umount /mnt
```

### 安装系统

首先挂载各个分区：

```bash
mount -t btrfs -o subvol=/@,compress=zstd /dev/sda4 /mnt # 挂载 / 目录
mount --mkdir -t btrfs -o subvol=/@home,compress=zstd /dev/sda4 /mnt/home # 挂载 /home 目录
mount --mkdir /dev/sda2 /mnt/boot # 挂载 /boot 目录
swapon /dev/sda3 # 挂载交换分区
```

因为我用于安装Archlinux的系统本身启用了swap分区，因此还要把这个系统的swap分区取消挂载，否则生成fstab文件时会把这个分区也加进去。

```bash
swapoff /dev/nvme0n1p6 # 取消原系统swap分区
mount --mkdir -t ntfs3 /dev/sda1 /mnt/media/windows # 挂载windows数据分区，用于生成fstab后自动挂载
df -h # 查看分区结果
free -h # 查看swap
```

安装系统，因为是从已有的Archlinux上安装Archlinux，因此加上`-c`参数，直接使用本地的缓存：

```bash
pacstrap -c /mnt base base-devel linux linux-firmware btrfs-progs
# 如果使用btrfs文件系统，额外安装一个btrfs-progs包
pacstrap -c /mnt amd-ucode intel-ucode # 移动硬盘跨平台，因此都AMD和intel的微码都装上
pacstrap -c /mnt sof-firmware networkmanager ntfs-3g dosfstools # 板载声卡驱动，网络，挂载NTFS分区
pacstrap -c /mnt neovim sudo fish man-db man-pages texinfo
```

生成并修改fstab文件，如果有subvolid参数需要移除btrfs分区中两个字卷的subvolid参数，避免Timeshift恢复 Btrfs 快照时，可能出现由于子卷 ID 变更导致无法挂载目录而无法进入系统。

```bash
genfstab -U /mnt >> /mnt/etc/fstab # 生成fstab文件
vim /mnt/etc/fstab # 修改fstab文件
```

### 系统设置

此处均参照[archlinux 简明指南](https://arch.icekylin.online/)，仅记录执行过的命令。
这里建议按[archlinux 简明指南](https://arch.icekylin.online/)，一步一步来：

```bash
arch-chroot /mnt
nvim /etc/hostname # 添加主机名，这里是myArch
```

设置hosts：

```bash
nvim /etc/hosts
```

加入，注意要和主机名一致：

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   myArch.localdomain myArch
```

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime # 设置时区
hwclock --systohc # 同步硬件时间
nvim /etc/locale.gen # 去掉注释 en_US.UTF-8 UTF-8 以及 zh_CN.UTF-8 UTF-8
locale-gen
echo 'LANG=en_US.UTF-8' >> /etc/locale.conf # 设置系统语言
passwd root # 设置root密码
```

按[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)的要求，需要修改`/etc/mkinitcpio.conf`文件：

```bash
nvim /etc/mkinitcpio.conf
```

在`HOOKS`中将`block`和`keyboard`移到`autodetect`之前：

```
HOOKS=(base udev keyboard block autodetect microcode modconf kms keymap consolefont filesystems fsck)
```

然后运行：

```bash
mkinitcpio -P
```

### 启动引导

首先安装grub:

```bash
pacman -S grub efibootmgr os-prober # 前两个是GRUB必需的，os-prober用于引导windows系统
```

按[Install Arch Linux on a removable medium](https://wiki.archlinux.org/title/Install_Arch_Linux_on_a_removable_medium)的要求，
运行以下命令来安装为BIOS和UEFI分别安装GRUB，并加上`--removable`以保证将硬盘移至另一台计算机时能够从硬盘启动：

```bash
grub-install --target=i386-pc /dev/sda2 --recheck # 未知原因，这个命令运行失败，可能是要另建一个BIOS分区
grub-install --target=x86_64-efi --efi-directory=/boot --removable --recheck
```

接下来修改`/etc/default/grub`文件，并生成配置：

```bash
nvim /etc/default/grub # loglevel=5 nowatchdog 取消注释最后一行启用os-prober
grub-mkconfig -o /boot/grub/grub.cfg
```

完成安装，退出系统：

```bash
exit # 退回安装环境
umount -R /mnt # 卸载新分区
reboot # 重启
```

### 重启后一般设置

此处均参照[archlinux 简明指南](https://arch.icekylin.online/)，仅记录执行过的命令。

#### 连接网络

```bash
systemctl enable --now NetworkManager
# 如果使用WiFi，运行以下命令
nmcli dev wifi list
nmcli dev wifi connect "wifi SSID" password "password"
```

#### 准备普通用户

```bash
useradd -m -G wheel -s /bin/bash myusername # 创建用户及用户家目录
passwd myusername # 设置密码
EDITOR=nvim visudo # 启用用户组权限，注释掉#%wheel ALL=(ALL:ALL) ALL
```

#### 用户设置

更改默认编辑器，在`/root/.bash_profile`及`/home/myusername/.bash_profile`中加入：

```
export EDITOR='nvim'
```

更改默认shell:

```bash
su myusername # 切换用户
chsh -l # 查看安装了哪些 Shell
chsh -s /usr/bin/fish # 修改当前账户的默认 Shell 为fish
```

#### 开启 32 位支持库与 Arch Linux 中文社区仓库

修改`/etc/pacman.conf`：

取消下面两行的注释：

```conf
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

并添加：

```conf
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch # 中国科学技术大学开源镜像站
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch # 清华大学开源软件镜像站
Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch # 哈尔滨工业大学开源镜像站
Server = https://repo.huaweicloud.com/archlinuxcn/$arch # 华为开源镜像站
```

#### 更新`pacman`数据库，安装一些基础功能包

```bash
pacman -Syyu
sudo pacman -S sof-firmware alsa-firmware alsa-ucm-conf # 声音固件
sudo pacman -S adobe-source-han-serif-cn-fonts wqy-zenhei # 安装几个开源中文字体。一般装上文泉驿就能解决大多 wine 应用中文方块的问题
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra # 安装谷歌开源字体及表情
sudo pacman -S archlinuxcn-keyring # cn 源中的签名（archlinuxcn-keyring 在 archlinuxcn）
sudo pacman -S yay # yay 命令可以让用户安装 AUR 中的软件（yay 在 archlinuxcn）
yay -S ttf-ms-win11-auto-zh_cn # 微软字体
```

### 设置`timeshift`备份

建议在安装驱动和Hyprland前备份，我习惯在安装hyprland前将显卡驱动装好。

```bash
sudo pacman -S timeshift
```

修改`/etc/timeshift/timeshift.json`如下，运行`lsblk`查看UUID：

```json
{
  "backup_device_uuid" : "21b2199f-38e4-4bf1-ae1e-0c9d4d0431f8",
  "parent_device_uuid" : "",
  "do_first_run" : "false",
  "btrfs_mode" : "true",
  "include_btrfs_home_for_backup" : "true",
  "include_btrfs_home_for_restore" : "false",
  "stop_cron_emails" : "true",
  "schedule_monthly" : "false",
  "schedule_weekly" : "false",
  "schedule_daily" : "true",
  "schedule_hourly" : "false",
  "schedule_boot" : "false",
  "count_monthly" : "2",
  "count_weekly" : "3",
  "count_daily" : "5",
  "count_hourly" : "6",
  "count_boot" : "5",
  "snapshot_size" : "0",
  "snapshot_count" : "0",
  "date_format" : "%Y-%m-%d %H:%M:%S",
  "exclude" : [],
  "exclude-apps" : []
}
```

安装Hyprland之后，如果遇到timeshift GUI 无法启动的情况，需要安装`xorg-xhost`，原因见[arch wiki](https://wiki.archlinux.org/title/Timeshift#Timeshift_GUI_not_launching_on_Wayland)

```bash
sudo pacman -S xorg-xhost
```

这里先用命令行启用备份，可见[简明教程里的介绍](https://arch.icekylin.online/guide/advanced/system-ctl.html#%E7%B3%BB%E7%BB%9F%E5%BF%AB%E7%85%A7-%E5%A4%87%E4%BB%BD-%E4%B8%8E%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93)：

```bash
timeshift --list # 查看快照
timeshift --create --comments "after update" --tags D # 创建快照，标签为每日 
# timeshift  --restore  --snapshot '2014-10-12_16-29-08' --skip-grub # 恢复指定快照，并跳过grub
```

### 安装显卡驱动

**这一步可以跳过，因为安装HyDE的时候，会自动检测显卡并安装驱动及设定相关参数，包括NVIDIA**

没有什么说的比[简明教程](https://arch.icekylin.online/guide/rookie/graphic-driver)，和Arch wiki说的更清楚的了。

因为是移动硬盘，且主力机是40系显卡配Intel核显以及AMD核显笔记本，我安装的是：

```bash
sudo pacman -S mesa lib32-mesa vulkan-intel lib32-vulkan-intel # Intel 核芯显卡
sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon #AMD 集成显卡
sudo pacman -S nvidia nvidia-settings lib32-nvidia-utils # NVIDIA 独立显卡
```

Arch Wiki 提示：
> NVIDIA还需要将 `kms` 从 `0/etc/mkinitcpio.conf` 里的 `HOOKS` 数组中移除，并重新生成 `initramfs`。 这能防止 `initramfs` 包含 `nouveau` 模块，以确保内核在早启动阶段不会加载它。

在`HOOKS`中，删除`kms`

```conf
HOOKS=(base udev keyboard block autodetect microcode modconf keymap consolefont filesystems fsck)
```

然后运行：

```bash
sudo mkinitcpio -P
```

此外Arch Wiki提到，Wayland + Nvidia 还需要设置[DRM 内核级显示模式设置](https://wiki.archlinuxcn.org/wiki/Wayland#%E7%B3%BB%E7%BB%9F%E9%9C%80%E6%B1%82)，不然可能会导致黑屏：

设置[环境变量](https://wiki.archlinuxcn.org/wiki/%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F)，在`/etc/environment`中添加:

```
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia
```

## Hyprland安装

推荐Hyprland的配置，是[prasanthrangan's HYDE](https://github.com/prasanthrangan/hyprdots)，简洁、干净、美观，包含多个不同风格的主题和基本的一些应用，适合于想要在一个美观的主题上搭建自己的Hyprland的用户，这也是GitHub上目前Star数最多的Hyprland配置。

原来的仓库已经不再维护，新的安装见：<https://github.com/Hyde-project/hyde>

此后便可按照自己的喜好安装其它程序以及更改配置了。

## Archlinux配置

### 服务启用

Hyprland没有启用蓝牙服务和Timeshift自动备份服务，因此手动启动：

```bash
sudo pacman -S bluez bluez-utils # HYDE 会自动安装这两个包
sudo systemctl enable --now bluetooth # 启用蓝牙服务
sudo systemctl enable --now cronie # 启用Timeshift自动备份
```

### 改键

需求：

- `CapsLock`单击为`Escape`
- `CapsLock` + `f,b,p,n,a,e,u,d` = `right`, `left`, `up`, `down`, `home`, `end`, `pageup`, `pagedown`
- `CapsLock` + `h,j,k,l` = `left`, `down`, `up`, `right`
- `Escape`为`CapsLock`
- 右`Ctrl`键与右`Alt`键互换

这里选用[keyd](https://github.com/rvaiya/keyd)来改键，如果只是单纯交换键位可以使用Hyprland自带的[改键配置]()，

安装并启用keyd：

```bash
sudo pacman -S keyd
sudo systemctl enable --now keyd
```

要查看各个键位的名称，运行并按下要查看的键位：

```bash
sudo keyd monitor
```

添加配置文件`/etc/keyd/default.conf`，如下所示：

```conf
[ids]

*

[main]

# Maps capslock to escape and a self-defined key -- capslock_layer
capslock = overload(capslock_layer, esc)

# Remaps the escape key to capslock
esc = capslock

# Swap rightalt and rightcontrol
rightalt = rightcontrol
rightctrol = rightalt
rightshift = rightshift

# When capslock_layer is pressed
[capslock_layer]
f = right
b = left
p = up
n = down
a = home
e = end
u = pageup
d = pagedown
h = left
j = down
k = up
l = right
```

运行以下命令来重载配置：

```bash
sudo keyd reload
```

### fish shell

安装并更改用户默认shell:

```bash
sudo pacman -S fish
chsh -l # 查看安装了哪些 Shell
chsh -s /usr/bin/fish # 修改当前账户的默认 Shell
```

更改fish编辑器，在`~/.config/fish/config.fish`中添加下一行:

见<https://fishshell.com/docs/current/language.html#exporting-variables>

```fish
set -gx EDITOR nvim
set -gx MANPAGER 'nvim +Man!' # 使用nvim来查看man-pages，自动高亮
```

#### prompt -- starship

HyDE为fish自动安装并启用[starship](https://github.com/starship/starship)

配置`~/.config/starship.toml`:

```toml
[directory]
fish_style_pwd_dir_length = 1
```

#### 插件

插件管理器使用[Fisher](https://github.com/jorgebucaran/fisher):

安装Fisher：

```fish
curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish | source && fisher install jorgebucaran/fisher
```

推荐插件，主要推荐fzf-fish：

- [fzf-fish](https://github.com/PatrickF1/fzf.fish) -- Fzf plugin for Fish
- [autopair](https://github.com/jorgebucaran/autopair.fish)
- [plugin-git](https://github.com/jhillyerd/plugin-git) -- Git aliases plugin for the Fish shell (similar to oh-my-zsh git)
- [fish-abbreviation-tips](https://github.com/gazorby/fish-abbreviation-tips?tab=readme-ov-file) -- Help you remembering your abbreviations
- [puffer-fish](https://github.com/nickeb96/puffer-fish) -- Text Expansions for Fish (.. for ../.. and !! for the previous cmd, etc)

##### fzf-fish配置

为`fzf-fish`使用catppuccin主题:<https://github.com/catppuccin/fzf>

fzf-fish使用`bat`预览代码，更换bat高亮主题为catppuccin:<https://github.com/catppuccin/bat>

```fish
# fzf-fish
# ctrl+O 用EDITOR打开文件，ctrl-d显示目录, ctrl-h显示隐藏，ctrl-f恢复
set fzf_directory_opts "--bind=ctrl-o:execute($EDITOR {} &> /dev/tty),ctrl-d:reload(fd --color=always --type directory),ctrl-h:reload(fd --color=always --hidden),ctrl-f:reload(fd --color=always)"
set fzf_preview_dir_cmd eza -lha --icons=auto --sort=name --group-directories-first --color=always # 用eza显示目录信息
```

#### zoxide 快捷跳转

```bash
sudo pacman -S zoxide
```

将下行加入到`~/.config/fish/config.fish`--仅用于`fish shell`:

`--cmd cd`用`cd`替换`z`，`cdi`替换`zi`

```fish
zoxide init --cmd cd fish | source # start zoxide and replace with `cd`
```

#### yazi 终端文件系统

查看安装指南，安装其他拓展：<https://yazi-rs.github.io/docs/installation/>

- nerd-fonts (recommended)
- ffmpegthumbnailer (for video thumbnails)
- 7-Zip (for archive extraction and preview)
- jq (for JSON preview)
- poppler (for PDF preview)
- fd (for file searching)
- rg (for file content searching)
- fzf (for quick file subtree navigation)
- zoxide (for historical directories navigation)
- ImageMagick (for SVG, Font, HEIC, and JPEG XL preview)
- xclip / wl-clipboard / xsel (for system clipboard support)

```bash
sudo pacman -S yazi ffmpegthumbnailer p7zip jq poppler fd ripgrep fzf zoxide imagemagick
```

添加Shell wrapper以便yazi能够更改当前目录

创建文件`~/.config/fish/functions/y.fish`并添加：

```fish
function y
 set tmp (mktemp -t "yazi-cwd.XXXXXX")
 yazi $argv --cwd-file="$tmp"
 if set cwd (command cat -- "$tmp"); and [ -n "$cwd" ]; and [ "$cwd" != "$PWD" ]
  builtin cd -- "$cwd"
 end
 rm -f -- "$tmp"
end
```

更改主题，见<https://github.com/yazi-rs/flavors?tab=readme-ov-file>

### timeshift备份

直接运行timeshift-gtk，或点击图标运行，会发现无法打开图形界面。详细说明见<https://wiki.archlinux.org/title/Running_GUI_applications_as_root#Wayland>

解决方法包括：

- `xorg-xhost`，原因见[arch wiki](https://wiki.archlinux.org/title/Timeshift#Timeshift_GUI_not_launching_on_Wayland)

- 运行以下命令打开timeshift:

```bash
sudo -E timeshift-gtk # -E 表示保留当前用户环境
```

详细解释见<https://github.com/linuxmint/timeshift/issues/147>

### 输入法安装配置

#### 安装输入法相关包

```bash
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-configtool # 输入法设置工具
sudo pacman -S fcitx5-rime # 安装rime输入法
```

#### 添加环境变量

创建`~/.config/environment.d/im.conf`，并添加(详细信息见[fcitx5 in wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#Chromium_.2F_Electron))：

```conf
XMODIFIERS=@im=fcitx
QT_IM_MODULE=fcitx
```

或者在`~/.config/hypr/userprefs.conf`中添加：

```conf
env = XMODIFIERS,@im=fcitx
env = QT_IM_MODULE,fcitx
```

#### 启动fcitx5，安装rime输入法

```bash
fcitx5 -d # 启动fcitx5
```

在终端中运行`fcitx5-configtool`(或者右键任务栏的输入法图标，点击configure)，打开设置界面，取消勾选`Only Show Current Language`，在上方搜索Rime并将其移动到左侧，点击`Apply`并退出即可。

此时，使用`Ctrl`+`Space`即可切换至`Rime`并输入中文。

第一次使用时，会自动生成rime配置文件，在`~/.local/share/fcitx5/rime`中。

#### 仅设置`RightShift`切换中英文输入法

`RightShift`默认是`LeftShift`需要用`keyd`改键
设置`Trigger Input Method`即可。

#### 使用雾凇拼音词库

```bash
yay -S rime-ice # 雾凇拼音输入方案

```

创建`~/.local/share/fcitx5/rime/default.custom.yaml`并添加：

```yaml
patch:
  # 仅使用「雾凇拼音」的默认配置，配置此行即可
  __include: rime_ice_suggestion:/
  # 以下根据自己所需自行定义
  __patch:
    menu/page_size: 7 #候选词个数
    schema_list: # 不使用小鹤双拼去掉下两行
      - schema: double_pinyin_flypy

  # 禁止`shift`切换Rime输入法中英文，仅通过切换输入法切换中英文，因为Rime输入法的中英文切换是全局的，而输入法是可以局限在应用的（一个应用对应一个输入法，对应是否中英文）
  ascii_composer/switch_key/Shift_L: noop
  ascii_composer/switch_key/Shift_R: noop
```

见<https://github.com/iDvel/rime-ice/tree/main/others/%E5%8F%8C%E6%8B%BC%E8%A1%A5%E4%B8%81%E7%A4%BA%E4%BE%8B>

```double_pinyin_flypy.custom.yaml
patch:
  translator/preedit_format: []
```

之后重新启动fcitx5(右键图标点击`restart`)，`Ctrl+Space`即可输入中文。

记得设置自动启动，在`~/.config/hypr/hyprland.conf`:

```conf
exec-once = fcitx5 --replace -d
```

#### 输入法美化

这里使用fcitx5的[FluentDark](https://github.com/Reverier-Xu/Fluent-fcitx5)皮肤，透明黑色。

安装：

```bash
# Dark theme
yay -S fcitx5-skin-fluentdark-git
# Light theme
yay -S fcitx5-skin-fluentlight-git
```

之后进入fcitx5-configtool，在`Addons`-`UI`-`Classic User Interface`中，在`Theme`和`Dark Theme`中下拉选中自己想要的主题即可。

### kitty

#### 终端字体

我使用[Maple Font](https://github.com/subframe7536/maple-font)，支持连字符。

Release中下载MapleMono-NF-CN.zip，解压并放在`~/.local/share/fonts/`中。

在kitty配置文件`~/.config/kitty/kitty.conf`(HyDE中直接在`~/.config/kitty/userprefs.conf`里修改即可)将字体修改为`Maple Mono NF CN`

#### Kitty点击链接时浏览器为`Brave`，无缩放

解决方法：设置默认浏览器为Firefox：

```bash
xdg-settings set default-web-browser firefox.desktop
```

或者：

> Open the browser >> navigate chrome://flags/ >> search for Preferred Ozone platform >> Select wayland

以及：

```bash
flatpak uninstall Brave
```

并在`~/HyDE/Scripts/.extra/custom_flat.lst`修改HyDE自动安装的flatpak包

#### 类neovide光标拖尾特效

```conf
cursor_trail 3
```

### neovim

#### neovim配置

直接使用[lazyvim](http://www.lazyvim.org/installation)的配置。

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
nvim
```

安装成功后在nvim中运行`:LazyHealth`、`:Mason`、`:MasonLog`检查是否存在问题。

`:LazyExtra`查看额外配置。

#### neovim切换目录到当前文件目录

```keymap.lua
-- cd to directory of current file
vim.keymap.set("n", "<leader>cd", "<cmd>cd %:p:h<cr><cmd>pwd<cr>")
```

#### neovim切换Copilot补全

`<leader>at`切换是否启用Copilot补全:

```keymap.lua
-- Auto toggle Copilot completion
-- Reference: https://github.com/LazyVim/LazyVim/discussions/4232
local copilot_exists = pcall(require, "copilot")
if copilot_exists then
  Snacks.toggle({
    name = "Copilot Completion",
    -- color = {
    --   enabled = "azure",
    --   disabled = "orange",
    -- },
    get = function()
      return not require("copilot.client").is_disabled()
    end,
    set = function(state)
      if state then
        require("copilot.command").enable()
      else
        require("copilot.command").disable()
      end
    end,
  }):map("<leader>at")
end
```

#### blink cmdline补全以及使用Tab补全

LazyVim默认取消cmdline补全，并且用`enter`补全

这里使用`tab`进行补全，确保`enter`始终是换行和执行命令

```blink.lua
return {
  {
    "saghen/blink.cmp",
    event = { "CmdlineEnter", "InsertEnter" },
    opts = function(_, opts)
      opts.cmdline.enabled = true
      opts.keymap.preset = "super-tab"

      -- added due to bug in <https://github.com/Saghen/blink.cmp/issues/1247>
      if vim.g.neovide then
        opts.completion.accept.dot_repeat = false
      end
    end,
  },
}
```

#### neovide

安装[neovide](https://neovide.dev/)(一个Neovim的图形用户界面)以获得更好的视觉及输入体验：

```bash
sudo pacman -S neovide
```

记得在`~/.config/hypr/keybindings.conf`中添加启动快捷键：

```conf
bindd = $mainMod, N, $d start neovide, exec, neovide # launch terminal emulator
```

neovide自动读取neovim的设置，因此neovide的设置也放在neovim中，如`~/.config/nvim/lua/config/options.lua`

```lua
-- neovide configurations
if vim.g.neovide then
  vim.o.guifont = "Maple Mono NF CN:h9" -- 设置neovide字体
  vim.g.neovide_scale_factor = 1.0 -- neovide缩放与系统缩放叠加了，因此取消neovide的缩放
  vim.g.neovide_refresh_rate = 120 -- 设置刷新率为120Hz
end
```

`~/.config/nvim/lua/config/keymaps.lua`:

```lua
-- neovide keymaps
if vim.g.neovide then
  -- paste with Ctrl+shift+V
  vim.keymap.set({ "i", "c" }, "<CS-v>", "<C-r>+")
  vim.keymap.set({ "n", "v" }, "<CS-v>", '"+p')
end
```

#### neovim主题美化

创建并添加`~/.config/nvim/lua/plugins/colorscheme.lua`:

添加的几种主题是为了适配不同的HyDE主题

```lua
return {
  { "rose-pine/neovim", name = "rose-pine" },
  { "EdenEast/nightfox.nvim" },
  { "sainnhe/gruvbox-material" },
  -- 设置term_colors避免neovide中内置terminal颜色不对
  { "catppuccin/nvim", name = "catppuccin", opts = { term_colors = true, dim_inactive = { enabled = true } } },
  {
    "LazyVim/LazyVim",
    opts = {
      colorscheme = "catppuccin-mocha",
    },
  },
}
```

#### HyDE切换主题时自动切换neovim的主题

`Super`+`Shit`+`T`切换HyDE主题时，hyprland、waybar、kitty、kvantu、rofi、gtk的主题也会同时切换，但neovim的主题不会，因此需要手动更改neovim主题，不然会与终端主题十分割裂

见github上的讨论<https://github.com/prasanthrangan/hyprdots/issues/1225>

ChatGPT帮我写了一个脚本，用于切换HyDE主题时，同时切换neovim主题，见<https://gist.github.com/coinhere/63bcab29bdabec1ec6b6c8ab263fa058>

##### 工作原理

- 切换HyDE主题时，从stdout输出中获取切换后的主题名字
- 将主题对应上neovim的主题名字后，替换neovim主题配置文件中的*设定主题一行*，以永久切换主题
- 通过`pynvim`发送切换主题的命令给所有的neovim实例，以实时切换主题

##### 注意事项

- 没有为HyDE上的所有主题给出了对应的neovim主题（除部分外其余均使用默认主题），你可以自行添加或修改对应的neovim主题

- 这里使用`LazyVim`的配置，创建的neovim主题配置文件为`~/.config/nvim/lua/plugins/colorscheme.lua`，更换neovim主题只需替换`colorscheme`一行：

```lua
  {
        "LazyVim/LazyVim",
        opts = {
          colorscheme = "catppuccin-mocha",
        },
      },
```

你可能需要根据自己的配置文件，修改替换文件这部分代码

- 脚本使用`pynvim`给nvim实例发送命令，需要提前安装这个python包
- 这里系统上nvim实例默认的socket路径皆在"/run/user/1000"，可以在nvim中运行`:echo v:servername`查看监听路径，你可能需要修改代码中的对应路径，以防脚本找不到所有nvim实例的接口

#### Leetcode neovim

在neovim中刷Leetcode，见<https://github.com/kawre/leetcode.nvim>:

添加`~/.config/nvim/lua/plugins/leetcode.lua`:

```lua
return {
  {
    "folke/snacks.nvim",
    opts = function(_, opts)
      local projects = { icon = " ", key = "t", desc = "Leet Code", action = ":Leet" }
      table.insert(opts.dashboard.preset.keys, 9, projects)
    end,
  },
  {
    "folke/which-key.nvim",
    opts = {
      spec = {
        { "<leader>l", group = "+leetcode" },
      },
    },
  },
  {
    "kawre/leetcode.nvim",
    cmd = "Leet",
    lazy = true,
    build = ":TSUpdate html", -- if you have `nvim-treesitter` installed
    dependencies = {
      "nvim-telescope/telescope.nvim",
      -- "ibhagwan/fzf-lua",
      "nvim-lua/plenary.nvim",
      "MunifTanjim/nui.nvim",
    },
    opts = {
      ---@type lc.lang
      lang = "python3",

      cn = { -- leetcode.cn
        enabled = true, ---@type boolean
        translator = false, ---@type boolean
        translate_problems = false, ---@type boolean
      },

      -- hooks = {
      --   ---@type fun()[]
      --   ["enter"] = {
      --     function()
      --       vim.wo.winfixbuf = false
      --     end,
      --   },
      --
      --   ---@type fun(question: lc.ui.Question)[]
      --   ["question_enter"] = {
      --     function()
      --       vim.wo.winfixbuf = false
      --     end,
      --   },
      -- },
    }, -- configuration goes here
    keys = {
      { "<leader>l", false },
      { "<leader>lz", "<cmd>Lazy<cr>", desc = "Lazy" },
      { "<leader>lm", "<cmd>Leet<cr>", desc = "Leetcode menu" },
      { "<leader>lc", "<cmd>Leet console<cr>", desc = "Leetcode console" },
      { "<leader>li", "<cmd>Leet info<cr>", desc = "Leetcode info" },
      { "<leader>l<tab>", "<cmd>Leet tabs<cr>", desc = "Leetcode tabs" },
      { "<leader>ly", "<cmd>Leet yank<cr>", desc = "Leetcode yank" },
      { "<leader>la", "<cmd>Leet lang<cr>", desc = "Leetcode lang" },
      { "<leader>l<cr>", "<cmd>Leet run<cr>", desc = "Leetcode run" },
      { "<leader>ls", "<cmd>Leet submit<cr>", desc = "Leetcode submit" },
      { "<leader>ll", "<cmd>Leet list<cr>", desc = "Leetcode list" },
      { "<leader>lo", "<cmd>Leet open<cr>", desc = "Leetcode open" },
      { "<leader>lr", "<cmd>Leet reset<cr>", desc = "Leetcode reset" },
      { "<leader>lt", "<cmd>Leet last_submit<cr>", desc = "Leetcode last_submit" },
      { "<leader>ld", "<cmd>Leet desc<cr>", desc = "Leetcode desc" },
    },
  },
}
```

### vscode

#### vscode无法输入中文

在`~/.config/code-flags.conf`中加入：

```conf
--enable-wayland-ime
```

#### catppuccin主题美化

- [vscode](https://github.com/catppuccin/vscode)
- [vscode-icons](https://github.com/catppuccin/vscode-icons)

### Firefox

#### 取消触摸板滑动惯性

触摸板滑动后页面仍然会滑动直至速度降为0，就像有惯性一样。

实际效果经常是，两根手指稍微动了1毫米，页面直接滑到底了。

降低滑动速度只能稍微减缓症状，索性直接禁止了。

在`about:config`将`apz.gtk.kinetic_scroll.enabled`改为false

#### firefox字体

Hyprlan默认的字体有些奇怪，这里修改字体设置。需要安装Windows字体。

在`Settings`-`Fonts`-`Advanced`-`Fonts for Simplified Chinese`，设置为：

{% asset_img font-settings.png 字体设置 %}

#### github字体

在github的上查看代码，一些nerd font的字符不会显示，github也不支持自定义字体，需要通过外置插件来修改。

这里通过插件[Stylus](https://chromewebstore.google.com/detail/stylus/clngdbkpkpeebahjckkjfobafhncgmne)/[Stylus for Firefox](https://addons.mozilla.org/en-US/firefox/addon/styl-us/)来更改默认字体。

安装后在Stylus里下载`Github Custom Fonts`，并将字体设置为喜欢的nerd font即可。
这里是`Maple Mono NF CN`。

#### 插件

- onetab
- 欧路词典
- Vimium -- 类vim按键浏览网页，全键盘工作必备
- Infinity New Tab
- Stylus
- Tampermonkey -- 油猴脚本

#### 取消视频自动静音

在`Settings`中搜索`Autoplay`，将`Default for all websites:`从`Block Audio`修改为`Allow Audio and Video`

### wayland-electron设置

将需要修改的`desktop`文件从`/usr/share/applications/`复制到`~/.local/share/applications/`，再加上：

```desktop
--enable-features=WebRTCPipeWireCapturer --ozone-platform-hint=auto --enable-wayland-ime
```

参数意义见<https://wiki.archlinux.org/title/Wayland#Electron>

#### 设置Electron配置文件

详细说明见<https://wiki.archlinux.org/title/Wayland#Configuration_file>

如`~/.config/electron-flags.conf`：

```conf
--enable-features=WaylandWindowDecorations
--ozone-platform-hint=auto
--enable-wayland-ime
```

以及`~/.config/electron13-flags.conf`：

```conf
--enable-features=UseOzonePlatform
--ozone-platform=wayland
--enable-wayland-ime
```

### openRGB 光污染必备

```bash
sudo pacman -S openrgb
```

设置自动启动：

启动后加载灯效配置后会自动退出

```conf
exec-once = openrgb --profile "your-profile-name"
```

### npm换源

```bash
npm config set registry=https://registry.npmmirror.com # 最新淘宝源
```

### btop 类似任务管理器

```bash
sudo pacman -S btop
```

安装后btop没有显示AMD的GPU信息，查看[README](https://github.com/aristocratos/btop?tab=readme-ov-file#gpu-compatibility)发现是少了一个依赖包，Archlinux运行下面的命令安装：

```bash
sudo pacman -S rocm-smi-lib
```

catppuccin主题安装<https://github.com/catppuccin/btop>

设置下拉式btop窗口，方便随时查看:

见[Pyprland Scratchpads](#pyprland)

## Hyprland配置

详细见[Hyprland Wiki](https://wiki.hyprland.org/Getting-Started/Preconfigured-setups/#prasanthrangan)

### 调整屏幕刷新率

HYDE中设置屏幕参数在`~/.config/hypr/monitors.conf`，详细设置查看<https://wiki.hyprland.org/Configuring/Monitors/>

默认是：

```conf
monitor = ,preferred,auto,auto
```

首先查看屏幕支持的分辨率和刷新率设置:

```bash
hyprctl monitors
```

，选择你需要的设置，添加到`~/.config/hypr/monitors.conf`中，例如:

```conf
monitor = ,2880x1800@120.00,auto,auto
```

> *注意*：如果你使用的电脑和笔者一样是`联想小新14Pro 2023`，且搭载的CPU是`AMD7840HS`，那么你会发现运行`hyprctl monitor`的结果中没有刷新率为120Hz的显示器设置，然而在Windows中可以正常应用120HZ，并且在Hyprland中强制使用120Hz会发现屏幕闪烁、变色、模糊。
>
> 这是因为小新主板提供的EDID信息（主板提供给操作系统显示器的信息，包括可使用的分辨率和刷新率）的校验和错误，需要将错误的EDID反编译、更正再编译后加载进内核，方能在Hyprland中使用正常的120Hz，Bug探讨和详细的解决方法见<https://bbs.archlinux.org/viewtopic.php?id=289701>。
>
### Hyprland Variable 配置

Hyprland配置文件为`~/.config/hypr/hyprland.conf`，HyDE推荐修改`~/.config/hypr/userprefs.conf`，这会覆盖`hyprland.conf`中的内容：

```conf
input {
  touchpad {
    natural_scroll = true # 反转触摸板下滑方向
  }
}

cursor {
  inactive_timeout = 5 # 光标不动5秒后自动隐藏
}

# group bar颜色调整
group {
  groupbar {
    col.active = rgba(a6e3a188)
    col.inactive = rgba(6c708688)
    col.locked_active = rgba(a6e3a188)
    col.locked_inactive = rgba(6c708688)
  }
}
```

### 配置window rules

```conf
windowrulev2 = opacity 0.90 0.90,class:^(google-chrome)$
windowrulev2 = opacity 0.80 0.80,class:^(kitty)(.*)$|^(neovide)$ # 透明kitty的下拉窗口和neovide
windowrulev2 = opacity 0.90 0.90,class:^(steamwebhelper)$
windowrulev2 = noblur,class:^(kitty)(.*)$|^(neovide)$,focus:0 # 未锁定的kitty和neovide窗口取消模糊
windowrulev2 = noblur,class:^(qalculate-gtk)$,focus:0 # 未锁定的qalculate-gtk窗口取消模糊
windowrulev2 = bordercolor rgba(d20f39ff) rgba(fe640bff) 45deg, fullscreen:1 # 最大化窗口时改变边框颜色
windowrulev2 = float,class:^([Ss]potify)$
layerrule = order -1, mpvpaper # 避免mpvpaper桌面被覆盖
```

### pyprland

[pyprland](https://hyprland-community.github.io/pyprland/)自带一些非常有用的插件，这里用到的只是`scratchpads`，可以用来设置想要的下拉式窗口：

这里设置的下拉式窗口有：

- 下拉式终端，类似KDE中的yakuake
- btop
- volume，随时调整音量

```pyprland.toml
[pyprland]
plugins = ["scratchpads"]

[scratchpads.term]
animation = "fromTop"
command = "kitty --class kitty-dropterm --title kitty-dropterm"
class = "kitty-dropterm"
size = "75% 50%"                                                # percent of full screen
max_size = "1920px 100%"
margin = "7%"                                                   # percent of half screen
offset = "210%"                                                 # percent of half size, offset = (2*size + margin)/size

[scratchpads.dropbtop]
animation = "fromBottom"
command = "kitty --class kitty-btop --title kitty-btop btop"
class = "kitty-btop"
size = "75% 75%"                                             # percent of full screen
max_size = "1920px 100%"
margin = "25%"                                               # percent of half screen
offset = "233%"                                              # percent of half size, offset = (2*size + margin)/size

[scratchpads.volume]
animation = "fromRight"
command = "pavucontrol"
class = "org.pulseaudio.pavucontrol"
size = "20% 90%"
unfocus = "hide"
lazy = true
margin = "2%"                        # percent of half screen
offset = "210%"                      # percent of half size, offset = (2*size + margin)/size

[scratchpads.qalculate]
animation = "fromLeft"
command = "qalculate-gtk"
class = "qalculate-gtk"
size = "40% 50%"          # percent of full screen
lazy = true
margin = "2%"             # percent of half screen
offset = "205%"           # percent of half size, offset = (2*size + margin)/size
```

还需要设置自动启动和快捷键：

```conf
exec-once = /usr/bin/pypr

# pyprland scratchpad
bindd = ,F12, $d toggle term, exec,pypr toggle term
bindd = ,F9, $d toggle btop, exec,pypr toggle dropbtop
bindd = ,F1, $d toggle volume, exec,pypr toggle volume
bindd = ,F2, $d toggle qalculate, exec,pypr toggle qalculate
```

### waybar任务栏设置

waybar配置文件为`~/.config/waybar/config.jsonc`，HyDE中该文件是根据`~/.config/waybar/config.ctl`自动生成的。

`~/.config/waybar/config.ctl`中，第一个数字代表正在启用的配置，第二个数字代表高度。

我的设置为：

```conf
1|28|top|( idle_inhibitor clock##custom ) ( network group/hardware custom/updates battery ) ( custom/cava custom/lyrics )|( hyprland/workspaces##custom wlr/taskbar##custom )|( mpris##custom pulseaudio pulseaudio#microphone backlight ) ( tray ) ( custom/cliphist custom/theme##custom custom/wallchange##custom custom/power )
```

{% asset_img waybar.png 分区示例 %}

#### waybar style.css设置

```css
#hardware {
      color: @main-fg;
      background: @main-bg;
      opacity: 1;
        margin: 3px 0px 3px 0px;
        padding-left: 0px;
        padding-right: 0px;
}

#clock,
#pulseaudio {
      color: @main-fg;
      background: @main-bg;
      opacity: 1;
        margin: 3px 0px 3px 0px;
        padding-left: 0px;
        padding-right: 4px;
}

```

#### waybar module修改

微调了几个waybar的module:

##### custom/clock##custom

简单更改了时间显示的格式：

```clock#custom.jsonc
    "clock": {
          "format": "{:%I:%M %p}",
          "rotate": "${r_deg}",
          "format-alt": "{:%R 󰃭 %Y-%m-%d}",
          "tooltip-format": "<span>{calendar}</span>",
          "calendar": {
                "mode": "month",
            "mode-mon-col": 3,
            "on-scroll": 1,
            "on-click-right": "mode",
            "format": {
                    "months": "<span color='#ffead3'><b>{}</b></span>",
              "weekdays": "<span color='#ffcc66'><b>{}</b></span>",
              "today": "<span color='#ff6699'><b>{}</b></span>"
                  }
                },
          "actions": {
                "on-click-right": "mode",
            "on-click-forward": "tz_up",
            "on-click-backward": "tz_down",
            "on-scroll-up": "shift_up",
            "on-scroll-down": "shift_down"
              }
            },
```

##### custom/theme

给切换主题的命令加上了自动切换nvim主题的脚本，见[脚本](#HyDE切换主题时自动切换neovim的主题)

```jsonc
    "custom/theme": {
          "format": "{}",
          "rotate": "${r_deg}",
          "exec": "echo ; echo 󰟡 switch theme",
          "on-click": "themeswitch.sh -n | python3 $HOME/.config/hypr/scripts/change_nvim_colorscheme.py",
          "on-click-right": "themeswitch.sh -p | python3 $HOME/.config/hypr/scripts/change_nvim_colorscheme.py",
          "on-click-middle": "sleep 0.1 && themeselect.sh | python3 $HOME/.config/hypr/scripts/change_nvim_colorscheme.py",
          "interval": 86400, // once every day
          "tooltip": true
          },
```

##### custom/wallchange

`custom/wallchange`单击切换下一张壁纸，右键单击切换上一张壁纸，中键单击出现壁纸选择页面。

因为我用的是[mpvpaper](#动态壁纸)动态壁纸，所以把命令改成了对应的：下一个视频、上一个视频、暂停：

```jsonc
    "custom/wallchange": {
          "format": "{}",
          "rotate": "${r_deg}",
          "exec": "echo ; echo 󰆊 switch wallpaper",
          // "on-click": "swwwallpaper.sh -n",
          // "on-click-right": "swwwallpaper.sh -p",
          // "on-click-middle": "sleep 0.1 && swwwallselect.sh",
          "on-click": "echo 'playlist-next' | socat - /tmp/mpv-socket",
          "on-click-right": "echo 'playlist-prev' | socat - /tmp/mpv-socket",
          "on-click-middle": "echo 'cycle pause' | socat - /tmp/mpv-socket ",
          "interval": 86400, // once every day
          "tooltip": true
          },
```

##### group/hardware

显示cpu、内存、CPU温度、GPU温度。

将`cpu`、`memory`、`custom/cpuinfo`、`custom/gpuinfo`合在一起堆叠，光标移过去会展开所有组件。

```jsonc
    "group/hardware": {
      "orientation": "inherit",
      "drawer": {
        "transition-duration": 500
      },
      "modules": [
        "cpu",
        "memory",
        "custom/cpuinfo",
        "custom/gpuinfo"
      ]
    },
    "cpu": {
      "interval": 10,
      "format": "󰍛 {usage}%",
      "rotate": "${r_deg}",
      "format-alt": "{icon0}{icon1}{icon2}{icon3}",
      "format-icons": [
        "▁",
        "▂",
        "▃",
        "▄",
        "▅",
        "▆",
        "▇",
        "█"
      ]
    },
    "memory": {
      "states": {
        "c": 90, // critical
        "h": 60, // high
        "m": 30, // medium
      },
      "interval": 30,
      "format": "󰾆 {used}GB",
      "rotate": "${r_deg}",
      "format-m": "󰾅 {used}GB",
      "format-h": "󰓅 {used}GB",
      "format-c": " {used}GB",
      "format-alt": "󰾆 {percentage}%",
      "max-length": 10,
      "tooltip": true,
      "tooltip-format": "󰾆 {percentage}%\n {used:0.1f}GB/{total:0.1f}GB"
    },
    "custom/cpuinfo": {
      "exec": " cpuinfo.sh",
      "return-type": "json",
      "format": "{}",
      "rotate": "${r_deg}",
      "restart-interval": 5, // once every 5 seconds
      "tooltip": true,
      "max-length": 1000
    },
    "custom/gpuinfo": {
      "exec": " gpuinfo.sh",
      "return-type": "json",
      "format": "{}",
      "rotate": "${r_deg}",
      "interval": 5, // once every 5 seconds
      "tooltip": true,
      "max-length": 1000,
      "on-click": "gpuinfo.sh --toggle",
    },
    "custom/gpuinfo#nvidia": {
      "exec": " gpuinfo.sh --use nvidia ",
      "return-type": "json",
      "format": "{}",
      "rotate": "${r_deg}",
      "interval": 5, // once every 5 seconds
      "tooltip": true,
      "max-length": 1000,
    },
    "custom/gpuinfo#amd": {
      "exec": " gpuinfo.sh --use amd ",
      "return-type": "json",
      "format": "{}",
      "rotate": "${r_deg}",
      "interval": 5, // once every 5 seconds
      "tooltip": true,
      "max-length": 1000,
    },
    "custom/gpuinfo#intel": {
      "exec": " gpuinfo.sh --use intel ",
      "return-type": "json",
      "format": "{}",
      "rotate": "${r_deg}",
      "interval": 5, // once every 5 seconds
      "tooltip": true,
      "max-length": 1000,
    },
```

##### cava module设置

该模组在waybar上可视化显示音乐频率

需要先安装`cava`:

```bash
sudo pacman -S cava
```

更改waybar上cava的外观:

将下列代码添加至`~/.config/hyde/config.toml`中

需要在`~/.config/waybar/config.ctl`中启用`custom/cava`

```conf
[cava.waybar]
# 'cava.sh waybar' configuration
max_instances = 1 # Maximum number of cava instances // fix and workaround when waybar won't kill 'cava ' process
# bar = "▁▂▃▄▅▆▇█"
bar = "🌑🌒🌓🌔🌕🌖🌗🌘"
width = 8
range = 7
# standby = "🎶"
standby = "🌑🌒🌓🌔🌕🌖🌗🌘"
```

类型包括：

```
# waybar_cava_bar="▁▂▃▄▅▆▇█"
# waybar_cava_bar="▏▎▍▌▋▊▉█"
# waybar_cava_bar="░▒▓█"
# waybar_cava_bar="▖▗▘▙▚▛▜▝▞▟"
# waybar_cava_bar="▂▃▄▅▆▇█"
#waybar_cava_bar="▕▏▎▍▌▋▊▉"
# waybar_cava_bar="⣀⣄⣤⣦⣶⣷⣿"
# waybar_cava_bar="⠁⠂⠄⡀⢀⠠⠐⠈"
# waybar_cava_bar="⠋⠙⠹⢸⣰⣤⣦⣶"
# waybar_cava_bar="⠁⠃⠇⡇⣇⣧⣷⣿"
waybar_cava_bar="🌑🌒🌓🌔🌕🌖🌗🌘"
# waybar_cava_bar="🌑🌘🌗🌖🌔🌓🌒🌑"
#waybar_cava_bar="🌕🌖🌗🌘🌒🌓🌔🌕"
# waybar_cava_bar="★☆★☆★☆★☆"
# waybar_cava_bar="⣾⣽⣻⢿⡿⣟⣯⣷"
# waybar_cava_bar="ᗧᗣᗤᗥᗦᗧᗣᗤᗥᗦ"
```

##### custom/lyrics

需配合[洛雪音乐](https://github.com/lyswhut/lx-music-desktop)一起使用，目前支持落雪音乐、yesplaymusic和Spotify。

开启洛雪音乐的[API](https://lxmusic.toside.cn/desktop/open-api)后，播放音乐时显示歌词在waybar上，需安装curl命令。

需要安装python.

还需要安装[sptlrx](https://github.com/raitonoberu/sptlrx)，用于获取Spotify歌词，需要配置Spotify的Cookie。

```fish
yay -S curl sptlrx-bin
```

{% asset_img lx_lyrics.png 分区示例 %}

代码非常简单:

见：<https://gist.github.com/coinhere/5c32ddf615574a5565ac83301c42ec64>

##### hyprland/workspaces

Toggle Special Workspaces时显示图标

```jsonc
    "hyprland/workspaces": {
          "disable-scroll": true,
          "rotate": "${r_deg}",
          "all-outputs": true,
          "active-only": false,
          "show-special": true,
          "special-visible-only": true,
          "on-click": "activate",
          "disable-scroll": false,
          "on-scroll-up": "hyprctl dispatch workspace -1",
          "on-scroll-down": "hyprctl dispatch workspace +1",
          "persistent-workspaces": {},
          "format": "{icon}",
          "format-icons": {
                "special": " ",
          }
            },
```

##### wlr/taskbar

显示所有窗口的图标，点击会切换该窗口的工作区并聚焦其上

将两个scratchpad加入了忽略名单中，避免干扰正常窗口：

```jsonc
  "wlr/taskbar": {
    "all-outputs": true,
    "markup": true,
    "format": "{icon}",
    "rotate": "${r_deg}",
    "icon-size": "${i_task}",
    "icon-theme": "${i_theme}",
    "spacing": 0,
    "tooltip-format": "{title}{app_id}",
    "on-click": "activate",
    "on-click-right": "maximize",
    "on-click-middle": "close",
    "ignore-list": [
      "kitty-dropterm",
      "kitty-btop",
      "org.pulseaudio.pavucontrol",
      "qalculate-gtk"
    ],
    "app_ids-mapping": {
      "firefoxdeveloperedition": "firefox-developer-edition",
      "firefoxnightly": "firefox-nightly",
      "Spotify Free": "Spotify"
    },
    "rewrite": {
      "Firefox Web Browser": "Firefox",
      "Foot Server": "Terminal",
      "Spotify Free": "Spotify",
      "org.kde.dolphin": "dolphin",
      "libreoffice-writer": "writer"
    }
  },
```

注意需要正确设置这两个应用的`title`，见[dropterm](#hyprland-drop-down-terminal), [dropbtop](#设置下拉式btop窗口，方便随时查看)

##### mpris

显示并可控制当前播放的音频

简单去掉了一些信息，因为太长了:

```jsonc
    "mpris": {
      "format": "{player_icon} {dynamic}",
      "rotate": "${r_deg}",
      "format-paused": "{status_icon} <i>{dynamic}</i>",
      "player-icons": {
        "default": "",
        "mpv": " ",
        "spotify": " ",
        "chromium": "󰎇"
      },
      "status-icons": {
        "paused": "▶"
      },
      "ignored-players": [
        "firefox",
        "chrome"
      ],
      "max-length": 50,
      "interval": 1,
      "dynamic-order": [
        "title",
        // "artist",
        // "album",
        "position",
        "length"
      ],
      "dynamic-importance-order": [
        "position",
        "length",
        "title",
        "artist",
        "album"
      ],
      "tooltip-format": "{title} - {album}\nartist {artist}({player})"
    },
```

### SDDM theme

#### SDDM主题字体过小

在`/usr/share/sddm/themes/<theme-name>/theme.conf`修改：

```conf
GeneralFontSize="18"
```

#### 修改SDDM背景图片

在`/usr/share/sddm/themes/Candy/backgrounds/`里添加自己想要的背景，再在`/usr/share/sddm/themes/Candy/theme.conf`里修改。

#### 设置sddm触摸板轻触为点击

ref: <https://gitlab.com/Matt.Jolly/sddm-eucalyptus-drop/>

sddm默认运行在Xorg上，因此要修改X11的设置：

`cat /etc/X11/xorg.conf.d/20-touchpad.conf`

```
Section "InputClass"
        Identifier "libinput touchpad catchall"
        MatchIsTouchpad "on"
        MatchDevicePath "/dev/input/event*"
        Driver "libinput"

        Option "Tapping" "on"
        Option "NaturalScrolling" "on"
        Option "MiddleEmulation" "on"
        Option "DisableWhileTyping" "on"
EndSection

```

### 动态壁纸

这里使用[mpvpaper](https://github.com/GhostNaN/mpvpaper)，可以将视频作为桌面，并支持mpv的设置。

#### 安装

```bash
sudo pacman -S mpvpaper
```

#### 设置自动启动

在`~/.config/hypr/userprefs.conf`中添加：

```conf
exec-once = mpvpaper -f -n 7200 -o "input-ipc-server=/tmp/mpv-socket --shuffle --loop --loop-playlist --panscan=1.0 --osd-level=0" "*" /home/Videos/Wallpapers
```

参数解释:

mpvpaper详细参数意义见`man mpvpaper`，mpv详细设置见<https://mpv.io/manual/master>

- `-f` -- fork mpvpaper从而可以关闭终端
- `-n 7200` -- 幻灯片模式每2小时（7200秒）播放播放列表中的下一个视频，需配合`--loop`, `--loop-playlist`使用
- `-o` -- mpvpaper传递参数给mpv
  - `input-ipc-server=/tmp/mpv-socket` -- 提供mpvpaper的控制接口
  - `shuffle` -- 启动时打乱播放列表
  - `loop` -- 循环播放视频
  - `loop-playlist` -- 循环播放列表
  - `panscan=1.0` -- 拉伸以填充整个屏幕（不留黑边）
  - `osd-level=0` -- 去除所有mpv渲染在视频上的OSD信息，避免禁音mpv时显示"Mute: yes"
- `"*"` -- 显示在所有屏幕上
- `/home/Video/Wallpapers` -- # 播放的可以是视频文件或者包含视频文件的文件夹

参数过多，可以设置mpv配置文件，再引用配置文件`~/.config/mpv/mpv.conf`，详细见<https://mpv.io/manual/master/#configuration-files>：

```conf
[mpvpaper]
profile-desc="profile for mpvpaper"
vo=gpu-next
gpu-api=auto
hwdec=auto-safe
profile=fast
input-ipc-server=/tmp/mpv-socket
shuffle
loop
loop-playlist
panscan=1.0
osd-level=0
```

之后自动启动便可写为：

```conf
exec-once = mpvpaper -f -n 7200 -o "profile=mpvpaper" "*" /home/Videos/Wallpapers
```

#### 设置mpvpaper桌面图层排序

此外，为避免视频桌面被其他桌面程序覆盖(HyDE中是swww-daemon)，可以设置mpvpaper的layer优先度最低，或者直接不启动其他桌面程序：

```conf
layerrule = order -1, mpvpaper
```

#### 自动暂停、禁音

mpvpaper提供`--auto-pause`和`--auto-stop`的参数，但在hyprland中没有效果，因此自己写了两个脚本：

- [auto_pause_mute_mpvpaper.sh](https://gist.github.com/coinhere/b97695322f9079a2178bb55120f2a795)，作用是：
  - 工作区窗口数量由0变为1时静音，由1变为0时取消静音
  - 窗口全屏时（非最大化）暂停，退出全屏时取消暂停
  - 切换工作区后，根据工作区窗口数量和是否全屏选择禁音、暂停mpvpaper与否
- [mpvpaper.sh](https://gist.github.com/coinhere/b97695322f9079a2178bb55120f2a795#file-mpvpaper-sh)，作用是：
  - 如果以及有mpvpaper.sh进程则直接退出
  - 启动mpvpaper和auto_pause_mute_mpvpaper.sh
  - 退出时关闭mpvpaper和auto_pause_mute_mpvpaper.sh
  - 每隔一秒检测系统是否有其他音频输出，有则将mpvpaper音量降至零

需要开启mpvpaper控制接口，并安装`socat`

自动启动修改为：

```conf
exec-once = $HOME/.config/hypr/scripts/mpvpaper.sh
```

#### 添加控制壁纸快捷键

在`~/.config/hypr/keybindings.conf`：

需要开启mpvpaper控制接口，并安装`socat`

```conf
# mpv-paper
# 切换mpvpaper运行状态，关闭/启动mpvpaper及相关进程
bindd = $mainMod, F4, $d toggle mpvpaper, exec, pkill -x mpvpaper.sh >/dev/null || ~/.config/hypr/scripts/mpvpaper.sh
bindd = $mainMod, F5, $d toggle mpvpaper voice, exec, echo 'cycle mute' | socat - /tmp/mpv-socket # 静音/取消静音
bindd = $mainMod, F6, $d mpvpaper play prev, exec, echo 'playlist-prev' | socat - /tmp/mpv-socket # 播放上一个
bindd = $mainMod, F7, $d toggle mpvpaper play, exec, echo 'cycle pause' | socat - /tmp/mpv-socket # 暂停/取消暂停
bindd = $mainMod, F8, $d mpvpaper play next, exec, echo 'playlist-next' | socat - /tmp/mpv-socket # 播放下一个
```

### hyprlock锁屏

HyDE现在默认使用hyprlock锁屏

#### hyprlock配置

设置见`~/.config/hypr/hyprlock.conf`，模板见`~/.config/hypr/hyprlock/`

这里对`hyprlock/Anurati.conf`略作修改:

```custom.conf
#      ░▒▒▒░░░░░▓▓          ___________
#    ░░▒▒▒░░░░░▓▓        //___________/
#   ░░▒▒▒░░░░░▓▓     _   _ _    _ _____
#   ░░▒▒░░░░░▓▓▓▓▓▓ | | | | |  | |  __/
#    ░▒▒░░░░▓▓   ▓▓ | |_| | |_/ /| |___
#     ░▒▒░░▓▓   ▓▓   \__  |____/ |____/
#       ░▒▓▓   ▓▓  //____/

$fontFamily = JetBrainsMono Nerd Font # We already have this font installed

# Resolving custom fonts
# Provide the font name and the download link separated by a pipe |
# Run font.sh resolve $LAYOUT_PATH to install the font
$resolve.font=Anurati|https://font.download/dl/font/anurati.zip
$resolve.font=Inter|https://github.com/rsms/inter/releases/download/v4.1/Inter-4.1.zip

background {
    monitor =
    color = $wallbash_pry1_rgba
    path = $BACKGROUND_PATH
    blur_size = 4
    blur_passes = 3 # 0 disables blurring
    noise = 0.0117
    contrast = 1.3000 # Vibrant!!!
    brightness = 0.8000
    vibrancy = 0.2100
    vibrancy_darkness = 0.0
}

# DAY
label {
    monitor =
    text = cmd[update:1000] echo "$(date +"%A" | sed 's/./& /g' | tr '[:lower:]' '[:upper:]')" # Add a thin space between each character
    color = $wallbash_1xa9_rgba
    font_size = 110 # Wednesday is too long
    font_family = Anurati
    position = 0, 220
    halign = center
    valign = center
}

# DATE
label {
    monitor =
    text = cmd[update:1000] echo "$(date +"%B %d")"
    color = $wallbash_2xa9_rgba
    font_size = 35
    font_family = $font_family Thin
    position = 0, 100
    halign = center
    valign = center
}

# TIME
label {
    monitor =
   # text = cmd[update:1000] echo "$(date +"%-I:%M %p")"
   text = cmd[update:1000] echo "$(date +"- %H:%M -")"
    #text = $TIME
    color = $wallbash_3xa9_rgba
    font_size = 20 #!
    font_family = $font_family
    position = 0, 30
    halign = center
    valign = center
}

# INPUT FIELD
input-field {
    monitor =
    size = 200, 50 #!
    outline_thickness = 3
    dots_size = 0.33 # Scale of input-field height, 0.2 - 0.8
    dots_spacing = 0.15 # Scale of dots' absolute size, 0.0 - 1.0
    dots_center = true
    dots_rounding = -1 # -1 default circle, -2 follow input-field rounding
    outer_color = rgba(0,0,0,0)
    inner_color = rgba(255,255,255,0.1)
    # outer_color = $wallbash_pry4_rgba
    # inner_color = $wallbash_pry2_rgba
    font_color = $wallbash_3xa9_rgba
    fade_on_empty = false
    # fade_timeout = 1000 # Milliseconds before fade_on_empty is triggered.
    placeholder_text = <i>Input Password...</i> # Text rendered in the input box when it's empty.
    hide_input = false
    rounding = -1 # -1 means complete rounding (circle/oval)
    check_color = $wallbash_pry4_rgba
    fail_color = rgba(FF0000FF) # if authentication failed, changes outer_color and fail message color
    fail_text = <i>$FAIL <b>($ATTEMPTS)</b></i> # can be set to empty
    fail_transition = 300 # transition time in ms between normal outer_color and fail_color
    capslock_color = -1
    numlock_color = -1
    bothlock_color = -1 # when both locks are active. -1 means don't change outer color (same for above)
    invert_numlock = false # change color if numlock is off
    swap_font_color = true # see below
    position = 0, -150
    halign = center
    valign = center
}

# User tag
label {
    monitor =
    text =    $USER
    color = $wallbash_4xa9_rgba
    outline_thickness = 2
    dots_size = 0.2 # Scale of input-field height, 0.2 - 0.8
    dots_spacing = 0.2 # Scale of dots' absolute size, 0.0 - 1.0
    dots_center = true
    font_size = 20
    font_family = Inter Display Medium
    position = 0, -80
    halign = center
    valign = center
}

# MPRIS text
label {
    monitor =
    text = cmd[update:1000] $MPRIS_TEXT
    # color = $wallbash_4xa9_rgba
    color = $wallbash_txt1_rgba
    font_size = 20
    font_family = Inter Display Medium
    position = 0, 30
    halign = center
    valign = bottom
}
```

#### 自动退出、启动动态壁纸

hyprlock锁屏时，动态壁纸还会运行，因此写了个脚本锁屏时退出mpvpaper，解锁时再打开mpvpaper，这里使用上面脚本启动mpvpaper：

```bash
#!/bin/bash

lock() {
  # avoid starting multiple hyprlock instances.
  if ! pidof hyprlock >/dev/null; then
    # quit mpvpaper script
    pkill -x mpvpaper.sh
    # pause music
    # playerctl pause
    # when unlock, restart mpvpaper
    lockscreen.sh && lock_hook
  fi
}

lock_hook() {
  # run mpvpaper again
  # playerctl play
  ~/.config/hypr/scripts/mpvpaper.sh
}

lock
```

#### hypridle

hypridle是Hyprland的一个空闲daemon。空闲时自动计时，到达设定的触发器的时间便会执行触发器的命令。

对默认的配置修改了触发时间和锁屏命令

```conf
#
# $LOCKSCREEN = hyprlock # preferred LOCKSCREEN e.g swaylock
$LOCKSCREEN = $HOME/.config/hypr/scripts/lock.sh


general {
    lock_cmd = $LOCKSCREEN
    unlock_cmd = #notify-send "unlock!"      # same as above, but unlock
    before_sleep_cmd = $LOCKSCREEN    # command ran before sleep
    after_sleep_cmd = # notify-send "Awake!"  # command ran after sleep
    ignore_dbus_inhibit = 0
}

# Dims the display
listener {
    timeout = 150
    on-timeout = { brightnessctl -s && brightnessctl s 1% ;}
    on-resume = brightnessctl -r
}

# Lock it first before dpms off so that screen won't show for a moment after wake up.
listener {
    timeout = 300
    on-timeout = $LOCKSCREEN
}

# DPMS off
listener {
    timeout = 360
    on-timeout = Hyde run audio_idle ; hyprctl dispatch dpms off #do not turn off display while media is playing
    on-resume = hyprctl dispatch dpms on
}

# Suspend
listener {
    timeout = 900
    on-timeout = Hyde run audio_idle ; systemctl suspend
}

# hyprlang noerror true
# Source anything  from this path if you want to add your own listener
# source command actually do not exist yet
source = ~/.config/hypridle/*
# hyprlang noerror false
```
