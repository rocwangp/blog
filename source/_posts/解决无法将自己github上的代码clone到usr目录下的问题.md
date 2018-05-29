---
title: 解决无法将自己github上的代码clone到/usr目录下的问题
date: 2018-05-28
categories: github
tags: github
---



Linux下代码存在的位置有两种

* /home下面，需要当前用户权限
* /user/local/include以及其他类似的位置，需要root用户权限

与之对应的.ssh位置也有两个

* ~/.ssh，保存当前用户的public key
* /root/.ssh，保存root用户的public key



<!--more-->



## 将代码clone到/home目录下

申请ssh公钥（email替换成注册github时候的邮箱）

```shell
ssh-keygen -t rsa -C "email"
```

一路回车（如果提示重名说明之前创建过公钥，如果之前的公钥没用了就先删掉，或者重命名）

最后会在~/.ssh/下面出现id_rsa和id_rsa.pub两个文件，将id_rsa.pub中的内容复制到github中即可

操作流程

* 点击Github网站用户头像下的Settings
* 点击左侧SSH and GPG keys
* 点击New SSH key
* 随便写上title，将Key填入id_rsa.pub中的内容
* 点击Add SSH key

此时就可以将github上的代码clone到/home/下面，也可以push上去



## clone到/usr/local/include下

申请ssh公钥（注意这里多了一个sudo，会改变文件的创建位置）

```shell
sudo ssh-keygen -t rsa -C "email"
```

一路回车

最后会在/root/.ssh/下面出现id_rsa和id_rsa.pub两个文件（先使用su命令进入root用户，再进入/root/.ssh/目录）

采用同样的方法将id_rsa.pub添加到github上

此时就可以将github上的代码clone到/usr/local/include下面，同时也可以push上去

