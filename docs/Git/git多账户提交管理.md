在一台设备上管理多个 Git 账户（例如 GitHub/GitLab 等），需要解决 SSH 密钥匹配 和 用户身份区分 两个核心问题。以下是详细步骤：

## 为每个账户生成独立的 SSH 密钥
1. 生成第一个账户的密钥（如个人账户）：

```bash
ssh-keygen -t ed25519 -C "personal@email.com"
```

保存路径改为：`~/.ssh/id_ed25519_personal`（避免覆盖默认密钥）

2. 生成第二个账户的密钥（如工作账户）：

```bash
ssh-keygen -t ed25519 -C "work@email.com"
```
保存路径改为：`~/.ssh/id_ed25519_work`

> ed25519 是一种现代、高效的公钥加密算法名称，特别用于 SSH 密钥的生成和安全通信。

## 配置 SSH 识别不同密钥
在 `~/.ssh/config` 中配置规则（没有则创建）,目的是通过 Host 别名区分不同账户的仓库地址。：
```
# 个人账户
Host github-personal  # 自定义别名
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_ed25519_personal

# 工作账户
Host github-work      # 另一个别名
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_ed25519_work
```
注意config中的密钥为公钥，配置好后，可以在不同的git平台添加密钥

## 测试连接
```
ssh -T git@github.com-work
# 应返回：Hi workaccount! You've successfully authenticated...

ssh -T git@github.com-personal
# 应返回：Hi personalaccount! ...
```

## 克隆与推送时的操作规范
```
# 克隆工作账户的仓库
git clone git@github.com-work:company/project.git

# 克隆个人账户的仓库
git clone git@github.com-personal:myusername/myrepo.git

```

