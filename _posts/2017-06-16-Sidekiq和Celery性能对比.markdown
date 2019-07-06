---
title: Sidekiq和Celery性能对比
date: 2017-06-16 12:24:46 +0800
categories: 计算机技术 
tags:
 - sidekiq
 - celery
 - python
 - ruby
---

Sidekiq和Celery分别是Ruby和Python中的后台任务异步队列，很想了解它们之间的速度如何，所以做了个简单测试，最后的结果让人惊讶。

<!-- more -->

测试环境
=======
操作系统: CentOS Linux 7
CPU: i7
内存: 16GB
硬盘：机械硬盘

安装Redis
========
安装过程参照了Redis官方网站的说明，

	$ wget http://download.redis.io/releases/redis-3.2.9.tar.gz
	$ tar xzf redis-3.2.9.tar.gz
	$ cd redis-3.2.9
	$ make


安装Celery
=========
Celery是Python的包，所以通过pip安装

	$ pip install celery-with-redis
	$ pip install flower
	$ pip install redis


安装Sidekiq
==========


前提是Ruby环境已经安装好，最新的sidekiq需要Ruby 2.2.2以上版本，而CentOS自带的自有2.0的版本，所以要先安装rbenv环境。

1. 安装rbenv

	git clone https://github.com/rbenv/rbenv.git ~/.rbenv

2. 编译

	$ cd ~/.rbenv && src/configure && make -C src

3. 设置变量

	echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
	echo 'eval "$(rbenv init -)"' >> ~/.bashrc

4. 安装Build插件

	$ yum install -y gcc openssl-devel libyaml-devel libffi-devel readline-devel zlib-devel gdbm-devel ncurses-devel
	$ git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

5. 安装设置新版Ruby

	$ rbenv install 2.4.1
	$ rbenv global 2.4.1
	
6. Sidekiq是Ruby的包，安装更简单

	$ gem install sidekiq
	$ gem install redis-namespace

经过上面的步骤，环境就已经配置好了，接下来准备测试

测试Celery的速度
===============
首先来测试Celery

1. 编写一个名为tasks.py的文件，在这个文件中定义异步方法

	from celery import Celery

	app = Celery('tasks', broker='redis://localhost/')

	@app.task
	def add(x, y):
	    return x + y

这里我们简单点，就做一个加法

2. 创建测试文件，命名为test_tasks.py，内容如下:

	from tasks import add
	i = 0
	while i<100000:
		x = add.delay(5, 6)
		i += 1

测试文件里，做10万次的任务。注意，不要在这里计算时间，因为是异步的。

3. 在后台启动Redis服务

	$ ./redis-3.2.9/src/redis-server &

4. 启动Celery任务，一样是在后台

	$ celery -A tasks worker &

5. 启动监控web

	$ celery flower --broker=redis://localhost:6379 --address=0.0.0.0 &

这项启动后，可以通过浏览器访问这台机器的5555端口，看到任务的执行状态。

6. 最后启动测试代码

	$ python test_tasks.py

7. 分析测试结果
通过浏览器上的观察，看到Celery开始接受数据是在14:43，处理完10万个任务后的时间是17:16，总过是153秒，通过简单的数学可以知道

	10万/153 = 654

Celery默认跟进CPU的核心数启动后台进程的数量，I7的CPU是8个核心，所以Celery起了8个进程在后台处理。不过，一秒钟只能处理650多个任务，还是让人挺失望的。

Celery的后台还可以用eventlet来处理，如果把上面的第4部换成

	$ celery -A tasks worker --concurrency=1000 -P eventlet > /dev/null 2>&1

得到的结果是，处理完10万个任务花了144秒（开始02:49，结束05:13），每秒也就

	10万／144 = 694.44

与前面的每秒654相比，性能并没有显著提升。

测试Sidekiq的速度
===============
Sidekiq的官方说明,Sidekiq的速度可以达到每秒4500个任务，那我们就测一下

1. 任务代码
从这下载原始的测试代码`https://raw.githubusercontent.com/mperham/sidekiq/master/examples/por.rb`，然后修改

	puts "Workin' #{how_hard}"

为
	5 + 6

我们也是仅仅做个加法，跟测试Celery一样

2. 编写一个测试脚本，命名为test_por.rb

	require './por'

	i=0
	while i<100000
		PlainOldRuby.perform_async "like a dog", 0

由于我们修改了任务代码，这里的第一个参数就没有实际意义了，最后一个参数是不让它睡眠，有多快就执行多快

3. 启动后台任务
如果按照本文的顺序来，redis服务应该已经起来，如果没有，需要首先启动redis服务。

	$ sidekiq -r ./por.rb

4. 启动web后台监控
	
建立一个config.ru文件

	require 'sidekiq'

	Sidekiq.configure_client do |config|
	  config.redis = {:namespace=>'x', :size => 1 }
	end

	require 'sidekiq/web'
	run Sidekiq::Web

使用rackup运行
	
	$ rackup -o 0.0.0.0 config.ru

现在可以通过浏览器访问这台电脑的9292端口来实时查看任务的状态了

5. 运行测试脚本

	$ ruby test_por.rb

6. 分析结果
Sidekiq接受任务的时间是39:27，处理完10万个任务的时间是39:53，总共是26秒，每秒处理任务的数量是

	10万／26 ＝ 3846

虽然离官方的数据还有点差距，但是已经很接近了。SideKiq默认采用一个队列，25个线程处理任务。

总结
===

在相同的测试条件下测试了Celery和Sidekiq的速度，得到的结果是Sidekiq处理任务的吞吐达到3846/秒，是Celery（694／秒）的5.5倍, 可以说是完胜。



