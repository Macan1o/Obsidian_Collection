
---

> [!NOTE] 核心思想
> 基本思想就是，先在本电脑创建一个新的密钥，然后在账户的设置那完成密钥的认证。从而获得推送的权限

## 阶段 1：清理本地所有旧 ssh 文件

```
# 删除整个 .ssh 文件夹，全部旧密钥、旧config一次性清空
Remove-Item $env:USERPROFILE\.ssh -Recurse -Force -ErrorAction SilentlyContinue
```

## 阶段 2：网页端清理

清理掉设置里面的旧SSH key


## 阶段 3：全新生成一套密钥

```
ssh-keygen -t ed25519 -C "1781130988@qq.com"
```


## 阶段 4：复制公钥，添加到GitHub

```
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
```

### 浏览器添加 New SSH key

- Title：`Windows`
	
- Key type：`Authentication key`
	
- 直接粘贴公钥
	
- Add SSH key。


## 阶段 5：测试 SSH 连通

```
ssh -T git@github.com
```

