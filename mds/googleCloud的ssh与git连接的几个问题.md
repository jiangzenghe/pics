---
title: googleCloud的ssh与git连接的几个问题
date: 2019-06-20 11:43:48
tags:
---

最近再玩google自建服务器，打算建立一个git仓库，然后远程push代码到服务器，然后部署到nginx服务器上，做一个自有blog，但是在配置完nginx之后，再搭建git服务器等的时候，出现了一系列问题，之前曾经解决过一遍的，这次又费了很大周折，故在此记录一下。

<!--more-->

* 自建git服务器教程一般都会说新建一个git用户，添加ssh的密钥等，其实如果我不建立git用户，而是随便建立aabb用户，只要按ssh的规则连接，链接所创立的用户并且设置对git仓库对应的用户权限，完全可以不是建立git用户，关键点有两个，一个是连接用户要确定，再一个ssh密钥的设置正确
可以使用 ssh -T 用户名@ip来测试ssh的联通性，也可以使用ssh 用户名@ip -v来查看链接过程中的信息，正确链接与否，无法链接的细节报错等等
git在保持客户端和服务端的链接其实本质都是用ssh做的，所有的链接等的问题都是在查找ssh的问题

* clone git仓库的格式问题，git clone ssh://git@104.198.96.241:/gitserver/viney.git
其中@前面的git应该是用户名，类似ssh 用户名@ip这种格式
ip后面是有:的，:后面应该是跟的文件夹路径，但是我一般用到的是git@192.168.1.1:fota-mobile/aaa.git这种格式的，在我自建git的时候，我的gitserver建在了/根目录下，：后面没跟/发现路径不对，我猜测如果是在所连接的用户目录下面的gitserver，可以不带/，命令会自动到所连接用户的目录下找，如果是像我一样的情况，在根目录下，那么：后面的/gitserver是必须的，否则找不到

* 在查找与测试ssh的链接问题的时候，经常会碰到ip在某个命令执行之后无法ping通的问题（规律表现），起初以为是ip又被墙了，最近被墙的比较厉害。所以停止-启动google服务好几次，以更换临时ip（goolecloud停止-启动compute engine有时候可以更换掉临时ip，注意不是重置，重置无法更改掉临时ip），但是后来发现好像不是被墙了，虽然ping不同，但是总是在调试ssh的过程中，在某个命令执行之后突然就莫名其妙的ping不通了，为了验证不是被墙，开启ss的全局模式保证能翻墙，但是开了全局模式还是ping不通；使用手机的4g流量，（共享热点）发现可以ping通了，那样的话应该是客户端自己的网络问题，由于什么原因导致无法ping通google的ip了，联系到在调试ssh的过程中才会出现，一步步排查，发现在git clone ssh或者ssh的命令中会出现连接失败的情况，导致客户端的可known_hosts的链接被标为错误，下次在ssh就超时了，其实标为错误之后ping就会失效，把known_host删除，下次链接客户端会自动生成，问题就解决了
导致ping失效的命令，如下：

> tiger@jiang:~/Desktop/MyWorkSpace/temp$ git clone ssh://git@104.198.96.241:/gitserver/viney.git
正克隆到 'viney'...
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:9/NzkbfJZ7vN0DPGdXoSblP0NNzXQ9XYk6ZodBvYI2E.
Please contact your system administrator.
Add correct host key in /home/tiger/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /home/tiger/.ssh/known_hosts:3
remove with:
ssh-keygen -f "/home/tiger/.ssh/known_hosts" -R 104.198.96.241
ECDSA host key for 104.198.96.241 has changed and you have requested strict checking.
Host key verification failed.
fatal: Could not read from remote repository.

> Please make sure you have the correct access rights
and the repository exists.

* ssh的密钥添加，就是正常操作了，找到客户端的.ssh的pub，打开之后copy里面的内容到服务端的对应用户的.ssh，添加至authorized_keys中即可。一般也不需要更改sshd_config的内容，其实里面默认是开了那些个权限的比如RSAAuthentication yes、PubkeyAuthentication yes等。注意googleCloud中会给你的默认用户，我的是jiangzenghe自动添加更改authorized_keys，所以git服务对应的需要新建一个用户，指定连接并修改其authorized_keys，google对应的jiangzenghe会经常被删除或修改，导致我添加的密钥无效。注意！！！ 