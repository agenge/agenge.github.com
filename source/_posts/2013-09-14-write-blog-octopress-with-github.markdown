---
layout: post
title: "利用GitHub搭建免费的个人blog"
date: 2013-09-12 22:18
comments: true
categories: 
-  Github
-  Blog[博客]
---

# 前言
## 为啥写篇文章
1. 熟悉WordPress的或许都明白，为了写作，你要学习很多东西，例如Linux、Apache/Nginx、Mysql以及各种中间件；

2. 你要花钱买VPS/虚拟主机，你要自己优化Web Server和Mysql，你要定期备份Mysql；

3. WordPress官方已经被墙（对于合格的IT屌丝，这点不是问题）；

你可能已经在心理产生疑问：“你说这么多费话，无非就是想让我放弃WP，难道你有解决上述问题的完美方案？”
虽不敢说完美解决，但至少有这么一个blog平台：

* 无须数据库
* 无须Web Server
* 无须备份且数据永不丢失（相对自己备份而言）
* 所有操作均有版本记录

看了之后是不是很“鸡动”？没错，这个平台就是Github，结合jekyll/Octopress就完全可以专心写作，
不过相对WP，功能确实要少些，图片和视频功能实在太弱。


## Jekyll/Octopress是啥

Jekyll即是一个轻量级Blog系统，而且还是一个强大的静态网站生成系统，具体的介绍请参考以下链接：

1、[Jekyll建站之旅](http://calefy.org/2012/03/03/my-process-of-building-jekyll-blog.html)

2、[通过GitHub Pages建立个人站点（详细步骤）](http://www.cnblogs.com/purediy/archive/2013/03/07/2948892.html)

3、[Jekyll介绍](http://ztpala.com/2011/09/12/jekyll-and-github-pages/)

Octopress其实就是一个基于Jekyll实现的静态网站系统，具体介绍如下（引用）：

* 使用 Markdown 标记语言书写源文件， 通过 Markdown 解析器转换为 HTML 文件

* 通过 Octopress 提供的站点模板提供所需的 Web 资产文件 （Javascript、CSS、image 等）

* 只包含静态网页，无需数据库支持，对系统要求低且迁移方便

* 以编写程序的方式编制网站，便于实现版本控制

+ Octopress / Jekyll 使用简洁的Ruby框架实现。

    * Octopress 以 rake 任务的形式实现静态站点页面生成, 操作十分简单

    * Octopress 以 rake 任务的形式实现到普通网站和 Github 的发布
    
    * Octopress 与 Github 完美结合，你无需学习过多的 git 命令语法，使非专业人士的使用成为可能    


<!—more—>

#准备工具
说一堆费话，现在说说要准备些什么工具

1. Git for Windows(如果你是Linux/Mac就更方便，不过这篇文章主要是为了Windows下的用户)

    猛撮 [下载Git](https://code.google.com/p/msysgit/downloads/list) 下最新版本。
偶是下载的 Git-1.8.3-preview20130601.exe

2. 安装Ruby环境

    猛撮 [下载RubyInstaller](https://rubyforge.org/frs/?group_id=167)  
    * 如果你是Octopress，请下载 rubyinstaller-1.9.2-p290
    * 如果你是Jekyll，请下载 rubyinstaller-1.9.1-p430
	
 安装之后确保C:\Ruby193\bin 在系统的环境变量中。

3. 安装DevKit

  ruby 的模块工具 gem 在生成本地模块时可能需要用到编译环境，通常有2种选择：

 [MinGW and MSYS](http://www.mingw.org/) 或 [RubyInstaller DevKit](https://github.com/oneclick/rubyinstaller/wiki/development-kit)

 偶是使用的 DevKit-tdm-32-4.5.2-20111229-1559-sfx.exe
 , 在”Start Command Prompt with Ruby”命令行中进入DevKit解压缩的目录：
  
        ruby dk.rb init 
        
        ruby dk.rb install
        gem install rdiscount --platform=ruby
    
 如果没问题，会提示如下信息：

        Fetching: rdiscount-2.1.6.gem (100%)
        Temporarily enhancing PATH to include DevKit...
        Building native extensions.  This could take a while...
        Successfully installed rdiscount-2.1.6
        1 gem installed
        Installing ri documentation for rdiscount-2.1.6...
        Installing RDoc documentation for rdiscount-2.1.6...
 
4. 安装Python for Windows
  
 最后还要安装Python，相对Ruby环境，安装Python就简单许多，下载python-2.7.5.amd64.msi即可。

5. 配置环境变量
环境变量需要在两个地址设置： Windows系统与Git Bash
  * 在你的Windows系统创建两个新的环境变量
    * LANG 环境变量，值设置为：zh_CN.UTF-8
    * LC_ALL 环境变量，值设置为: zh_CN.UTF-8
    
  * 打开Git Bash
  
    * `$ echo "export LANG LC_ALL" > ~/.bash_profile`
    * `$ echo "alias ll='ls -l --color=tty'" >> ~/.bash_profile`
    * `$ echo "alias ls='ls --color=tty'" >> ~/.bash_profile`

6. 准备一个文本编辑器，只要能保存UTF-8无BOM格式的即可，例如UltraEdit.
7. 设置无密码登录github

        ssh-keygen -C "username@email.com" -t rsa

 username@email.com换成你的github注册时的邮箱。

* 创建成功后，在你的C:\Users\%username%\.ssh 目录会生成一个 id_rsa.pub公钥文件。
* 登录github.com（如果没有就注册一个），在“Account Settings”->“SSH Keys”，点击右侧的“Add SSH Key”。
* 将id_rsa.pub文件中的内容复制到“Key”中并保存。

* 测试：
    

       ssh -T git@github.com


 如果没有任何问题，提示如下：
    
        Hi agenge! You've successfully authenticated, but GitHub does not provide shell access.
    


# 初始化Ruby环境
## gem源的问题
作为天朝的屌丝，有时候你总得多墙外的人多做些工作，gem的更新源也要墙？还好还好，国内有良心的企业还是有的（虽然不多），对于gem源，直接使用淘宝的吧。

    gem sources --remove https://rubygems.org/
    gem sources -a http://ruby.taobao.org
如果你觉得不放心，还可以确认是否设置正确：
    
    gem sources -l
## 安装bundler(必须) 和 rdoc(非必须)

    gem install bundler rdoc
    bundle install

# 安装Octopress

    git clone git://github.com/imathis/octopress.git octopress
    cd octopress
    rake install

# 配置Octopress
## 修改_config.yml
你可以根据这个文件你的以下信息：

* title: 网站名称
* subtitle: 网站子名称
* author: 作者名称

除了这3个以外，把Twitter相关全部注释，更多的选项你可以看下注释或Google搜索。

## 发表文章
激动人心的时刻即将到来，终于到了发表文章的这一天，octopress居然把写作与发表文件变得如此简单。

    rake new_post[“tile”]  # 创建一个新文章
执行这条命令之后，会在octopress\source\_posts目录下生成一个"YYYY-MM-DD-title.markdown"的文件
    
在生成静态页面之前，我们需要解决Windows下中文编码的问题，有两个地方需要注意：

1. .markdown文件有中文的话必须保存：UTF-8无BOM
2. 编辑C:\Ruby193\lib\ruby\gems\1.9.1\gems\jekyll-0.12.0\lib\jekyll\convertible.rb第28行，修改为：

    
    self.content = File.read(File.join(base, name), :encoding => 'utf-8')

生成静态页面

    rake generate           # 生成新的静态页面

在本地预览文章效果：
    
    rake preview

 如果成功的话，访问: http://localhost:4000 就能看到效果。
 
# 设置本地仓库与远程仓库关联

* 创建git账号与仓库
    
    * 用户名：username
    * 仓库名：username.github.com

* 关联github仓库
    
    * rake setup_github_pages
    
提示输入远程仓库地址，请输入：
    
    git@github.com:username/username.github.com.git

表示将blog与 username.github.com仓库关联.
 

## 部署Blog到Github
    
    rake deploy
 
这条命令表示发布文件到github。

## 将Octopress源文件推送到 Github的 source 分支

    git add .
    git commit -m “comment”
    git push origin source

## 访问Blog
打开浏览器输入：username.github.com  

如果一切正常，就能看到你的文章。

# rake常用命令

    $ rake -T
    rake clean                     # Clean out caches: .pygments-cache, .gist-c...
    rake copydot[source,dest]      # copy dot files for deployment
    rake deploy                    # Default deploy task
    rake gen_deploy                # Generate website and deploy
    rake generate                  # Generate jekyll site
    rake install[theme]            # Initial setup for Octopress: copies the de...
    rake integrate                 # Move all stashed posts back into the posts...
    rake isolate[filename]         # Move all other posts than the one currentl...
    rake list                      # list tasks
    rake new_page[filename]        # Create a new page in source/(filename)/ind...
    rake new_post[title]           # Begin a new post in source/_posts
    rake preview                   # preview the site in a web browser
    rake push                      # deploy public directory to github pages
    rake rsync                     # Deploy website via rsync
    rake set_root_dir[dir]         # Update configurations to support publishin...
    rake setup_github_pages[repo]  # Set up _deploy folder and deploy branch fo...
    rake update_source[theme]      # Move source to source.old, install source ...
    rake update_style[theme]       # Move sass to sass.old, install sass theme ...
    rake watch                     # Watch the site and regenerate when it changes




Edit By [agenge](http://agenge.github.com)