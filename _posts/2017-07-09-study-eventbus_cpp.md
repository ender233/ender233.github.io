---
layout: post
title: "[源码学习]eventbus-cpp学习"
date: 2017-07-09 21:43:44 +0300
categories: 消息总线 c++
---

地址：https://github.com/dpolishuk/eventbus-cpp

### 消息总线
系统中有多个对象需要交互,  通过`消息总线`进行通信, 最大程度的降低不同对象的耦合. 一个最简化的消息总线可以用`观察者`模式完成;
传统的观察者模式实现是通过继承,　即父类定义好所有消息的接口(消息的注册/分发刷新函数), 子类实现;　引入c++11后，由于同样函数签名函数调用可以封装到`std::function<>`中，因此不用强制所有使用消息的类统一接口，去除了继承, 这使得对象的通信解耦,　使消息总线的实现成为可能. 找到一个非常简单的消息总线库`eventbus-cpp`,只有不到100行，非常适合对消息总线的概念/观察者模式/boost::signal2的学习. 并且可以在其基础上添钻加瓦.

### 消息总线三要素
无论何种实现，消息总线都要实现如下三种接口,　在读`eventbus-cpp`的时候首先就看如何给出这三种接口.
* 通用消息的定义
* 消息的注册
* 消息的分发

### boost::signal2
根据`README`文档可以看到这个库依赖于[boost::signal2](http://www.boost.org/doc/libs/1_64_0/doc/html/signals2.html), 因此首先了解下这个库.
>
The Boost.Signals2 library is an implementation of a managed signals and slots system. Signals represent callbacks with multiple targets, and are also called publishers or events in similar systems. Signals are connected to some set of slots, which are callback receivers (also called event targets or subscribers), which are called when the signal is "emitted."

这是`signal2`库的简介，是一种信号槽的实现,　没玩过QT,　但是听过在QT里面比较常用的技术就是信号槽,　看描述实际上就是和事件发布/订阅者相关,　已经和消息总线的元素比较对应了(消息所有者/订阅者), 给几个基本的例子:

```
#include <iostream>
#include <boost/signals2/signal.hpp>
struct HelloWorld
{
    void operator() () const {
        std::cout<<"Hello, World" <<std::endl;
    }
};
struct HelloWorld2
{
    void operator()() const {
        std::cout<<"Hello, World2" <<std::endl;
    }   
};
int main()
{
    {   
        boost::signals2::signal<void ()> sig;
        sig.connect(1, HelloWorld());
        sig.connect(0, HelloWorld2());
        sig();
    }   
}
```

`boost::signal2`还有很多更复杂更强大的功能(比如对消息排序),但是这个最简单的例子实际上已经有了消息总线的影子，上面提到了，消息总线三要素, 这里一一对照一下:
* 消息体定义:实际上是通过函数调用(体现可能是lambda/成员函数/函数/函数指针等等),　这里的消息是　`void()`
* 消息的订阅:很明显，`sig.connect(1, HelloWorld())`, `HelloWorld`类传入了函数对象
* 消息的分发:`sig()`对触发了消息的分发,　执行后，两个函数调用分别打印各自的结果.

那么`eventbus-cpp`的实现就很有谱了,　老大难的活儿已经被`boost::signal2`接口给干了,　那么`eventbus-cpp`应该是对齐进行了封装,出并且统一了消息接口,　不至于每一个新消息类型都新定义一个signal对象.

### eventbus-cpp实现
#### 1.提供接口
先看用例, 注解直接写在注释中
```
  　// 声明消息总线对象
    event_bus bus;
    // boost::function统一消息类型
    boost::function<void(int)> f = &test;
    boost::function<void(int, int)> f2= [&](int a, int b){
    std::cout<<"a+b:"<<a+b<<std::endl;
    };  

    // 订阅消息
    bus.subscribe(f);
    bus.subscribe(f2);
    // 分发消息
    bus.post<void(int)>(1);
    bus.post<void(int, int)>(2,3);
```
#### 2.实现
由接口可见,eventbus-cpp提供的接口还是比较简单的, 用一个消息总线对象处理不同的消息类型,　那么这样可以把消息对象设置为全局有效,　供所有类对象使用(由源码可见,实际上`eventbus-cpp`可以用单例构建),因为源码比较短,　仍然以注释的形式给出各部分的解释.
```
#ifndef EVENTBUS_H_
#define EVENTBUS_H_
#include <string>
#include <vector>
#include <boost/signals2.hpp>

// 单例父类的实现, 如果想要消息总线全局可用，那么可以将event_bus类变成单例类
// 使用: event_bus * p = singleton<event_bus>::get_instance();
// 或者　event_bus * p = event_bus::get_instance();
template<class T>
class singleton {
 public:
  //　惯例
  template<typename ... Args>
  static T* get_instance(Args ... args) {
    if (!m_instance) {
      m_instance = new T(std::forward<Args>(args)...);
    }

    return m_instance;
  }

  static
  void destroy_instance() {
    delete m_instance;
    m_instance = nullptr;
  }

 private:
  static T* m_instance;
};
template<class T> T* singleton<T>::m_instance = nullptr;

//　继承单例类
class event_bus : public singleton<event_bus> {
 public:
  event_bus() {}
  virtual ~event_bus() {}

  // 新创建signal实例
  template<typename T>
  boost::signals2::signal<T>* create_signal() {
    typedef boost::signals2::signal<T> signal_t;

    if (m_signals.find(typeid(T).name()) == m_signals.end()) {
      signal_t* signal = new signal_t();
      m_signals[typeid(T).name()] = signal;
      return (signal);
    }

    return 0;
  }

  //　消息总线提供订阅接口
  template<typename T>
  boost::signals2::connection subscribe(const boost::function<T>& callback) {
    typedef boost::signals2::signal<T> signal_t;
    signal_t* signal = nullptr;

    // 函数签名一致的声明一个　boost::signal2::signal<T>对象,　key值通过typeid(T).name()界定
    //　因此,　首先看实例是否已经生成，如果没有则新创建; 否则,转成基类提取调用connect
    if (m_signals.find(typeid(T).name()) == m_signals.end()) {
      signal = create_signal<T>();
    } else {
      signal = dynamic_cast<signal_t*>(m_signals[typeid(T).name()]);
    }
    boost::signals2::connection ret = signal->connect(callback);
    return (ret);
  }

  // 消息总线消息分发接口
  template<typename T, typename ... Args>
  void post(Args ... args) {
    typedef boost::signals2::signal<T> signal_t;
    if (m_signals.find(typeid(T).name()) != m_signals.end()) {
      signal_t* signal = dynamic_cast<signal_t*>(m_signals[typeid(T).name()]);
      signal->operator()(std::forward<Args>(args)...);
    }
  }

 private:
  // boost::signals2::signal<T> *存储于boost::signal2::signal_base*中
  // 所有同函数标签的消息体存储在一个pair中.利用函数签名类型的名字作为key值
  std::map<std::string, boost::signals2::signal_base*> m_signals;
};
```

### 待优化
* 没有提供去订阅的接口
* 订阅接口仍然不够简洁,　仍需要人工给`std::function<>`指定类型,　可以借助于`function_traits`简化接口,　自动抽取类型

