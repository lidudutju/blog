打开/Desktop/blog目录，用markdown写文章，然后命名为.md后缀的文件，再修改hexo.sh里的文件名
然后执行命令 sh ./hexo.sh
提示输入密码，输入两次 然后进入到远程服务器
在远程服务器的~目录有一个 hexo.sh文件
执行之，然后就可以再次预览博客了

关于在markdown中如何添加图片：
1.用第三方工具
极简图床
http://yotuku.cn/#/

2. 使用本地服务器存储
在hexo的source目录下新建img文件夹
然后在markdown写文章时用/img/lidu.jpeg来引用


关于文章的配置：

--
title: 
author:
date:
tag:
category:
cover:
---