---
layout: post
title:  "20190805PHP搭建socket聊天室"
date:   2019-08-05 19:19:12
categories: PHP
tags: PHP
excerpt_separator: ~~~
---
### PHP聊天室框架



​		聊天室需要使用socket来传递信息,在写聊天室之前,我们先来了解下socket.

#### 1.什么是socket

​		套接字（socket）是一个抽象层，应用程序可以通过它发送或接收数据，可对其进行像对文件一样的打开、读写和关闭等操作。套接字允许应用程序将I/O插入到网络中，并与网络中的其他应用程序进行通信。网络套接字是IP地址与端口的组合。

​		所谓套接字，实际上是一个通信端点，每个套接字都有一个套接字序号，包括主机的IP地址与一个16位的主机端口号，即形如（主机IP地址：端口号）。例如，如果IP地址是210.37.145.1，而端口号是23，那么得到套接字就是（210.37.145.1:23）。

~~~

#### 2.服务器端代码

```php
<?php
	//创建一个socket
    //AF_INET ipv4
    //SOCK_STREAM 一个双工流
    //SOL_TCP 使用TCP协议
	$sock = socket_create(AF_INET,SOCK_STREAM,SOL_TCP);

	//服务器IP
	$host = "127.0.0.1";
	//服务器端口
	$port = 11911;
	//绑定socket
	socket_bind($sock,$host,$port);
	//开启监听	
	socket_listen($sock,1024);
	//等待客户端连接
	$io = socket_accept($sock);
	//获取客户端消息
	$msg = socket_read($io,1024)；
?>
```





![](/home/black/文档/socket/socketserver1.png)



服务器运行之后回挂起.



#### 3.客户端代码

```php
<?php
	//创建一个socket
    //AF_INET ipv4
    //SOCK_STREAM 一个双工流
    //SOL_TCP 使用TCP协议
	$sock = socket_create(AF_INET,SOCK_STREAM,SOL_TCP);

	//服务器IP
	$host = "127.0.0.1";
	//服务器端口
	$port = 11911;
	//连接服务器
	socket_connect($sock, $host,$port);
	//监听输入流
	$msg = fgets(STDIN);
	//向服务器发送
	socket_write($sock,$msg,strlen($msg));
?>
```



![](/home/black/文档/socket/socketClient.png)



运行客户端并且输入aaaa之后

服务器端受到了信息并且打印

但是输出之后服务器就关闭了连接,这显然不是我们需要的聊天室功能。



#### 4.修改server端

既然已经可以接收到了客户端的消息,至少我们目前方向没有错,只不过需要做一点调整.

````php
<?php
	//创建一个socket
    //AF_INET ipv4
    //SOCK_STREAM 一个双工流
    //SOL_TCP 使用TCP协议
	$sock = socket_create(AF_INET,SOCK_STREAM,SOL_TCP);

	//服务器IP
	$host = "127.0.0.1";
	//服务器端口
	$port = 11911;
	//绑定socket
	socket_bind($sock,$host,$port);
	//开启监听	
	socket_listen($sock,1024);
	//等待客户端连接
	$io = socket_accept($sock);
	//获取客户端消息
	
	//循环监听客户端信息
	while(1)
	{
		$msg = socket_read($io,1024);
	
		echo $msg;
	}
?>
````

然后再运行服务器端代码



![多次交互](/home/black/文档/socket/serverwhile.png)



奥,可以接收到所有输入了,那么我们再开个客户端试试

![服务器没鸟第二个客户端](/home/black/文档/socket/serverthree.png)



NO，服务器完全没有响应第二个客户端



​		因为服务器的socket被第一个客户端占用,所以之后的客户端就算连接,也不能和客户端建立连接,将客户端的信息发送到服务器上。

​		那么我们需要再次改写服务器端的代码,每次接收连接之后,不急着监听客户端的消息,而是把连接扔到一个数组里,直到这个数组里面的socket有状态改变,然后再收客户端信息之后再处理。幸好php官方已经给出了socket_select方法



##### 5.server代码

```php
<?php
//创建一个socket
    //AF_INET ipv4
    //SOCK_STREAM 一个双工流
    //SOL_TCP 使用TCP协议
	$sock = socket_create(AF_INET,SOCK_STREAM,SOL_TCP);

	//服务器IP
	$host = "127.0.0.1";
	//服务器端口
	$port = 11911;
	//绑定socket
	socket_bind($sock,$host,$port);
	//开启监听	
	socket_listen($sock,1024);

	//定义3个数组
	$sock_read = array();

	$sock_write = array();
	
	$sock_exp = array();

	//将服务器socket加入读取监听数组
	$sock_read[] =  $sock;

	while(true)
	{
        //复制2个临时数组
		$tmp_read = $sock_read;

		$tmp_write = $sock_write;

        //如果有状态变化
		if(socket_select($tmp_read,$tmp_write ,$sock_exp,null)>0)
		{
            //遍历监听数组
			foreach($tmp_read as $sr)
			{
                //如果是服务器端的socket有变化
				if($sr == $sock)
				{
					//等待客户端连接
					$io = socket_accept($sock);

                    //将新连接放入读数组中
					$sock_read[] = $io;
				}
				else //如果不是服务器socket发生变化
				{
					//获取客户端发送消息
					if($msg = socket_read($sr,1024))
					{
					
						echo "recive msg:".$msg."\n";
						// var_dump($tmp_write);

						foreach($sock_read as $tw)
						{
                            //跳过服务器socket与发送消息的socket
							if($tw == $sock || $tw == $sr)
							{
								continue;
							}
						
							socket_write($tw,$msg,strlen($msg));

							echo "send to {$tw} {$msg}\n";
						}		
					}
					else
					{
						continue;
					}
				}
			}
		}
	}
?>

```

![](/home/black/文档/socket/serverall.png)



现在就可以收到所有客户端的请求了！

