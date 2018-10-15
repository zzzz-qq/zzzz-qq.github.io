---
title: Hexo + GitHub Page 博客诞生记
tags:
    - Hexo
    - GitHub Pages
---
&emsp;&emsp;最近工作关系，需搭建一个SQL Server集群，同样的事情之前就做过一次，但这次仍然麻烦不断，茫茫互联网，频频Google，浪费了好多时间。好记性不如烂笔头，遂决定建个博客，来记录工作学习中的问题。
&emsp;&emsp;开篇，就简单记录一下Hexo + GitHub Page搭建这个博客的过程吧。<!-- more -->
#### 安装 Hexo
[Hexo](https://hexo.io)基于Node，Ubuntu官方源的Node版本很旧，先用[nvm](https://github.com/creationix/nvm)安装新版Node。
npm官方源在国内也比较慢，替换为淘宝的源。
``` bash
# 安装nvm
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash

# 安装最新稳定版Node.js
$ nvm install --lts

# 修改npm源为淘宝
$ npm config set registry https://registry.npm.taobao.org/

# 安装Hexo
$ npm install -g hexo-cli -g
```

#### 搭建过程

1. 在GitHub注册帐号`zzzz-qq`，新建空的repo `zzzz-qq.github.io`
2. Hexo初始化
需要两个分支：hexo分支存放`hexo init`生成的源文件，master分支存放Hexo生成用于部署的静态网站。
新建一个repo`zzzz-qq.github.io.hexo`专门用于管理Hexo源文件也可以，不过我还是偏向于分支。
```bash
$ hexo init zzzz-qq.github.io
$ cd zzzz-qq.github.io
$ git init
$ git remote add origin https://github.com/zzzz-qq/zzzz-qq.github.io.git
$ git checkout -b hexo
```
3. submodule 管理主题
选择了[NexT Theme](https://theme-next.iissnan.com/)，用的人比较多，文档很详细。
另外，若直接clone主题，无法与主题更新保持同步，所以每个主题应该作为一个submodule单独管理，先到GitHub fork要用的主题，主题配置作为单独的commit。
```bash
$ git submodule add https://github.com/zzzz-qq/hexo-theme-next.git themes\next

# 主题配置: 编辑themes/next/_config.yml
$ cd themes/next && git add .
$ git commit -m "init config"
$ git push

# 更新主题，只需要在fork的repo rebase原repo，然后更新submodule
git submodule update --init
```

#### 写作与部署
1. 写文章
```bash
hexo new test_hexo

# 编辑文章 source/_posts/test_hexo.md
```

2. Hexo部署到master分支
* 修改站点配置文件`zzzz-qq.github.io\_config.yml`
```yml
deploy:
  type: git
  repo: https://github.com/zzzz-qq/zzzz-qq.github.io.git
  branch: master
```
* 安装部署工具
```bash
$ npm install hexo-deployer-git --save
```
* 部署
```bash
hexo generate
hexo clean
hexo deploy
```
