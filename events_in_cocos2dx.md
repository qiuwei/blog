Cocos2d-x中的事件调用方式汇总

从 AS3 转到 Cocos2d-x 后，最纠结的一点就是，事件怎么调？

在 AS3 中，只要继承一下 EventDispatcher ，就能发事件了。而 C++ 没有这些。所有的轮子，都要自己造。

好在 Cocos2d-x 内部已经造好了一些轮子供我们使用。这些轮子分别是：

1. 回调函数
2. CCNotificationCenter
3. Signals
<!--more-->

**本文基于 cocos2d-x 2.1.5**

## 1. Cocos2d-x 中的回调函数

Cocos2d-x 内部大量使用回调函数来进行消息传递（或者说事件调用）。 例如 CCMenu 的事件触发，CCAction 中的结束回调等等。
 
具体实现在 `cocos2dx/cocoa/CCObject.h` 中，这里包含了菜单、Action和shedule的回调。

<pre lang="CPP">
typedef void (CCObject::*SEL_SCHEDULE)(float);
typedef void (CCObject::*SEL_CallFunc)();
typedef void (CCObject::*SEL_CallFuncN)(CCNode*);
typedef void (CCObject::*SEL_CallFuncND)(CCNode*, void*);
typedef void (CCObject::*SEL_CallFuncO)(CCObject*);
typedef void (CCObject::*SEL_MenuHandler)(CCObject*);
typedef void (CCObject::*SEL_EventHandler)(CCEvent*);
typedef int (CCObject::*SEL_Compare)(CCObject*);

#define schedule_selector(_SELECTOR) (SEL_SCHEDULE)(&_SELECTOR)
#define callfunc_selector(_SELECTOR) (SEL_CallFunc)(&_SELECTOR)
#define callfuncN_selector(_SELECTOR) (SEL_CallFuncN)(&_SELECTOR)
#define callfuncND_selector(_SELECTOR) (SEL_CallFuncND)(&_SELECTOR)
#define callfuncO_selector(_SELECTOR) (SEL_CallFuncO)(&_SELECTOR)
#define menu_selector(_SELECTOR) (SEL_MenuHandler)(&_SELECTOR)
#define event_selector(_SELECTOR) (SEL_EventHandler)(&_SELECTOR)
#define compare_selector(_SELECTOR) (SEL_Compare)(&_SELECTOR) 
</pre>
 
以菜单组件常用的 menu_selector 来分析。

首先，使用typedef定义了一个成员函数指针 SEL_MenuHandler。

<pre lang="CPP">
typedef void (CCObject::*SEL_MenuHandler)(CCObject*);
</pre>

SEL_MenuHandler 是 CCObject 的成员，接收一个 CCObject 指针形参。

用 C++11 提供的方式，也可以这样写：

<pre lang="CPP">
using SEL_MenuHandler = void (CCObject::*)(CCObject*);
</pre>
 
接着，定义进行这种类型转换的宏。

<pre lang="CPP">
#define menu_selector(_SELECTOR) (SEL_MenuHandler)(&_SELECTOR)
</pre>

这个宏将使用 menu_selector 封装的代码，转换成一个 SEL_MenuHandler 函数指针的定义。(SEL_MenuHandler) 的作用是进行类型强制转换。

让我们看看具体的使用代码，位于 HelloCpp 项目的 ActionTest.cpp 中：

<pre lang="CPP">
CCMenuItemImage *item1 = CCMenuItemImage::create(s_pPathB1, s_pPathB2, this, menu_selector(ActionsDemo::backCallback) );
</pre>

在这句代码中，将 ActionDemo::backCallback 这个函数作为指针传递进入 CCMenuItemImage 中。

CCMenuItemImage 在 initWithTarget 方法中将 ActionDemo 的实例 this， 以及 this 中的 backCallback 函数保存为 m_pListener 和 m_pfnSelector 。

<pre lang="CPP">
bool CCMenuItem::initWithTarget(CCObject *rec, SEL_MenuHandler selector)
{
    setAnchorPoint(ccp(0.5f, 0.5f));
    m_pListener = rec;
    m_pfnSelector = selector;
    m_bEnabled = true;
    m_bSelected = false;
    return true;
}
</pre>

在 CCMenuItemImage 的 activate 方法中，对这个函数指针进行了调用。

<pre lang="CPP">
void CCMenuItem::activate()
{
    if (m_bEnabled)
    {
        if (m_pListener && m_pfnSelector)
        {
            (m_pListener->*m_pfnSelector)(this);
        }
    }
}
</pre>

若希望对上面函数指针的内容做进一步的了解，可以查看 《C++ Primer中文版（第5版）》 **6.7 函数指针** 和 **19.4.2 成员函数指针** 。

## 2.  CCNotificationCenter

CCNotificationCenter 在 cocos2d-x 内部提供了一套观察者模式的实现。

下面是注册观察者的代码。注意这里依然用到了上面提到的函数指针的方法，使用的是 `callfuncO_selector` 这个宏。最后一个参数用于保存需要的数据到观察者中，之后可以使用 `CCNotificationObserver::getObject()` 来获取到这个数据。

<pre lang="CPP">
//定义事件
#define CLICK_EVENT "clickEvent"
//注册观察者
CCNotificationCenter::sharedNotificationCenter()->addObserver(this, callfuncO_selector(NotifTestScene::onClick), CLICK_EVENT, NULL);
//收到事件之后要移除观察者以避免内存泄露
void NotifTestScene::onClick(CCObject* __obj)
{
	CCMessageBox(static_cast<CCString*>(__obj)->getCString(), "onClick");
	CCNotificationCenter::sharedNotificationCenter()->removeObserver(this, CLICK_EVENT);
}
</pre>

下面是发送事件的代码。发送事件的同时可以传递一个 CCObject 指针作为数据。

<pre lang="CPP">
CCNotificationCenter::sharedNotificationCenter()->postNotification(CLICK_EVENT, &CCString("Hello World"));
</pre>

CCNotificationCenter 源码位于 `cocos2dx/support` 目录中。

## 3. Signals

我在 [获取CCArmature动画的播放状态][1] 一文中对 Signals 做了介绍。

Signals 并非是 cocos2d-x 内部通信的常用方式，Signals 也并不是 cocos2d-x 核心代码的一部分。

[1]: http://zengrong.net/post/1949.htm

