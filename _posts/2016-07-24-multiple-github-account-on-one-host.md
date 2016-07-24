---
layout: post
title:  "一个客户端设置多个github账号"
date:   2016-07-24 19:05:18 +0800
categories: GitHub
tags: Git GitHub
author: TanQuan
---

* content
{:toc}

### 前言

最近遇到了这样的需求，需要在一台电脑上同时使用两个github账号，负责不同的用途，所以搜索了一些文章，进行了一下实践。





### 1. 前期工作

- 至少有两个[github](https://github.com/)账号 (假设有两个账号 一个为 *one* ，另一个为 *two*)

- 取消git全局设置

```
git config --global --unset user.name
git config --global --unset user.email
```

### 2. SSH配置

- 生成 id_rsa 私钥，id_rsa.pub 公钥 

one 可以直接回车，默认生成 id_rsa 和 id_rsa.pub 。

```
ssh-keygen -t rsa -C "one@126.com"
```

但是 two 会出现提示输入文件名，输入与默认配置不一样的文件名，比如: id_rsa_two。

```
cd ~/.ssh
ssh-keygen -t rsa -C "two@126.com"  # 之后会提示输入文件名
```

- [github](https://github.com/) 添加公钥 id_rsa.pub , id_rsa_two.pub 

分别登陆 *one*, *two* 的账号，在 `Account Settings` 的 `SSH Keys` 里，点 `Add SSH Keys` ，将公钥(.pub文件)中的内容粘贴到“Key”中，并输入”Title”.

- 添加 ssh key

```
ssh-add ~/.ssh/id_rsa
ssh-add ~/.ssh/id_rsa_two
```

可以在添加前使用下面命令删除所有的key

```
ssh-add -D
```

最后可以通过下面命令，查看key的设置

```
ssh-add -l
```


### 3. 修改ssh config文件

```
cd ~/.ssh/
touch config
```

打开.ssh文件夹下的config文件，进行配置

```
# default
Host github.com
HostName github.com
User git
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa

# two
Host two.github.com  # 前缀名可以任意设置
HostName github.com
User git
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa_two
```

这里必须采用这样的方式设置，否则push时会出现以下错误:

> ERROR: Permission to two/two.github.com.git denied to one.

简单分析下原因，我们可以发现 ssh 客户端是通过类似：
`git@github.com:one/one.github.com.git` 
这样的 git 地址中的 User 和 Host 来识别使用哪个本地私钥的。 
很明显，如果 User 和 Host 始终为 git 和 github.com，那么就只能使用一个私钥。 
所以需要上面的方式配置，每个账号使用了自己的 Host，每个 Host 的域名做 CNAME 解析到 github.com，这样 ssh 在连接时就可以区别不同的账号了。

```
ssh -T git@github.com        # 测试one ssh连接
#Hi ***! You've successfully authenticated, but GitHub does not provide shell access.
ssh -T git@two.github.com    # 测试two ssh连接
#Hi ***! You've successfully authenticated, but GitHub does not provide shell access.
```

但是这样还没有完，下面还有关联的设置。


### 4. 新建git项目或者clone已有的项目

- 可以用 git init 或者 git clone 创建本地项目
- 分别在 *one* 和 *two* 的git项目目录下，使用下面的命令设置账号关联

```
git config user.name "__name__"            # __name__ 例如 one
git config user.email "__email__"          # __email__ 例如 one@126.com
```

查看git项目的配置

```
git config --list
```

查看 *one* 的 `remote.origin.url=git@github.com:one/one.github.com.git`

查看 *two* 的 `remote.origin.url=git@github.com:two/two.github.com.git` 

由于 *one* 使用的是默认的Host，所以不需要修改，但是two使用的是`two.github.com`，则需要进行修改

```
git remote rm origin
git remote add origin git@two.github.com:two/two.github.com.git
```

### 5. 上传更改

上面所有的设置无误后，可以修改代码，然后上传了。

```
git add -A
git commit -m "your comments"
git push
```

-- The End --
