# Ubuntu 中文输入法安装指南

本指南将帮助你在 Ubuntu 系统上安装和配置中文输入法。

## 目录

- [安装 iBus 框架](#安装-ibus-框架)
- [安装 Fcitx 框架](#安装-fcitx-框架)
- [常见问题与解决方案](#常见问题与解决方案)

## 安装 iBus 框架

iBus 是 Ubuntu 默认的输入法框架，安装配置相对简单。

### 1. 安装 iBus 拼音

```bash
sudo apt update
sudo apt install ibus ibus-pinyin
```

### 2. 配置 iBus

安装完成后，需要配置系统使用 iBus：

1. 打开"设置"（Settings）
2. 选择"区域和语言"（Region & Language）
3. 点击"管理已安装的语言"（Manage Installed Languages）
4. 在"键盘输入法系统"中选择"iBus"
5. 点击"应用系统范围"（Apply System-Wide）
6. 重启系统以使更改生效

### 3. 添加中文输入源

1. 打开"设置"（Settings）
2. 选择"区域和语言"（Region & Language）
3. 点击"输入源"下的"+"按钮
4. 选择"中文（Chinese）"，然后选择"智能拼音"（Intelligent Pinyin）
5. 点击"添加"（Add）

现在你可以使用 Super+空格键（或 Super+Shift+空格）切换不同的输入法。

## 安装 Fcitx 框架

Fcitx 是另一个流行的输入法框架，支持多种中文输入法。

### 1. 安装 Fcitx 和中文输入法

```bash
sudo apt update
sudo apt install fcitx fcitx-pinyin fcitx-googlepinyin fcitx-config-gtk
```

如果你想要使用搜狗拼音输入法：

```bash
# 添加i386架构支持（对于64位系统）
sudo dpkg --add-architecture i386
sudo apt update

# 下载并安装搜狗拼音
wget "https://ime.sogoucdn.com/dl/index/1571302197/sogoupinyin_2.3.1.0112_amd64.deb"
sudo dpkg -i sogoupinyin_2.3.1.0112_amd64.deb

# 解决可能的依赖问题
sudo apt -f install
```

### 2. 配置 Fcitx

1. 打开"设置"（Settings）
2. 选择"区域和语言"（Region & Language）
3. 点击"管理已安装的语言"（Manage Installed Languages）
4. 在"键盘输入法系统"中选择"fcitx"
5. 点击"应用系统范围"（Apply System-Wide）
6. 重启系统以使更改生效

### 3. 配置 Fcitx 输入法

1. 重启后，在桌面右上角托盘找到键盘图标
2. 右键点击并选择"配置"（Configure）
3. 点击"+"添加输入法
4. 取消勾选"只显示当前语言"，然后搜索"pinyin"或"sogou"
5. 选择你想要的输入法，点击"OK"添加

默认情况下，你可以使用 Ctrl+空格切换不同的输入法。

## 常见问题与解决方案

### 输入法图标不显示

```bash
sudo apt install gnome-shell-extension-appindicator
```

重启系统或注销后重新登录。

### 输入法无法切换

确保快捷键没有冲突：

1. 打开设置 > 键盘 > 键盘快捷键
2. 检查输入法切换快捷键是否与其他快捷键冲突

### Fcitx 设置不生效

运行以下命令，确保环境变量正确设置：

```bash
im-config -n fcitx
```

编辑 `~/.xprofile` 文件，添加：

```bash
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
```

### 搜狗拼音崩溃或不稳定

尝试清除缓存：

```bash
rm -rf ~/.config/SogouPY
rm -rf ~/.config/sogou*
```

然后重启 Fcitx：

```bash
fcitx -r
```

## 推荐配置

对于大多数用户，推荐使用 Fcitx + 搜狗拼音或 Fcitx + Google 拼音的组合，这些输入法词库丰富，使用体验较好。

如果追求稳定性和系统集成，可以选择默认的 iBus + 智能拼音组合。 