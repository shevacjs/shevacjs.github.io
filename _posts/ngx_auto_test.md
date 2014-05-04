

##  nginx 自动化测试框架入门简介 ##

author : chenjiansen@baidu.com  
date : 2014/05/03  

---


### 概述 ###

nginx扩展的迭代速度一直比较快，我们希望有一个比较简单单元测试框架，这样一旦代码升级，业务需求变更，都能快速的自测，回归。基本上每种语言都有一些单元测试的支持，比如C/C++的gtest，jave，php，python等等也有。对于nginx的扩展的自动化测试，这里主要介绍的是基于perl的nginx test支持。这套框架是前淘宝的牛人章亦春开发的。我这边亲测，一旦熟习起来，case等东西的编写，测试效率还是非常高的。

---

### 基本原理 ###

我们主要使用的是 `Test::Nginx::Socket` 这个类（具体什么搞的不清楚啊），这个类对外暴露了控制nginx行为（比如日志级别，nginx配置），输入格式（request相关）以及验证 response（body & header）是否满足规则的接口。我们只要描述我们的case 就可以了。

总的说来可以分成两个大步骤，其一是编写自动化case；其二是运行自动化case。 这里分别简单说明这两个大流程是如何开展的。

* __编写自动化测试case__

主要保护如下四个部分:  
1 .描述case部分： 这部分主要用于描述你要编写的case的大致情况，一个demo如下：  
		
	use Test::Nginx::Socket;  
	  
	#说明case的格式
	plan tests => repeat_each() * ( 2 * blocks());  
	$Test::Nginx::LWP::LogLevel = 'debug';  
	run_tests(); #开始运行case
	  
	# 下面开始具体case 行为描述，需要以 __DATA__ 开头
	__DATA__

  
  	
2 .描述nginx行为部分：这部分主要说明nginx的配置，我们可以告诉nginx我们的main，http，server断的配置是怎么样的，具体demo如下:   

	
	--- http_config #说明http段的配置情况
	underscores_in_headers on;
	chunked_transfer_encoding off;
	pbmsgpack_flag on;
	pbmsgpack_conf_path "./nginx/modules/pbmsgpack-transfer-module/conf/proto.conf";
	pbmsgpack_buf_size 16k;
	
	log_format main '$http_x_bd_data_type $request_body $pbmsgpack_transfer_cost $status ';
	access_log ./nginx/modules/pbmsgpack-transfer-module/t/servroot/logs/access.log main;
		
	--- config #说明server端的配置情况
	location = /pb2msgpack {
		pbmsgpack_transfer_flag on;
		echo $request_body;
	}

当然还有描述main段等其他地方的配置情况，不一一列出。  

3 .描述输入部分:  我们可以定义我们的输入是什么样的，比如常用的有控制header和包体内容，如下:  

	
	--- more_headers #设置请求header
	Content-type: multipart/form-data; boundary=--------7da3d81520810*
	x_bd_data_type: protobuf

	--- request eval #支持用perl语法来描述发送的包体内容
	my $bindata;
	open(FF, '</home/users/chenjiansen/dev/pbtest/163_cspv_body.txt') or die "no such file\n";
	binmode(FF);
	read(FF,$bindata,10240);
	close(FF); #以上是读取文件内容
	#发送一个post请求，body内容就是 $bindata
	"POST /pb2msgpack?cmd=303001
	$bindata"

发送包体还有很多很强大的功能支持，比如说pipeline请求，multi-post/get的方式。具体可见后面的推荐文档。

4 .描述预期输出部分: 这里主要说明我们预期返回的结果，如果结果是非预期的，则case就测试失败，比如常见的有:  

	
	--- response_body_like #要求response的返回，意思是要求返回内容包含imgTyep字符串
	.*imgType.*

	--- error_code: 200 #指定返回200

同样的response check也有很多很强大的支持，不再细述。

<br>

* __运行自动化测试case__
当case写好之后，我们关注的是如何运行自动化case，perl的单元测试框架提供了[prove](http://perldoc.perl.org/prove.html)命令，可以很方便的运行相关case，具体的，到`Test::Nginx::Socket`，运行命令一般如下:  


	PATH=/home/users/chenjiansen/nginx_access/sbin/:$PATH prove -r ./t

其中PATH前面的变量指定编译好的nginx的路径（一般的prove命令从这个地方拉取nginx bin文件），`./t` 指定了case所在的目录，一旦运行成功，就会在/t目录生成 `./t/serroot` 目录，里面有最近一次单元测试跑的log, conf等文件。

---

### 简单使用 ###
可以从echo，memc等nginx扩展里面的/t/目录查看具体的case，加以学习，具体有:  
1. [ngx_memc](http://github.com/agentzh/echo-nginx-module)  
2. [ngx_echo](http://wiki.nginx.org/NginxHttpMemcModule)  

---

### 问题&不明之处 ###

这里仅简单说明`Test::Nginx::Socket`的用法，一些更深一点的东西需要进一步跟进，比如:  
1. 更优雅，简单的设置upstream，方便测试  
2. 模块的具体原理，方便做定制化开发  


### 更多 ###
其实网上的资料并不是很多，主要是:  
1. 章亦春自己的官方说明，[点击这里](http://search.cpan.org/dist/Test-Nginx/lib/Test/Nginx/Socket.pm#___top)  
2. [百度官方博客对perl单元测试的介绍](http://baidutech.blog.51cto.com/4114344/744396)  
3. [淘宝对nginx test框架的简单说明](http://www.taobaotest.com/blogs/2433)  

---
