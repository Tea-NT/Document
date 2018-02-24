# Git配置和Gitolite-admin管理git用户

最开始安装通用的教程在CentOS云主机上搭建Git服务端，参考教程[Git 服务器搭建](http://www.runoob.com/git/git-server.html)。包括使用证书登录，
但是后来跟其他人同步开发的话，特别是增加了新的用户，总不可能每次都登录服务器把公钥考过去把。后面发现原来已经有管理工具Gitolite-admin。


可以像管理普通git项目一样来管理所有的git用户，并且对用户或者项目进行授权，权限管理也非常简单。

这里由于已经先搭建好了直接通过证书来同步代码的方式，安装好Gitolite-admin后就出现了各种坑，不是没有权限访问这个主要的gitolite-admin库就是添加新的用户没有权限，看不到代码库。删除了又重装，弄了很久才弄好，特此记录一遍。

首先，如果按照直接用证书登录方式，需要手动来设置git用户目录下的.ssh/authorized_keys文件，添加一个新用户以及管理用户的权限非常难以控制。

而gitolit-admin则是相当于解放出来，通过它来配置authorized_keys文件，我们要做的只需要
1、git clone 出gitolit-admin项目
2、添加新用户的的公钥进gitolite-admin项目中keydir文件夹
3、在项目中conf/gitolite.conf配置文件中设置新用户的权限，将新用户加入用户组或者单独赋予某个项目的权限。
4、提交修改并推送至git服务器

像这样完全不需要单独操作服务器。轻轻松松添加新用户。

如果之前已经手动将公钥写入到authorized_keys文件中，再配置gitolite就会出问题，提示之前的公钥已经作为登录ssh的验证，将会被忽略。
> WARNING: keydir/git_key.pub duplicates a non-gitolite key, sshd will ignore it


英文提示大意是这样的，并且无法克隆出刚刚生成的gitolite-admin项目，提示没有权限访问。

这个时候，就需要将authorized_keys文件清空，删除gitolite重新安装，并且删除$HOME/.gitolite和\$HOME/.gitolite.rc文件，并重新生成指定gitolite使用的密钥对，使用新的密钥作为gitolite的管理员登录。详细见参考资料[git权限管理工具gitolite使用笔记(一)](http://www.cnblogs.com/seanvon/archive/2013/05/28/gitolite.html)



如果有多个git服务器，怎么指定连接某个服务器的时候用哪个Key呢？同理，如果提示没有权限克隆gitolite-admin项目也可能是因为git没有使用对应的管理员私钥钥。这里就用到了~/.ssh/config文件了，通过配置这个文件实现连接指定的服务器时使用指定的私钥。

```
host gitolite
	user git
	hostname 106.xx.xx.xx
	identityfile ~/.ssh/git_admin

```
- hostname 就是需要连接的服务器，可以使用网址或者ip地址
- identityfile 是针对这个服务器使用的验证密钥（私钥）

再测试，就能克隆gitolite项目了。
> ssh git@servername //用来测试git的权限，已经有权限访问的项目

```shell
Administrator@XXXX MINGW64 ~/.ssh
$ ssh git@gitolite
PTY allocation request failed on channel 0
hello git_admin, this is git@xxxxxx running gitolite3 v3.6.7-19-g2cfc81f on git 1.8.3.1

 R W    gitolite-admin
 R W    testing
Connection to 106.xx.xx.xx closed.

***************************************************************************
Administrator@XXXX MINGW64 ~/.ssh
$ ssh git@106.xx.xx.xx
PTY allocation request failed on channel 0
hello Blus, this is git@xxxxxx running gitolite3 v3.6.7-19-g2cfc81f on git 1.8.3.1

 R W    PowerOnline
 R W    gitolite-admin
 R W    testing
Connection to 106.xx.xx.xx closed.

```
上面的操作即实现针对同一个git服务器使用了不同的密钥访问，同时返回了不同的权限。

参考gitolite.conf文件
```
 repo gitolite-admin
	RW+     =   git_admin Blus

repo testing
	RW+     =   @all
repo PowerOnline
	RW+	=   Blus Sukxxxx
```

# 参考资料
1、[Pro Git](http://iissnan.com/progit/)

2、[gitolite官方指南](https://git-scm.com/book/zh/v1/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-Gitolite)

3、[Git 服务器搭建](http://www.runoob.com/git/git-server.html)


4、[git服务器新增用户](http://blog.csdn.net/augusdi/article/details/28099819)


5、[安装配置Git服务器Gitolite](http://williamherry.com/blog/2012/10/03/install-gitolite/)

6、[git权限管理插件gitolite](http://blog.csdn.net/stormragewang/article/details/43487749) 参考gitolite安装步骤

7、[git权限管理工具gitolite使用笔记](http://www.cnblogs.com/seanvon/archive/2013/06/07/3124226.html) 参考权限配置

8、[gitolite: can connect via ssh, can't clone](https://stackoverflow.com/questions/9339272/gitolite-can-connect-via-ssh-cant-clone/9340778#) 配置config文件 解决连接后没有权限克隆gitolite-admin项目的问题

9、[git clone git@myserver:gitolite-admin fails](https://stackoverflow.com/questions/12617672/git-clone-gitmyservergitolite-admin-fails#) 测试gitolite权限

