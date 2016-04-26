---
title: 第一篇博客
date: 2016-04-25 19:47:31
catogories:
- 随笔
tags:  
- Hexo 
- Pacman
---

总算搭好了自己的博客，希望能记录下一些东西吧。中间也绕了不少弯路，整理出来供查阅。目前字体的大小和颜色以及代码高亮都还不是很满意，有空再调整一下。

搭建的过程主要是参考了官方文章和网上搜罗的一些教程。

1. [Github托管](https://github.com)，有300M免费空间，免费。
2. 域名管理：[dnspod](https://www.dnspod.cn/)，免费。
3. 域名是在[阿里云(万网)](https://wanwang.aliyun.com/)购买，比GoDaddy便宜不少（即使我找到了35%的折扣券）。原因是一开始GoDaddy支付始终不成功，然后发现阿里云有购买域名的服务，而且更便宜。
4. 博客使用Hexo框架搭建，原因是比Jekyll感觉要简单方便，主题Pacman也是原因之一。
5. 图床使用[七牛](http://www.qiniu.com/)，可以使用提供的接口命令行传图片会更方便一点。

参考的链接整理出来如下

### 参考文章
推荐阅读顺序按照序号来

1. 从零开始  
     
     > [从域名到Github账号到新建博客模板](http://cnfeat.com/blog/2014/05/10/how-to-build-a-blog/)全方位的介绍，并且在博客结尾有许多其他博主写的相关搭建经验的文章 
2. [GitHub+Hexo+绑定自定义域名](http://blog.zfan.me/2015/09/03/%E4%B8%BA%E9%83%A8%E7%BD%B2%E5%9C%A8Github%E4%B8%8A%E7%9A%84Hexo%E5%8D%9A%E5%AE%A2%E7%BB%91%E5%AE%9A%E4%B8%AA%E6%80%A7%E5%9F%9F%E5%90%8D/)  
>添加CNAME文件到 ./sources/

3. 使用hexo，如果换了电脑怎么更新博客？
> [参考排名第一的回答](https://www.zhihu.com/question/21193762)。他引用的自己的[blog](http://crazymilk.github.io/2015/12/28/GitHub-Pages-Hexo%E6%90%AD%E5%BB%BA%E5%8D%9A%E5%AE%A2/#more)说得更清楚。注意在`_config.yml`中的`deploy`中的`repo`参数值需要填写你的`git`仓库的`SSH的地址`,而不是`HTTP的地址`，否则在执行`hexo deploy`，既不会报错，也不会按照你预想的进行

4. 一台机器有多个Git账号怎么办？
> 平时拥有两个github账号，在根据上面的教程执行到最后一步时，由于我全局使用的是工作的git账号，导致git总不成功，跟ssh-key有关，这个博客有[解决方案](http://www.tuicool.com/articles/zqa6Rz)
> 有几个用户就生成几个ssh-key，并且`ssh-add`，然后在`~/.ssh/config`中写好对应的别名配置，然后在使用时 使用别名代替`repo url`，就可以在配置中找到对应的`email`以及`ssh-key` 进行操作了。

5. 修改评论为DISQUS
> [解決 Hexo Comment !](http://morris821028.github.io/2014/04/12/web/hexo-comment/)

### 官方文章
官方的介绍以及教程

1. [Github Pages](https://pages.github.com/)
2. [hexo.io](https://hexo.io)
3. [Pacman主题](https://yangjian.me/workspace/introducing-pacman-theme/)
4. [DISQUS](https://disqus.com/)
