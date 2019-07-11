---
title: github 提交问题
date: 2018-05-12 18:24:15
tags:
---

使用git push origin master 命令，提交本地的develop仓库到远程的master分支。
（直接使用这种命令提交是不可以的）
好在后面要求输入Enter passphrase for key '/c/Users/DELL/.ssh/id_rsa': 123
可能输入的密码不对，最后提示了：

git@git.btb-inc.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

提示Permission denied(publickey)。

<!--more-->

另外其他人提交代码提示“”（这种提示是因为github账号不是master/own权限，只是develop权限，只能提交到develop库，不能提交到master库）

最后操作步骤：
通过android studio的cvs ui提交的
1. 把本地develop的代码commit，（可以顺便一起push）
2. 点击右下侧的git栏，更改为本地master，并checkout，这时候本地仓库会变为master，代码会变为master的代码。
3. update（poll）一下最新代码。
4. 点击右下侧的git栏，选中本地develop，并merge，这时候本地master仓库会把本地develop的代码合并进来.
5. 正常提交，将会把合并后的master本地仓库代码提交到远程master仓库。
6. 切回本地的develop分支继续开发。

上述步骤应该可以通过git命令 完美实现。

不要轻易相信网上乱七八糟随便搜罗来的消息，有很多瞎扯淡的，会让你把项目的remote分支搞乱套。
第一个问题可能也并不是key的问题，因为key在创建个人账户关联git的时候已经提交过了。应该不是key错误的问题，而可能是自己输入错误的问题，
但是恰恰因为输入密码错误，阻止了一场将要发生的灾难。