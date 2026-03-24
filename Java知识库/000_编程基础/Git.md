## 问题排查
### closed by xx port 22

```bash
ssh -T git@github.com

# Connection closed by 198.18.0.20 port 22
```

大概原因：本质不是 **SSH key 配置问题**，而是 **网络层把你到 GitHub 的 22 端口连接拦掉了**。估计是 VPN 的原因。

解决方案： 
先测试 443 端口是否可以用：

```BASH
ssh -T -p 443 git@ssh.github.com
```
测试通过后**改用 443 端口 SSH**
```BASH
vim ~/.ssh/config
```

```CONFIG
Host github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_rsa
```
