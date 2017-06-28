SSH是每台Linux电脑的标准配置。
随着Linux设备从电脑逐渐推广到手机、外设和智能家电，SSH的使用场景越来越广泛。

## 一、什么是SSH?
简言之，SSH是一种网络协议，用于计算机之间的加密登录。

如果用户从本地计算机，通过SSH协议，登录远程计算机。这种登录行为被认为是安全的，即使被中途截获，密码也不会泄露。

早期，互联网明文通信，一旦被截获，内容就暴露无疑。1995年，芬兰学者Tatu Ylonen设计了SSH协议，将登录信息全部加密，成为互联网安全的一种基本解决方案，迅速在全世界获得推广，目前已经成为Linux系统的标准配置。

SSH只是一种协议，既有商业实现，也有开源实现。这里只讨论[OpenSSH](http://www.openssh.com)

## 二、最基本的用法
SSH主要用于远程登录。假定你以用户名user，登录远程主机host，只要一条简单命令就可以了。
```shell
$ ssh user@host
```
如果本地用户名和远程用户名一致，登录时可以省略用户名
```shell
$ ssh host
```
SSH默认的端口号是22，使用参数p,可以修改端口号
```
$ ssh -p 2222 user@host
```
## 三、中间人攻击
SSH之所以安全，主要是因为它采用了公钥加密。整个过程如下
1. **用户**发送登录请求，**远程主机**把自己的公钥`/etc/ssh/ssh_host_rsa_key.pub`发给用户
2. **用户**使用这个公钥加密登录密码，发送给**远程主机**
3. **远程主机**用自己的私钥`/etc/ssh/ssh_host_rsa_key`解密登录密码，如果密码正确，就同意用户登录
在实际应用中（比如在公共的wifi区域），黑客会截获登录请求，冒充远程主机，将伪造的公钥发送给用户，用户很难辨别真伪，黑客获取用户的密码后登录远程主机，这就是著名的[中间人攻击](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)。SSH协议的公钥是自己签发的，不像https协议有证书中心（CA）公证。
SSH协议如何应对呢？

## 四、口令登录
如果首次登录对方主机，系统会出现以下提示：
```
$ ssh user@host
The authenticity of host 'host (12.18.429.21)' can't be established.
RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
Are you sure you want to continue connecting (yes/no)?
```
这段话的意思是，无法确认主机的真实性，只知道它的公钥指纹，问你还想继续连接吗？
所谓“公钥指纹”，是指公钥长度较长（这里采用RSA算法，长达1024位），很难对比，所以对其进行MD5计算，将它变成一个128位的指纹。
用户怎么知道远程主机的公钥指纹应该是多少？回答是没有好办法，远程主机必须在自己的网站上贴出公钥指纹，以便用户自己核对。
经过风险衡量后，用户输入`yes`接受远程主机的公钥。
系统会出现一句提示，表示host主机已经得到认可。
```
Warning: Permanently added 'host,12.18.429.21' (RSA) to the list of known hosts.
```
然后，会要求输入密码。
```
Password:(enter password)
```
如果密码正确，就可以登录了。
当远程主机的公钥被接受以后，它就会被保存在用户的文件`$HOME/.ssh/known_hosts`之中。下次再连接远程主机，本地主机会认出它的公钥已经保存在`$HOME/.ssh/known_hosts`，从而跳过警告部分，直接提示输入密码。
此外，系统也有一个这样的文件`/etc/ssh/ssh_known_hosts`，保存对所有用户都可依赖的远程主机的公钥。

## 五、公钥登录
SSH还提供了公钥登录，可以省去输入密码的步骤。那如何设置公钥登录呢？
1. 首先用户必须提供自己的公钥。用户运行下面的命令可以产生自己的公钥
```
$ ssh-keygen
```
运行上面的命令以后，系统会出现一系列提示，可以一路回车。其中有一个问题是，要不要对私钥设置口令(passphrase)，如果担心私钥的安全，这里可以设置一个。
运行结束以后，在`$HOME/.ssh/`目录下面，会生成两个文件:`id_rsa.pub`和`id_rsa`，前者是你的公钥，后者是你的私钥。
2. 将公钥传添加到远程主机`$HOME/.ssh/authorized_keys`上面
```
$ ssh-copy-id user@host
```
改用下面的命令可以达到`ssh-copy-id`同样的效果
```
$ ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```
通常完成这个步骤后，你再登录远程主机，就不需要输入密码了。
3. 如果还是不行，就打开远程主机的`/etc/ssh/sshd_config`这个文件，检查下面几行前面的#注释是否取掉
```
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```
然后，通过下面的命令重启远程主机的SSH服务：
```
// Linux/Ubuntu系统
$ /sbin/service ssh restart
// Debian系统
$ /etc/init.d/ssh restart
```
