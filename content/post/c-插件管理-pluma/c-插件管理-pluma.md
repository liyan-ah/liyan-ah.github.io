---
title: c++插件管理--pluma<实践>
author: 李岩
tags:
  - c++
  - pluma
categories:
  - code
date: 2020-10-29 21:19:00
---
最近研究了一下[pluma](http://pluma-framework.sourceforge.net/)的使用。发现官网上的简单示例对于刚入门的人来说还是麻烦了些（而且还有语法错误）。  
下面重新整理了一个例子，作为备注。  
<!--more-->

其中，<code>device</code>为一个虚基类，作为接口类存在。<code>keyboard</code>及<code>screen</code>作为实现了device的子类存在，实现具体的操作。在pluma上注册后，在<code>main</code>中调用接口，实现<code>keyboard</code>及<code>screen</code>的调用。
```
// device.hpp
#ifndef _DEVICE_HPP_
#define _DEVICE_HPP_
#include "Pluma/Pluma.hpp"

class Device{
public:
    virtual std::string getDescription() const=0;
};
// create DevicedProvider class
PLUMA_PROVIDER_HEADER(Device);

#endif
```
```
// device.cpp
#include "device.hpp"
PLUMA_PROVIDER_SOURCE(Device, 6, 3);
```
如上所示，是<code>device</code>的定义。其中<code>PLUMA_PROVIDER_HEADER</code>和<code>PLUMA_PROVIDER_SOURCE</code><code>pluma</code>提供的宏。功能暂且不论。我们继续往下看。  
```
// screen.hpp
#include "Pluma/Pluma.hpp"
#include "device.hpp"

class Screen: public Device{
public:
    std:: string getDescription() const{
        return "screen";
    }
};

PLUMA_INHERIT_PROVIDER(Screen, Device);
```
```
// keyboard.hpp
#include "Pluma/Pluma.hpp"
#include "device.hpp"

class Keyboard: public Device{
public:
    std:: string getDescription() const{
        return "keyboard";
    }
};

PLUMA_INHERIT_PROVIDER(Keyboard, Device);
```
上面实现了<code>screen</code>及<code>keyboard</code>的逻辑。实现了之后，需要进行注册：  
```
// connect.cpp
 #include <Pluma/Connector.hpp>
 #include "keyboard.hpp"
 #include "screen.hpp"

 PLUMA_CONNECTOR
 bool connect(pluma::Host& host){
     // add a keyboard provider to host
     host.add( new KeyboardProvider() );
     host.add( new ScreenProvider() );
     return true;
 }
```
这里在<code>connect</code>中进行了两个子类的注册。之所以使用<code>connect</code>是因为后面的pluma使用的时候，官网给出的示例代码中，会从<code>connect</code>入口开始调用。  
```
// main.cpp
#include "Pluma/Pluma.hpp"

#include "device.hpp"
#include <iostream>
#include <vector>

int main()
{
    pluma:: Pluma plugins;
    plugins.acceptProviderType<DeviceProvider>();
    plugins.load("./plugin/connect.so");
    //plugins.load("./plugin/keyboard.so");

    std::vector<DeviceProvider*> providers;
    plugins.getProviders(providers);

    std::cout<<"size for providers are:" << providers.size()<< std:: endl;
    if (!providers.empty()){
        for (std::vector<DeviceProvider*>::iterator device=providers.begin();
            device != providers.end(); ++ device){
            Device* myDevice = (*device)->create();
            std::cout << myDevice->getDescription() << std::endl;
            delete myDevice;
        }
    }
    return 0;
}
```
这里就是主要的调用逻辑了。官网中<code>myDevice</code>附近的拼写有主意，这是个坑了。  
这里回顾下目录结构：
```
.
├── connect.cpp
├── device.hpp
├── device.cpp
├── keyboard.hpp
├── main.cpp
├── plugin                          # 用来存储插件结果的目录
├── Pluma                           # 为了方便，这里将Pluma的include及src文件均拷贝到这里
│   ├── Config.hpp
│   ├── Connector.hpp
│   ├── Dir.cpp
│   ├── Dir.hpp
│   ├── DLibrary.cpp
│   ├── DLibrary.hpp
│   ├── Host.cpp
│   ├── Host.hpp
│   ├── PluginManager.cpp
│   ├── PluginManager.hpp
│   ├── Pluma.hpp
│   ├── Pluma.inl
│   ├── Provider.cpp
│   ├── Provider.hpp
│   └── uce-dirent.h
└── screen.hpp
```
看下编译过程：
```
# 生成device.so
g++ connect.cpp device.cpp Pluma/*.cpp -shared -fPIC -o plugin/connect.so -I./

# 生成main
g++ main.cpp device.hpp device.cpp Pluma/*.cpp -o main -I./ -ldl
```
执行：
```
./main
size for providers are:2
keyboard
screen
```
以上就是实践的内容了。
