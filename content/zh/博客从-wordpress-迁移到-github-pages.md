---
title: 博客从 wordpress 迁移到 github pages
date: 2019-06-09 19:40:53
tags:
  - 杂记
---

## 前言

进入6月之后，可能是因为特殊时期吧，我的 aws 被墙掉了，我的博客也没法访问了。考虑过重新买个 aws，不过，考虑到这几天我连续被墙掉了 3 个 vps，还是打算直接使用 github pages 来部署博客，省的折腾。

## 环境部署

现在，生成静态博客的工具还是很多的，有基于 node.js 的 hexo、基于 ruby 的 jekyll 以及基于 golang 的 hugo。我也没做什么详细的比较，听说 hexo 的生态挺好的，就直接使用 hexo 了。

### hexo 环境搭建

hexo 是基于 node.js 的，所以，要先安装 node.js 的环境。node.js 的安装包可以中它的官网找到（[传送门](https://nodejs.org/zh-cn/download/)），它提供了各个平台的安装包以及源码，读者可以按需下载，我的环境是 windows 64 位的，就选择了第一个 64位 Windows 安装包。

![node.js安装包](https://raw.githubusercontent.com/LooJee/medias/master/images/20190609200028.png)

node.js 安装完成之后，就可以安装 hexo 了，它的官网为[传送门](https://hexo.io/zh-cn/)。因为，我对这个工具也不是很了解，这里也没法告诉大家有什么需要注意的地方。下面列一下几个安装命令吧，详细信息请到官网查看文档，官网文档写的十分详细：

```shell
$: npm install hexo-cli -g  //安装hexo
$: hexo init blog			//初始化一个静态网站
$: cd blog
$: npm install				//我对node.js不太熟悉，不知道这是在干什么，可能是配置一些环境
$: hexo server				//启动hexo服务
```

### wordpress 博客导出

hexo 的环境搭建好了，那么如何把 wordpress 的博客导出呢？读者在登陆到 wordpress 的后台之后，按照：选择工具 -> 导出 -> 所有内容 -> 下载导出文件，这样的步骤就可以导出您当前 wordpress 博客上的文章了，它是一个 xml 文件，打开之后可以看到里面是一些文章的信息。

![导出菜单](https://raw.githubusercontent.com/LooJee/medias/master/images/800px-manageexport_zh-hans.png)

### 根据 wordpress 的导出文件生成内容

上面的步骤都完成之后，剩下的工作就是内容迁移了。hexo 官方有一个工具叫做 [hexo-migrator-wordpress](https://hexo.io/zh-cn/docs/migration#WordPress) 是专门用来迁移 wordpress 的数据的。不过，这个工具不是很完美，我迁移了之后，还对生成的 md 文件做了一些微调。使用 hexo-migrator-wordpress 迁移数据的步骤如下：

```shell
$: cd /path/to/your/blog
$: npm install hexo-migrator-wordpress --save
$: hexo migrate wordpress /path/to/your/wordpress.xml
```

上面的步骤执行完成之后，就可以完成博客的迁移了。大家如果想要预览自己的博客的话，使用以下命令:

```javascript
$: hexo g
$: hexo s
```

## 博客发布到 github pages

github pages 的搭建十分方便，主要步骤如下：

- 登陆 [github](https://github.com/) 之后，选择创建新的仓库，仓库的名字命名为 `username/username.github.io 。比如，我的用户名为 loojee 那么我的仓库名称就命名为 loojee.github.io 。
- hexo 官方有一个[发布工具](https://hexo.io/zh-cn/docs/deployment#Git)，它通过十分简单的配置就可以在本地将生成好的页面发送到上面创建好的仓库中，具体命令我这里就不写了，链接所指向的官方文档十分详细。
- 经过以上的步骤，我们就可以通过 http://loojee.github.io 这样的 url 来访问我们的博客了。

## 设置域名

那我之前买的域名怎么办呢，总不能浪费了吧。其实，设置域名也非常简单。原来，我是将域名和一台有 IP 的服务器绑定的，所以，在设置解析的时候我使用的是 A 字段。而现在，loojee.github.io 是一个域名，那么，我们设置解析的时候将 A 字段改成 CNAME 字段，然后，将原来的 IP 更换成现在的 loojee.github.io 就完成域名解析的更改了。更改后，域名解析如下：

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190609210020.png)

但是，这个时候还不能直接通过 tech-seeker.cn 的域名来访问 github pages。进入仓库的设置 -> 找到 Github Pages -> 将 Custom domain 设置为 tech-seeker.cn -> 点击 Save ，经过这些操作后，就可以通过 http://tech-seeker.cn 来访问新的博客了。

但是有一个问题，就是每次发布了新的文章后，都要重新设置 Custome domain，我们可以在 blog 目录下的 public 目录，也就是最后要发布到 github pages 上的目录中新建一个文件，命名为 CNAME，然后，将网站的域名写入就可以了。这样每次发布的时候，CNAME 文件会一起发布到 github pages，就不需要单独设置 Custom domain 了。

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190609212336.png)

## 结语

其实，通过 github pages 来部署博客还是很方便的，hexo 的文档十分丰富，也很方便自定义，就是需要一点学习成本。还省去了买主机的钱，不用为主机被墙而烦恼了，万一 github 也被墙了的话，额。