# 电力计量——NodeMCU+Influxdb+Grafana
主要由一下几个部分构成：

- **数据库：Influxdb——开源的时序数据库**
- **前端：Grafana——开源的图表展示**
- **数据采集：NodeMCU**
- **传感器:电力计量模块**

[TOC]

基础技能：
- C/C++技能点，用于写开发板的程序，不过发现学了C#再来写这个没差多少。
- 硬件连接，串口啦，通信啦，调试啦。不过这里不多讲这个。

## InfluxDB数据库
参考资料1中讲的比较详细，但是有一些不太清晰。到这里梳理一下。
InfluxDB是InfluxData套件中的一个，最初是设计给做运维监控系统使用的，最初版本包含了网页端的管理页面，可以查询管理表等，在1.2.4版本以后就被剥离出来，总共分为了四个模块：
- Telegraf（被监控机器的信息获取）
- Influxdb（时序数据库）
- Chronograf（监控面板和权限管理）
- Kapacitor（即时数据流分析引擎？原文是[Kapacitor is a Real-time Streaming Data Processing Engine](https://www.influxdata.com/time-series-platform/kapacitor/)）。

![Tick](https://i.loli.net/2018/01/12/5a5854f61f380.png)
[InfluxData官网介绍](https://www.influxdata.com/time-series-platform/)


这里按参考资料1中配置好，如果需要管理数据的可以再下载chronograf下来一样配置下地址，命令行打开服务，按说明填好数据库地址端口，数据库名就可以了。
![chronograf](https://i.loli.net/2018/01/12/5a5857ecbaf90.png)
图例为在chronograf直接查询数据库的数据并预览图表

其他功能就不介绍了，看下界面进去看就知道是干什么的了。

### 数据库表和查询、插入数据
Windows和Linux一样，进入文件夹，通过命令行来连接本地或者远程Influxdb数据库
`influx.exe #连接本地数据库`
`influx.exe -host IP -precision rfc3339 #连接远端地址为IP的数据库，并以rfc3339格式显示时间戳`

这里命令不讲太多，详细的参见参考资料2中的[中文文档](https://jasper-zhang1.gitbooks.io/influxdb/content/)。

>E:\test>influx.exe -precision rfc3339
\#连接数据库
Connected to http://localhost:8086 version 1.4.2
InfluxDB shell version: 1.4.2
\> auth
\#认证，默认的用户名密码为admin和123456，可在配置文件里更改
username: admin
password:
\> show databases
\#查看所以数据库
name: databases
name
\----
_internal
\> create database Test
\#创建Test数据库
\> show databases
\#查看刚刚创建的数据库
name: databases
name
\----
_internal
Test
\> use Test
\#默认使用Test数据以进行后续操作，相当于进入那个数据库
Using database Test
\> insert Power,node=1 Vol=231.5,Amp=0.125,Kwh=1.05,Watt=10.5
\#插入一条数据到Power表中，tag值为1，field值为后面4个值
\> show measurements
\#查看所有的表
name: measurements
name
\----
Power
\> select * from Power
\#查询Power表中的所有数据
name: Power
time  　　　　　　　　　    　　　　Amp　Kwh　　Vol　　Watt　node
\----　　　　　　　　　　　　　　　---　　---　　---　----　----
2018-01-12T07:10:48.4141739Z　0.125　1.05　231.5　10.5　1

到此创建了Test数据和Power表并在Power表中插入了一条数据。
现在试一试用Influxdb的HTTP API接口直接写入数据，为后面的NodeMCU上传数据做好准备。

### HTTP　API上传数据至InfluxDB
参考中文文档，通过HTTP协议中的POST方法即可将数据写入InfluxDB中，方法是POST数据库所在的地址加上/write?db=Test，然后POST需要上传的数据。
完整地址即：
`http://localhost:8086/write?db=Test`

这里借助[Postman](https://www.getpostman.com/)这个软件，可以模拟http的各种协议操作，安装完即可使用。
![postman](https://i.loli.net/2018/01/12/5a5867eac7006.png)
选择POST方法，内容选择raw格式，下面的POST内容填写要insert的内容，这里其实跟命令行插入数据很相似。如果POST成功会返回204状态码
![POST204](https://i.loli.net/2018/01/12/5a58696369aaa.png)

这时我们返回命令行，查询Power表就能看到两条数据了。
至此，InfluxDB配置完毕，可以正常接收数据。

## Grafana配置
配置并添加好InfluxDB的数据源，参见参考资料1、4、5中的Grafana，这里只讲下后面的设置图形大概流程。

先添加一个新的Dashboard，相当于监控面板，然后添加一个graph（图形），点击标题来对要展现的数据进行配置。
比如刚刚是Power表中的4个值，直接通过勾选变量来显示出来。
![grafana1](https://i.loli.net/2018/01/12/5a586d1643ae1.jpg)  
勾选数据

![grafana](https://i.loli.net/2018/01/12/5a586d719795d.jpg)  
这里的selectors可以选择是平均值还是其他值，一般改成last，即最新的值。

![grafana2](https://i.loli.net/2018/01/12/5a586de8f094d.jpg)  
查询到数据的效果。

其他的显示效果就慢慢折腾把，什么单位，曲线颜色，尽量搭配出自己满意的效果。

## NodeMCU程序编写。
开发版用的NodeMCU的1.0版本。开发环境用上一篇文档中的VSCode加上Arduino库来进行开发。
同样是使用Arduino库中的示例，HTTPClient来进行POST操作。
```c++
    HTTPClient http;
    const String apiAddress = "/write?db=Test";//post的服务器的地址后缀

    int inx = http.begin("192.168.1.110", 8086, apiAddress);
    Serial.println(inx);
    String PostData = "Power,node=1 Vol=231.5,Amp=0.125,Kwh=1.05,Watt=10.5";
        //测试数据上传
    Serial.println("\r\n************\r\nPost:");
    Serial.println(PostDataStr);
    int httpCode = http.POST(PostDataStr);
    Serial.println("\r\n************\r\nPost End");
    Serial.print("code:");
    Serial.println(httpCode);
    if (httpCode == 204)
    { // 访问成功，取得返回参数
        Serial.println("Getingg payload...");
        //String payload = http.getString();
        //这里屏蔽了访问返回值的代码，因为上传成功后返回的内容为空，后续调试发现会造成bug卡在这一步

        Serial.println("upload success!");
        //Serial.println("U:5." + xxx + "I:0." + yyy);

        //Serial.println(payload);
    }
    else
    { // 访问不成功，打印原因
        String payload = http.getString();
        Serial.print("context:");
        Serial.println(payload);
    }
    ...
```

然后加入电力计量获取的代码，这里因为不同的模块就不贴代码，只要把获取到的数据拼接好成String字符串再通过上面的方法来POST就好了。
**注意：这里拼接好的POST数据要严格保证只有一个空格，即：[表名,tag值 field=Value]，tag值与field用一个空格隔开，多个field值用逗号隔开，否则上传不会成功。**

硬件还是讲一下把。通信是通过NodeMCU自带的串口引脚，不过因为串口电平的原因折腾了蛮久，电力模块的是5v的TTL串口，NodeMCU是3.3V的串口，无法通信，后面接了个三极管就能正常通信，这个不在我的专业范畴，就不多讲，怕误导了人。

最后，给上总的监控界面的图，已经实际运行了一个多月了，总体比较稳定。
![总览图](https://i.loli.net/2018/01/12/5a58731e52e10.jpg)


后续：
工作了一段时间后，发现这个没办法展现每天的用电量哇，没办法，啃文档。还好有中文文档，发现有一个东西叫[连续查询](https://jasper-zhang1.gitbooks.io/influxdb/content/Guide/downsampling_and_retention.html)。这个是在InfluxDB中比较特殊的一个用法。然后还有什么保存策略哇。真是，文档是个好东西。

# 参考资料:
1、[.Net Core 2.0+ InfluxDB+Grafana+App Metrics 实现跨平台的实时性能监控](http://www.cnblogs.com/landonzeng/p/7904402.html)  
2、[InfluxDB中文文档](https://jasper-zhang1.gitbooks.io/influxdb/content/)  
3、[InfluxDB中文文档项目地址](https://github.com/jasper-zhang/influxdb-document-cn)  
4、[数据可视化之 Grafana-Table Panel ](https://testerhome.com/topics/4550?locale=zh-CN)  
5、[Grafana + Zabbix --- 部署分布式监控系统](http://www.cnblogs.com/yyhh/p/4792830.html)  
