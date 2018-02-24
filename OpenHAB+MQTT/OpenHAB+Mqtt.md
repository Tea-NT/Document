# OpenHAB+MQTT+NodeMCU实现远程控制
开发环境基于之前的VSCode环境

OpenHAB简介:
全称为open Home Automation Bus，即[开放式家庭自动化总线](https://oschina.net/p/openhab)，该项目旨在为家庭自动化构建提供一个通用的集成平台。其是一个纯Java打造的开源项目，完全基于OSGi（Open Service Gateway Initiative），并使用Jetty作为web服务器。Jetty和Equinox OSGi运行时一起构成了OpenHAB的核心基础。

nodemcu简介：
很便宜的wifi开发版，可以使用Arduino的库以及IDE编程，本教程使用之前搭好的VSCode来进行测试。

[MQTT](https://baike.baidu.com/item/MQTT/3618851)协议：(Message Queuing Telemetry Transport，消息队列遥测传输）是IBM开发的一个即时通讯协议,

[TOC]

NodeMCU开发版通过MQTT协议与外界通信。以下测试中用到了两个mqtt客户端的库，一个是.Net Core的，一个是ESP8266的。然后OpenHAB托管在Myopenhab上，实现远程控制。

## MQTT服务端选择（以win系统为例）
MQTT的服务端选择国内的开源实现方案——[EMQ](http://emqtt.com/)，官网上下载下来开箱即用，默认不开启验证，下载后解压，命令行运行emqttd console即可开始运行。
```E:\emqttd\bin>emqttd console    #以控制台启动emqttd服务```

在这之前，先把其中一个Emqtt HTTP API服务占用的端口改成8081，与OpenHAB默认的web管理端口冲突，否则后面OpenHAB启动之后无法进入后台，反之改OpenHAB的配置也是可以的。
![emqtt_start](https://i.loli.net/2018/01/15/5a5c5b6844334.png)
EMQTT服务开启

服务启动后就可以进入EMQTT的控制台查看所有的后台控制，后台默认的地址是```http://localhost:18083/```默认用户名是```admin```密码是```public```。
![emqtt_dashboard](https://i.loli.net/2018/01/15/5a5c5c929535a.png)

后面测试用到的基本上有Topics和Clients两个模块，非常简单的就可以监控到连接到的Mqtt的客户端。

既然是用VSCode，就干脆找一找有没有.net的mqtt客户端的实现，一找居然发现github上都已经有.Net Core通过.Net Core写个简单的测试来连接刚刚启动MQTT服务器。

很简单，直接命令行```dotnet new console```一个新的控制台应用（.Net Core的环境很简单，直接下一个SDK装下，然后装C#的插件，就可以了。），然后用Nuget插件，在VSCode中的命令面板里Add Package 搜索M2MqttDotnetCore，选择最新版本就可以了。然后```detnet restore```就完成了添加引用。

添加好引用后，按github中的介绍写一个pub和sub就能测试了。
```C#
//发布端
using System;
using System.Text;
using uPLibrary.Networking.M2Mqtt;
using uPLibrary.Networking.M2Mqtt.Messages;

namespace _1212
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
            MqttClient client = new MqttClient("127.0.0.1");

            byte x = client.Connect("Pubtest");//连接成功为0

            Console.WriteLine(x.ToString());
            while (true)//循环测试发布消息
            {
                string xd = Console.ReadLine();
                client.Publish("/qos1", Encoding.UTF8.GetBytes(xd), MqttMsgBase.QOS_LEVEL_AT_MOST_ONCE, false);
                Console.WriteLine("message send");
            }

        }
    }
}


//订阅端
using System;
using System.Text;
using uPLibrary.Networking.M2Mqtt;
using uPLibrary.Networking.M2Mqtt.Messages;

namespace _1212
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
            MqttClient client = new MqttClient("127.0.0.1");
            client.MqttMsgPublishReceived += client_MqttMsgPublishReceived;
            //注册接收到消息的事件

            byte x = client.Connect("subTest");
            client.Subscribe(new string[]{"/qos1"},new byte[]{MqttMsgBase.QOS_LEVEL_AT_MOST_ONCE});

            Console.WriteLine(x.ToString());


        }

        private static void client_MqttMsgPublishReceived(object sender, MqttMsgPublishEventArgs e)
        {
            Console.WriteLine(Encoding.UTF8.GetString(e.Message));
        }
    }
}

```
生成运行程序之前先把两个项目中的```.vscode/launch.json```文件中的Console改成```"console":"integratedTerminal"```，否则程序会报错，说没有可以输入的设备，改成这个意思就是用内置的终端来当控制台的输入输出。  

两个程序跑起来，就能在Emqtt的DashBoard控制台Clients页面能看到这两个控制端了，Topics页面能看到已经有人订阅的主题/qos1。  

![pub_sub_test](https://i.loli.net/2018/01/15/5a5c655bac97a.png)  
查看连接到emqtt的客户端有哪些

![5、pub_sub_test_topic](https://i.loli.net/2018/01/15/5a5c66013ab0e.png)  
已有的主题

然后在发布端的终端里输入想要发送的数据并回车，订阅端就能接收到数据了。
![sub_pub_test](https://i.loli.net/2018/01/15/5a5c69d5e31f4.gif)
左边为sub端，右边为pub端

如果测试成功，可以接到pub端的消息，OK，下面进行下一步。


## NodeMCU的MQTT库并测试
ESP8266芯片上MQTT客户端键参考资料4，获取方法也很简单。打开Arduino IDE，项目——加载库——管理库，就可以搜索相关的第三方库。
![7、Arduino_ESP8266_MQTT](https://i.loli.net/2018/01/15/5a5c6cefa0603.png)
往下翻一下就能翻到，因为我已经安装了显示```INSTALLED```选择安装最新版本即可。

安装好后，在VSCode中打开ESP8266中MQTTClient的示例，

![8、MQTTexp](https://i.loli.net/2018/01/15/5a5c72e0cce56.png)  

修改代码内容为：

```C++
#include <ESP8266MQTTClient.h>
#include <ESP8266WiFi.h>
MQTTClient mqtt;
#define relay LED_BUILTIN//定义开发版中的LED

void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN,OUTPUT);//设置输出模式

  WiFi.begin("ssid", "pass");//WiFi名称和密码

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  //topic, data, data is continuing
  mqtt.onData([](String topic, String data, bool cont) {
    Serial.printf("Data received, topic: %s, data: %s\r\n", topic.c_str(), data.c_str());
    //mqtt.unSubscribe("/qos1");
    
    //处理接收到的数据，接收1就点亮LED，接收到0就关掉LED
    Serial.println("Data received,topic: %s,data:%s\r\n",topic.c_str(),data.c_str());
    if(data=="1")
    {
      digitalWrite(LED_BUILTIN,LOW);
    }
    if(data=="0")
    {
      digitalWrite(LED_BUILTIN,HIGH);
    }
    

  });

  mqtt.onSubscribe([](int sub_id) {
    Serial.printf("Subscribe topic id: %d ok\r\n", sub_id);
    mqtt.publish("/qos1", "qos0", 0, 0);
  });
  mqtt.onConnect([]() {
    Serial.printf("MQTT: Connected\r\n");
    Serial.printf("Subscribe id: %d\r\n", mqtt.subscribe("/qos1", 0));
//    mqtt.subscribe("/qos1", 1);
//    mqtt.subscribe("/qos2", 2);
  });

  //例子中的程序给的有问题，后面调试发现会执行onDisconnect方法，而这个方法没有定义，所以一直报错，后面发现github上有人提过issue
  mqtt.onDisconnect([}(){
    Serial.println("MQTT:Disconnect");
  }])

  mqtt.begin("mqtt://192.168.1.110:1883");//这里将ip换成启动emqtt服务的电脑ip地址
//  mqtt.begin("mqtt://test.mosquitto.org:1883", {.lwtTopic = "hello", .lwtMsg = "offline", .lwtQos = 0, .lwtRetain = 0});
//  mqtt.begin("mqtt://user:pass@mosquito.org:1883");
//  mqtt.begin("mqtt://user:pass@mosquito.org:1883#clientId");

}

void loop() {
  mqtt.handle();
}
```  

然后通过刚刚上一步写的控制台应用就可以控制这个LED灯了。
发送0，LED灯灭，发送1，LED亮。

## 配置OpenHAB并实现远端控制
### 配置OpenHAB
这里的OpenHAB配置基本上是对着参考资料2中的教程做的，但是那里面item和sitemap的**双引号**是中文格式的而不是英文的，弄的卡住了很久一直出不来。所以，一定要再三确认一下用的是英文的输入法。
对着教程新建一个item和sitemap，来控制刚刚写在程序里的LED灯。
这里又有神奇的事情了，可以直接在VSCode里安装openhab的插件，没错，[OpenHAB居然还在VSCode上开发了插件](https://marketplace.visualstudio.com/items?itemName=openhab.openhab)，真是神奇的一件事情，当然，这也是我折腾完后看OpenHAB的github主页才发现的。
![openhab-sitemap](https://raw.githubusercontent.com/openhab/openhab-vscode/master/images/openhab-sitemap.gif)  
可以直接在右边窗口预览新建的sitemap

然后配置完后的演示效果如下：  
![demo](https://i.loli.net/2018/01/15/5a5c8130132c8.gif)  
但是只能在本地网络中访问，后面我们继续配置成我们想要的远程访问。



这里说一下啊，OpenHAB是不提供外网直接访问的，甚至为了安全可以设置成本机访问的，因为OpenHAB默认的管理页面是不需要认证的（估计就是故意设计成这样的，折腾了半天外网就是访问不了，后面看文档才知道），如果需要通过互联网来控制，需要安装OpenHAB的插件——OpenHAB Cloud connecter，并将OpenHAB托管在OpenHAB Cloud服务器上才行，或者通过其他的方法架设vpn或者反向代理，这里选择安装connecter插件并将OpenHAB托管在[MyOpenHAB](https://myopenhab.org/)（官方的一个托管，但是一个帐号只能绑定一个站点）上，通过Myopenhab来实现远程访问。
### 配置openHAB Cloud Connector
这里基本上照翻[官方文档](https://docs.openhab.org/addons/ios/openhabcloud/readme.html)的内容了。英文好的同学就直接看英文文档把。
首先，到**add-on**找到openHAB Cloud Connector，安装，然后就能在**Configuration——Service**里看到Connector的设置了。
![9、cloudconnector插件](https://i.loli.net/2018/01/15/5a5c7ce82d97e.png)
安装插件

![9、cloudconnector插件设置](https://i.loli.net/2018/01/15/5a5c7cfb71c16.png)
Mode选**Notifications&Remote Access**，意思是“消息提醒和远程控制”
Base URL填写```https://myopenhab.org/```当然如果不嫌麻烦自己架设了OpenHAB Cloud服务器的这里填写自己架设的。
右边就是选择哪些设备可以被远程访问并控制的，刚刚添加的就只有那一个，直接勾选。

然后就是注册[MyOpenHAB](https://myopenhab.org/)的帐号了。![myopenhabsetting](https://i.loli.net/2018/01/15/5a5c7e71bf7b1.png)  
红框中要填的**UUID**和**Secret**分别在OpenHAB安装目录下的```openhab-2.2.0\userdata\uuid```和```openhab-2.2.0\userdata\openhabcloud\secret```文件中，用记事本打开复制并粘贴到这两个空格完成注册即可。

至此，完成了远程控制开发板上的LED的目的。
什么？折腾这么久就为了控制这个小灯泡？ (╯‵□′)╯︵┻━┻   
没错，灯泡都能控制了，其他东西还会远么？(ﾉ･ω･)ﾉﾞ  

后续：
发现OpenHAB直接有一个叫HABmin的UI。。。难道可以直接配置Item和Sitemap了？果断装上！
![HABmin](https://i.loli.net/2018/01/15/5a5c842752ebb.png)  
结果等了一圈一圈又一圈后。。。

结果还是不能配置。。只能看不能修改和添加。。。
![HABmin](https://i.loli.net/2018/01/15/5a5c85fb167dd.gif)  
∑(っ °Д °;)っ

# 参考资料：

1、[nodemcu实现一个PC的远程开关](https://www.plotcup.com/2017/06/06/nodemcu-3-pc-switch/)    

2、[自己做一个智能家居系统：NodeMCU+MQTT+OpenHAB改造灯开关篇](http://augix.me/archives/5804)  

3、[M2MqttDotnetCore](https://github.com/mohaqeq/paho.mqtt.m2mqtt)  

4、[MQTT Client library for ESP8266 Arduino](https://github.com/tuanpmt/ESP8266MQTTClient)  

5、[emqtt](http://emqtt.com/)  

6、[几个物联网Iot平台的对比Blynk、微信、OpenHAB](http://www.iot-online.com/IC/embedded/2017/102477651.html)
