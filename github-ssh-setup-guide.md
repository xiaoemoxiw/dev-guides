# GitHub SSH 连接设置指南

在国内网络环境中，使用 HTTPS 协议连接 GitHub 可能会遇到各种连接问题。这篇指南将帮助你设置 SSH 密钥并配置 Git 使用 SSH 协议连接 GitHub，从而获得更稳定的连接体验。

## 问题背景

使用 HTTPS 协议克隆或推送 GitHub 仓库时，可能会遇到以下错误：

```
fatal: unable to access 'https://github.com/user/repo.git/': Failed to connect to github.com port 443 after 75007 ms: Couldn't connect to server
```

这通常是由于网络连接问题、防火墙限制或者 GitHub 服务在特定区域访问受限导致的。

## 使用 SSH 代替 HTTPS 的优势

- 更稳定的连接，特别是在网络环境复杂的地区
- 无需每次操作都输入用户名密码
- 更安全的身份验证方式
- 避开 HTTPS 的一些网络限制

## 步骤 1：生成新的 SSH 密钥对

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱地址"
```

执行此命令后，会提示你设置保存位置和密码。通常可以使用默认位置（按 Enter），密码可以设置也可以留空（安全起见建议设置）。

## 步骤 2：启动 SSH 代理并添加密钥

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

## 步骤 3：复制 SSH 公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

复制输出的内容，它应该类似于：
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKJipgXMnFvyM6PRfJbvbSW13DkUr5EXsgKaC99gm3Ks your-email@example.com
```

## 步骤 4：添加 SSH 公钥到 GitHub

1. 登录你的 GitHub 账户
2. 点击右上角头像，选择 "Settings"（设置）
3. 在左侧菜单中，点击 "SSH and GPG keys"（SSH 和 GPG 密钥）
4. 点击 "New SSH key"（新建 SSH 密钥）按钮
5. 在 "Title"（标题）字段中，输入一个描述性名称（例如 "MacBook Pro 2024"）
6. 在 "Key"（密钥）字段中，粘贴之前复制的公钥
7. 点击 "Add SSH key"（添加 SSH 密钥）按钮

## 步骤 5：测试 SSH 连接

```bash
ssh -T git@github.com
```

如果成功，你会看到类似以下的消息：
```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

## 步骤 6：将仓库远程 URL 更改为 SSH

对于现有仓库，需要将远程 URL 从 HTTPS 更改为 SSH：

```bash
git remote set-url origin git@github.com:username/repository.git
```

请将 `username` 和 `repository` 替换为你的 GitHub 用户名和仓库名。

## 步骤 7：验证远程 URL 已更改

```bash
git remote -v
```

应该显示类似于：
```
origin  git@github.com:username/repository.git (fetch)
origin  git@github.com:username/repository.git (push)
```

## 步骤 8：使用 SSH 推送更改

```bash
git push
```

现在你应该能够成功推送代码到 GitHub 了。

## 故障排查

如果在使用 SSH 连接 GitHub 时遇到问题，可以尝试以下解决方法：

### 1. 确认 SSH 密钥已添加到 SSH 代理

```bash
ssh-add -l
```

### 2. 检查 GitHub SSH 密钥设置

确保你的公钥已正确添加到 GitHub 账户。

### 3. 测试 SSH 连接详细信息

```bash
ssh -vT git@github.com
```

这会显示详细的连接调试信息，帮助你找出问题所在。

### 4. 检查 SSH 配置

你可以创建或编辑 `~/.ssh/config` 文件，添加以下配置以优化 GitHub 连接：

```
Host github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  PreferredAuthentications publickey
  IdentitiesOnly yes
```

## 结论

使用 SSH 连接 GitHub 通常是解决网络连接问题的有效方法，特别是在网络环境复杂的区域。通过本指南设置完成后，你应该能够更稳定地进行代码推送和拉取操作。 