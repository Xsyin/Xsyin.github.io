---
title: hexo github搭建博客
date: 2020-02-19 17:39:12
updated: 2020-02-19 17:39:12
tags:
 - Hexo
 - 教程
categories: 教程
keywords: hexo
copyright: true
---

## 环境安装

> Ubuntu 18.04 

### nodejs     npm

 根据[hexo官方文档](https://hexo.io/zh-cn/docs/) ，建议使用 Node.js 10.0 及以上版本，而此时apt安装版本为8.14，故使用以下方法：

```shell
# Using Ubuntu
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
nodejs -v
npm -v
```

<!-- more -->

因为npm是国外的库，所以没有翻墙的话会很慢，更改为阿里源：

```sh
npm config set registry http://registry.npm.taobao.org
```

### Hexo

```
sudo npm install -g hexo-cli
mkdir blog
hexo init blog
cd blog
npm install
```

`blog`目录为`hexo`的工作空间，目录如下：

.
├── _config.yml
├── node_modules
├── package.json
├── package-lock.json
├── scaffolds
├── source
└── themes

其中，`_config.yml`是博客网站的配置文件,`source`文件夹是存放用户资源的地方，`_posts` 文件夹存放Markdown文件（.md），`scaffolds`是模板文件夹，当用hexo新建文章时，hexo会根据`scaffolds`文件夹下的文件来建立文件，`themes`文件夹是存放主题的文件夹，hexo根据主题生成静态页面。
关于_config.yml的补充说明：其中工作空间根目录下有一个_config.yml，是博客网站的配置文件，themes文件夹中对应不同主题的目录下也有一个_config.yml文件夹，是主题的配置文件。

### 配置

| 命令                    |      作用      |  简写  |
| :---------------------- | :------------: | :----: |
| hexo init "folder-name" |  新建一个网站  |        |
| hexo new "title-name"   |  新建一篇文章  |        |
| hexo generate           |  生产静态网站  | hexo g |
| hexo server             | 启动本地服务器 | hexo s |
| hexo deploy             |    部署网站    | hexo d |
| hexo clean              |  清除缓存文件  |        |

### 更换主题

默认主题为`landscape` ，更换为[NexT ](https://github.com/theme-next/hexo-theme-next) 。

```
#在 blog 目录下
git clone https://github.com/theme-next/hexo-theme-next themes/next
```

在工作目录`blog`下修改配置文件`_config.yml` 行theme: landscape 为

```
theme: next
```

重新生成网站：

```
hexo clean
hexo g
hexo s
```

在浏览器中输入：http://localhost:4000

### git 配置

```
sudo apt install git
git config --global user.name "your name"
git config --global user.email "your email"
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" # 创建git连接公钥
# 将 ssh-key 加入到 ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```

1. 公钥在`~/.ssh/id_rsa.pub`，打开该文件，复制全部内容。在github中，点击个人头像—>settings—>SSH and GPG keys—>New SSH key，粘贴刚才复制的文本。

   ```
   ssh -T git@github.com # 测试是否成功连接
   ```

2. 在github页面，选择New repository，Respository name输入`username.github.io`，点击确定

3. 编辑 `_config.yml` :

   ```
   deploy:
     type: git
     repository: git@github.com:username/username.github.io.git
     branch: master
   ```

4. 安装hexo插件 

   ```
   npm install hexo-deployer-git --save
   ```

 5. 部署

    ```
    hexo clean
    hexo g
    hexo d
    ```

### 上传源码

​        不难发现这个博客仓库是不包含原始文件的，比如文章对应的 **.md** 文件，而该文件都只保存在配置 **Hexo** 的机器本地，而上传到 **GitHub** 的只是转换渲染过后的 **.html** 网页文件。而更换机器后的迁移需要源码

​    在 **GitHub** 仓库中创建新分支 **hexo**，并将部分站点源文件 (如source文件夹，最好写个 **.gitignore** 文件) 提交到该分支，在博客根目录下执行命令：

```
$ rm -rf themes/next/.git # Git项目内不能再包含Git子项目
$ git init
$ git checkout -b hexo # 创建本地分支hexo 
$ git add .
$ git commit -m "create a new branch for coordination among multiple devices" 
$ git remote add origin git@github.com:username/username.github.io  #添加远程仓库 
$ git push origin hexo
```

附： 

```
git push origin --delete hexo     # 删除远程仓库分支
```

### 主题配置

[链接](https://www.jianshu.com/p/9f0e90cc32c2)

1. 设置 Menu

   theme/next/_config.yml：

   ```
   menu:
     home: / || home
     about: /about/ || user
     tags: /tags/ || tags
     categories: /categories/ || th
     archives: /archives/ || archive
     #schedule: /schedule/ || calendar
     #sitemap: /sitemap.xml || sitemap
     #commonweal: /404/ || heartbeat
   
   ```

   相应源码：

   ```
   hexo new page "tags"
   hexo new page "about"
   hexo new page "categories"
   ```

   编辑生成的index.md

2. 字数统计

   ```
   npm install hexo-symbols-count-time
   ```


3. nexT 7.1以上只需仔细阅读主题的配置文件，开启相应开关就行