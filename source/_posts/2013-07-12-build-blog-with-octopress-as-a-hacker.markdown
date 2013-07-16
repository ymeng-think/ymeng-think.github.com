---
layout: post
title: "Octopress帮你像黑客一样写博客"
date: 2013-07-12 15:02
comments: true
author: Meng Yu
published: true
categories: [Blog, Octopress]

---
{% img right /images/tech/build_blog_with_octopress_as_a_hacker/octopress.png 280 210 'image' 'images' %}

### 在开始创建自己的博客之前，我们首先要准备：

  * [创建个人Github帐号](https://github.com/signup/free)  
  * [安装Ruby 1.9.3](http://rvm.io/rubies/installing)
  
### 开始配置个人Octopress博客

假设已经创建了一个Github帐户：rubyonchina

	$ cd ~/dev/
	$ git clone git://github.com/imathis/octopress.git rubyonchina.github.com
	
将 ~/dev/rubyonchina.github.com/.rvmrc 的内容修改为：

	rvm use 1.9.3@rails31
	
安装相应的Gem：
	
	$ cd ~/dev/rubyonchina.github.com
	$ bundle update
	
*如果你正在使用 Ruby 1.9.2，则在执行“bundle update”过程中可能遇到 rdiscount 2.0.7.3 安装失败的问题。切换至 Ruby 1.9.3 问题解决。*


然后生成模版文件：

	$ rake install
	
在Github上创建一个新的Repository，名为：rubyonchina.github.com

添加远程Repository路径：

	$ git remote add rubyonchina git@github.com:rubyonchina/rubyonchina.github.com.git
	
生成静态站点：
	
	$ rake generate
	
配置Octopress与Github的连接：
	
	$ rake setup_github_pages
	
	该命令将：
	
		1. 询问你的Github Pages repository的URL。
		2. 将.git/config中 imathis/octopress 的远程路径，由“origin”改为"octopress"。
		3. 将你的 Github Pages repository 添加为默认的 origin remote。
		4. 将当前的活动分支由 master 切换为 source.
		5. 根据你的 repository 配置博客的URL。
		6. 在_deploy目录中为部署创建 master 分支。
	
根据提示填入你的Github项目网址，本示例是：

	$ git@github.com:rubyonchina/rubyonchina.github.com.git
	
部署到Github上：

	$ rake deploy
	
此时，当前目录中仍有一些目录没有被Git管理，可能如下所示：

	$ git status 
	# On branch source 
	# Your branch is ahead of 'origin/master' by 716 commits. 
	# 
	# Changes not staged for commit: 
	#   (use "git add <file>..." to update what will be committed) 
	#   (use "git checkout -- <file>..." to discard changes in working directory) 
	# 
	#		modified:   .rvmrc 
	#		modified:   Gemfile.lock 
	#		modified:   Rakefile 
	#		modified:   _config.yml 
	# 
	# Untracked files: 
	#   (use "git add <file>..." to include in what will be committed) 
	# 
	#		sass/ 
	#		source/
	
需要将这些文件提交至source分支上，如下：

	$ git add .
	$ git commit -m 'your message'
	$ git push origin source
	
此时，你会注意到Github上的 rubyonchina.github.com 库中出现了两个分支：

{% img left /images/tech/build_blog_with_octopress_as_a_hacker/two-branches-demo.png 932 187 'image' 'images' %}

好了，你已经在Github上创建了自己的博客，那么开始动手写吧！

### 参考文章
* [Octopress](http://octopress.org/)
* [Octopress——像黑客一样写博客](http://www.yangzhiping.com/tech/octopress.html)

	
