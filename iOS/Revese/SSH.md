# SSH OpenSSH SSL OpenSSL

## SSL

Secure Sockets Layer 的缩写，是未网络通讯提供安全及数据完整性的一种安全协议，在传输层对网络连接进行加密

## OpenSSL

SSL的开源实现

绝大部分HTTPS请求等价于：HTTP + OpenSSL

## SSH 

Secure Shell的缩写，意为"安全外壳协议"，一种可以为远程登录提供安全保障的协议

使用SSH，可以吧所有传输的数据进行加密，"中间人"攻击方式就不可能实现，能防止DNS欺骗和IP欺骗

## OpenSSH

是SSH协议的免费开源实现


***

OpenSSH的加密就是通过OpenSSL完成

# SSH的通信过程

## SSH通信主要分为3大主要方式

- 建立连接
	
	服务武器会提供自己的身份证明(公钥，私钥)，发送公钥等信息给客户端

	客户端把公钥相关存储在`~/.ssh/known_hosts`

- 客户端认证

	基于密码的客户端认证（使用账号密码即可认证）

	基于秘钥的客户端认证（免密码/最安全的一种认证方式）

	SSH-2默认会优先使用秘钥认证，然后再使用账号密码认证

- 数据传输

### 服务器身份信息变更

删除`~/.ssh/known_hosts` 下对应公钥信息

or

```ssh-keygen -R ipaddress```

### 客户端认证免密认证

在客户端生成一对公钥秘钥，然后将公钥内容追加到服务端的授权文件尾部

生成方法

```
ssh-keygen -t ras
```

生成后的地址在 `~/.ssh 下`

追加到服务器的授权文件尾部

```
ssh-copy-id user@ipaddress
```

授权文件地址`~/.ssh/authorized_keys`

### 远程拷贝

```scp ~/.ssh/id_rsa.pub root@ipaddress``` secure copy

# USB进行SSH登录

默认情况下，由于SSH走的是TCP协议，Mac是通过网络连接的方式SSH登录到iPhone，要求iPhone连接Wifi

为了加快传输速度，也可以通过USB连接的方式进行SSH登录

Mac上有个服务程序usbmuxd（开机启动），可以将Mac的数据通过USB传输到iPhone

```/System/Library/PrivateFrameworks/MobileDevice.framework/Resources/usbmuxd```

SSH登录到本地的一个端口，usbmuxd将本地的端口映射到iPhone的22端口
