---
title: 这个博客是怎么搭建的-使用github+hexos搭建个人博客教程
date: 2019-04-08 18:49:35
tags:
    - 随笔
---

# github建博客
## 参考链接
1. 我是如何利用Github Pages搭建起我的博客，细数一路的坑--https://www.cnblogs.com/jackyroc/p/7681938.html
2. 不蒜子，两行代码 搞定计数--http://busuanzi.ibruce.info/

## 前置
<!--more-->

1. 安装Node
2. 创建一个github账户
3. 安装git
4. 下定自己也要整一个博客的决心
5. 本地的`ssh key`添加到github

## 流程
1. github上建一个新的仓库。命名为`你的github名.github.io`。例如我自己的账户仓库命名就是`charleszhaowj.github.io`。
2. 开启Github Pages。点击项目名称进入项目后，选择`settings`当中打开github Pages。如果按照步骤一进行了设置后，该项目就会自动开启github Page。
3. 安装hexos。新建一个文件夹（路径上最好不要有中文？）用于储存博客项目，然后在该文件夹当中打开git bash（右击，选择git bash），然后输入如下命令：
```sh
npm install hexo-cli -g   
hexo init #初始化网站   
npm install   
hexo g #生成或 hexo generate   
hexo s #启动本地服务器 或者 hexo server,这一步之后就可以通过http://localhost:4000  查看了
```
打开 http://localhost:4000 ，如果创建成功，这时候就可以在这个网址看到模板页面了。
4. 写一个小文章。继续在命令行当中输入`hexo new "文章名" #新建文章`新建一篇文章。这篇文章默认创建于`/source/_posts/文章名.md`当中，通过对该markdown文件进行编辑即可完成该文章的写作。可以先不对这个文件进行编辑，其它关于使用hexo写博文的markdown语法更多的详细说明会在后续继续阐述。其它一些常用的hexos命令：
```
hexo n == hexo new
hexo g == hexo generate
hexo s == hexo server
hexo d == hexo deploy
```
5. 可以对博客进行预览了。首先在命令行中输入`Ctrl+C`结束上一次的运行。然后通过`hexo g`重新生成博客，之后通过`hexo s`启动博客。打开 http://localhost:4000 ，如果创建成功，这时候就能看到自己的博文已经出现在博客里面了。
6. 添加主题（给自己的博客换一个好康的模板）。 `Ctrl+C`结束运行后，在命令行当中输入：
```sh
hexo clean
git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia   
```
找到目录下的`_config.yml`文件,打开找到theme属性并且将其设置为：`theme:yilia`。
然后更新主题。继续在命令行当中输入：
```sh
cd themes/yilia
git pull
hexo g
hexo s
```
此时刷新http://localhost:4000/页面就能看到新的主题了。
7. 使用Hexo deploy部署到github
还是编辑根目录下_config.yml文件
```yml
deploy:
  type: git
  repo: git@github.com:cczeng/cczeng.github.io.git  #这里的网址填你自己的
  branch: master   
```
然后安装一个扩展：命令行当中输入（记得返回博客文件的根目录）`npm install hexo-deployer-git --save   
`
8. 部署到github。输入命令`hexo d`。然后打开`你的名字.github.io`就能看到一个好康的个人博客页面啦，是不是超勇的？（手动滑稽）

## 更进一步
是不是发现还有一些地方可以编辑呢？比如切换主题、Markdown语法、添加图片等比新游戏还刺激的内容还想懂？那样的话再给几个链接啦，后续也会有进一步的说明的：

1. 编辑yillia主题：https://github.com/litten/hexo-theme-yilia
2. 换电脑/别的电脑上怎么编辑：https://www.zhihu.com/question/21193762
3. Markdown语法说明：https://www.jianshu.com/p/1e402922ee32/
4. 想要给网站添加图片？Hexo发布博客引用自带图片的方法：https://www.jianshu.com/p/cf0628478a4e 
5. 此外，添加图片也可以把图片放入根目录 source\ 下建立一个文件夹，当你执行hexo g的时候此文件夹自动生成到public中，然后通过markdown语法对图片进行引用即可。