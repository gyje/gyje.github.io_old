---
layout: post
title: '用聊天机器人控制Arduino开关灯'
subtitle: ''
date: 2018-08-18 14:02:12
categories: 技术
cover: 'https://isujin.com/wp-content/uploads/2018/03/wallhaven-614370.jpg'
tags: Arduino IOT
---

这是毕业设计做的一个小项目，实现了一个利用QQ机器人控制arduino开关灯的物联网应用，论文设计中考虑了多种情况，用到了很多东西，还实现了一个DSL语法解析引擎，用来做语义判断，这里我们不介绍这么多，只做一个最简单的实现。

### 所需物料：
1. Arduino开发板
2. W5100网络通信模块
3. 网线等基础条件物料
4. 一个LED发光二极管或者继电器模块

### 通信及控制流程
![控制流程图](https://upload-images.jianshu.io/upload_images/2348227-0e061b3d368d20a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  简单解释：
User发送Control Command到Web Server，W5100轮询Web Server，得到的结果返回给Arduino，程序根据结果执行开关LED/继电器等操作。

#### 注意要点
Web Server可以没有公网IP，但是要保证能让W5100和QQ Robot Framework访问到。

### 整体设计
#### 一些废话
web server本来是用php/python实现的，但是要交付给用户（老师）还需要用户安装python或php，考虑使用pyinstaller打包，但是效果不是很好，所以改成使用golang，直接给一个二进制打包文件即可，其实golang也不是最优方案，因为这个web server需要的功能非常简单，用c/cpp写一个也行，甚至还考虑了powershell。

#### Web Server功能及设计
根据传入的token验证请求，然后改变一个全局变量，以便Arduino根据该变量做相应控制动作。
```go
/*
1. 程序运行后会在0.0.0.0开启80端口,请运行前确保80端口未被占用且在防火墙白名单
2. 原始电平为0
3. GET http://{host}/console.txt得到指令状态
4. GET http://{host}/change?level={0,1}&token=arduino_project_demo修改状态
*/
package main

import (
	"fmt"
	"net/http"
	"time"
)

var level = "0"
var token = "arduino_project_demo"

func console(w http.ResponseWriter, r *http.Request) {
	fmt.Printf(time.Now().Format("2006-01-02 15:04:05"))
	fmt.Println(" ", r.RemoteAddr, " ", r.Method, r.RequestURI, "200", r.Proto)
	fmt.Fprintf(w, level)
}

func change(w http.ResponseWriter, r *http.Request) {
	fmt.Printf(time.Now().Format("2006-01-02 15:04:05"))
	fmt.Println(" ", r.RemoteAddr, " ", r.Method, r.RequestURI, "200", r.Proto)
	r.ParseForm()
	get_token := r.Form.Get("token")
	if get_token != token {
		fmt.Fprintf(w, "token error!")
	} else {
		get_level := r.Form.Get("level")
		if get_level == "0" || get_level == "1" {
			level = get_level
			fmt.Fprintf(w, "success!")
		} else {
			fmt.Fprintf(w, "level error!")
		}
	}
}

func index(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path != "/" {
		w.WriteHeader(404)
		w.Write([]byte("404"))
		fmt.Printf(time.Now().Format("2006-01-02 15:04:05"))
		fmt.Println(" ", r.RemoteAddr, " ", r.Method, r.RequestURI, "404", " ", r.Proto)
	} else {
		w.Write([]byte("index"))
		fmt.Printf(time.Now().Format("2006-01-02 15:04:05"))
		fmt.Println(" ", r.RemoteAddr, " ", r.Method, r.RequestURI, "200", r.Proto)
	}
}

func main() {
	fmt.Println("ListenAndServe: 80")
	http.HandleFunc("/", index)
	http.HandleFunc("/console.txt", console) //设置访问的路由
	http.HandleFunc("/change", change)       //设置访问的路由
	http.ListenAndServe(":80", nil)          //设置监听的端口
}
```

#### QQ Robot
使用酷Q机器人框架，用易语言（真难用，一堆槽点~）做插件开发，**功能就是根据User的指令发送相对应的HTTP 请求**，也是很简单的程序，由于该语言源代码是二进制文件，无法贴出相应代码。

#### Arduino代码
简单来说就是定时访问web server然后根据返回结果做出对应的高低电平控制。
```c
#include <SPI.h>
#include <Ethernet.h>
byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED }; // 不要和局域网中的其它设备MAC地址冲突,一般不需要修改
char server[] = "x.x.x.x"; //改成服务器的IP
IPAddress ip(192, 168, 0, 101); //Arduino的IP地址,不要和局域网中的其它设备IP地址冲突
EthernetClient client;
void setup() {
  pinMode(6, OUTPUT); //Arduino的引脚,另一端接GND,不要接反
  Serial.begin(9600);
  while (!Serial) {
    ;
  }
  if (Ethernet.begin(mac) == 0) {
    Serial.println("使用DHCP配置网络失败!");
    Ethernet.begin(mac, ip);
  }
  delay(1000);
  Serial.println("连接中...");
}

void loop() {
  if (client.connect(server, 80)) {
    client.println("GET /console.txt HTTP/1.1");
    client.println("Connection: close");
    client.println();
    Serial.println("get over");
  } else {
    Serial.println("connect failed");
  }
  delay(1000); //必须有的延时时间,否则连接失败
  String console;
  while (client.available()) {
    char c = client.read();
    console = console + c;
  }
  Serial.println("console的值是:");
  Serial.print(console);
  if (console.endsWith("0")) {
    digitalWrite(6, LOW); //和前面的一致
  } else {
    digitalWrite(6, HIGH); //和前面的一致
  }

  if (!client.connected()) {
    Serial.println("connect stop");
    client.stop();
  }
  delay(100); //延时时间,可以微调,让响应更快,或者更慢.
}
```
### 展示
图片都在手机上，有时间贴出来。

### 结论
现在一些arduino物联网设计应用动不动就要用一些第三方平台，流程比较繁杂，其实很简单的几段代码就能做出一个控制应用，同理，我们也可以把QQ换成微信、短信、Telegram，做出更复杂的东西，例如：浇花，开关门锁等等。
