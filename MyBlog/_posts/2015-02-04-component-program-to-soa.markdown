---
layout: post
title:  "从组件化到SOA"
date:   2015-02-04 15:48:17
categories: jekyll update
---

说到组件化编程，不能不提COM，作为WINDOWS的核心技术之一，不论是其功能的完备性，还是其复杂性，都是首屈一指的，不过本文不打算去讨论COM（其实是笔者对COM的了解也不算多啦），而是用更简单的例子来演示什么是组件化编程，以及其思想是如何影响架构设计的。

为简单启见，本文中的代码全部使用PYTHON语法。首先来看一个简单的例子：
{% highlight python %}
class Image(object):
    """A Image class , can output to a file"""
    def __init__(self):
        super(Image, self).__init__()

    def output(self, path, format):
        if format == 'jpeg':
            JpegCodec(self).output(path)
        elif format == 'png':
            PngCodec(self).output(path)
        elif format == 'raw':
            RawCodec(self).output(path)
        else:
            BitmapCodec(self).output(path)
{% endhighlight %}
相信很多人都遇到过这样的情况，显然，这不算是好的实现方式，如果要新增codec类型或者修改某个codec，不但要实现新的codec类，还要修改Image类的output方法。如代码中的有多个类似的output方法，那么代码的可维护性就可想而之了。

接下来，可以很自然的想到用工厂模式对其进行重构：
{% highlight python %}
def CodecFactory(image, format):
    """codec类工厂函数，跟据format，创建相应的实例并返回"""

    if format == 'jpeg':
        return JpegCodec(image)
    elif format == 'png':
        return PngCodec(image)
    elif format == 'raw':
        return RawCodec(image)
    else:
        return BitmapCodec(image)

class Image(object):
    """A Image class , can output to a file"""
    def __init__(self):
        super(Image, self).__init__()

    def output(self, path, format):
        CodecFactory(image, format).output(path)
{% endhighlight %}
代码的变化似乎不大，不过，至少，output方法不在需要关心我要去使用哪个codec问题，但对于CodecFactory，仍然存在维护成本的问题。接下，见证奇迹的时刻到了：
{% highlight python %}
__codec_table = {} # codec注册表

def codec_register(cls):
    """注册一个codec类"""

    global __codec_table
    __codec_table[cls.FORMAT] = cls
    return cls


def CodecFactory(image, format):
    """codec类工厂函数，从注册表查询与format对应的类，创建相应的实例并返回"""
    
    global __codec_table
    codecClass = __codec_table.get(format, BitmapCodec)
    return codecClass(image)


# Codec类的声明及注册

@codec_register 
class JpegCodec(BaseCodec):

    FORMAT = 'jpeg'


@codec_register
class PngCodec(BaseCodec):

    FORMAT = 'png'


@codec_register
class RawCodec(BaseCodec):

    FORMAT = 'raw'


class Image(object):
    """A Image class , can output to a file"""
    def __init__(self):
        super(Image, self).__init__()

    def output(self, path, format):
        CodecFactory(image, format).output(path)
{% endhighlight %}
这里用到了一个PYTHON里的语法糖，在PYTHON中被称之为修饰器，代码段：
{% highlight python %}
@codec_register
class RawCodec(BaseCodec):

    FORMAT = 'raw'
{% endhighlight %}
等效于：
{% highlight python %}
class RawCodec(BaseCodec):

    FORMAT = 'raw'

codec_register(RawCodec)
{% endhighlight %}
至此，CodecFactory的维护成本也被降到了最低，且你可以很方便的去创建新的codec或替换已有codec，并且这些变化对codec以外的代码是完全无感知的。当然这一切都有一个隐含的前提条件：XxxCodec.output方法的接口格式保持不变或向下兼容。而这一点，也是组件化编程的重要的前提条件。

上面代码已经具备了组件化编程框架的雏形，每个codec都是一个组件，codec_register就是组件注册器，CodecFactory就是组件实例生成器。只不过它是为codec专门设计的，可以再稍加改造，使其变得更加通用：
{% highlight python %}
__component_table = {} # 组件注册表

def component_register(cls):
    """注册一个Component类"""

    global __component_table
    key = (cls.Implement, cls.ComponentName)
    __component_table[key] = cls
    return cls


def ComponentFactory(iface, name, *args, **kwargs):
    """Component工厂函数
    从注册表查询与(iface, name)对应的类，
    使用args与kwargs作为参数创建相应的实例并返回"""
    
    global __component_table
    key = (iface, name)
    key_default = (iface, None)
    if key in __component_table:
        return __component_table[key](*args, **kwargs)
    elif key_default in __component_table:
        return __component_table[key_default](*args, **kwargs)
    else:
        return None


# Codec类的声明及注册

class ICodec(object):
    """Python本身没有接口的语法定义，
    这里通过抛出NotImplementedError来模拟接口的特性"""
    
    Implement = ICodec

    def output(self, path):

        raise NotImplementedError()


@codec_register 
class JpegCodec(ICodec):

    ComponentName = 'jpeg'

    def output(self, path):
        pass


@codec_register
class PngCodec(ICodec):

    ComponentName = 'png'

    def output(self, path):
        pass


@codec_register
class RawCodec(ICodec):

    ComponentName = 'raw'

    def output(self, path):
        pass

@codec_register
class BitmapCodec(ICodec):

    ComponentName = None # ICodec接口的默认实现

    def output(self, path):
        pass


class Image(object):
    """A Image class , can output to a file"""
    def __init__(self):
        super(Image, self).__init__()

    def output(self, path, format):
        # 调整调用的方式
        ComponentFactory(ICodec, format, self).output(path)
{% endhighlight %}
到这里，一个最简单组件化框架就完成了，当然，作为框架而言，它还很不完善，如果你希望在生产环境下使用组件化编程，建议使用zope.component和zope.interface两扩展库。

注：本文所列代码权做为示例，均没有经过实际调试。


简单总结下，面向组件的编程的思想可以用下图来描述：

![组件化]({{ site.url }}/MyBlog/_site/images/2015-02-05/a.png)

由于组件的使用者同时也可以是一个组件，所有上图也可以简为下图：

![组件化]({{ site.url }}/MyBlog/_site/images/2015-02-05/b.png)

接下来该说说SOA了，还是先上图：

![SOA]({{ site.url }}/MyBlog/_site/images/2015-02-05/c.png)

呃~~，确定不是简单的替换了下文字吗？是的，我确实只是简单的替换了下文字，当然，在实际应用中的ServiceFrame要比这个图的复杂的多，但基最本的功能无外乎服务发现与服务代理。从这个角度来看，SOA就是放大版的组件化编程，只不过SOA是从整体个架构层面来对系统进行抽象，它不关心每个服务内部如果实现，只要接口协议一致，就可以放到一起，统一接入，统一维护。

组件化编程所关注的更多的是模块内部的实现，组件的之间调用也就是通过本地的代码调用实现，虽然也有像DCOM这样的分布式组件技术，可以将组件分布不在同的机器上通过RPC进行分布式调用，但其要求所有组件必须是基于COM技术的，所以它仍然是一个同构的架构。

而SOA所关注的是整体系统的业务抽像，服务之间的调用可以看做是消息的传递，因些，只要定义好消息的协议，SOA并不关心每个服务是如何实现的，只要按照标准，实现相应的协议就好，所以SOA是一个异构的架构。

至于说如何实现一个SOA框架，本文不打算进行全面讨论了（其实是笔者之前也没做过啦，只是看别人做过啦），只是拿出其中的消息传递部分来简单讨论下，毕竟作为一个系统架构，消息传递的效率，直接影响整个系统的性能。

消息传递的方式大体上可以分为以下几种：

*   点对点
*   点对面，也就是广播
*   发布/订阅
*   消息总线

> 点对点模式，也就是最基本的一对一的socket连接模式，消息从一个模块发送到到别一个模块，这是我们最常见的情况

> 点对面模式，也就是广播模式，消息从一个模块发出，所有与之相关的模块都会接收到消息，典型的应用场景，集群中的数据同步。

> 发布/订阅，可以看做是广播模式的改良。在广播模式中，发送方往往要维护静态的接收方的列表，如果接收方发生变化，则无法做到即时响应。在发布/订阅模下，发送方不需要维护静态的接收方列表，而由接收方动态发起订阅请求，并通过心跳维持活动状态，发送方可以动态的获取到当前处于活动状态的接收方，并可以有选择的进行消息的广播。典型的应用，移动应用中的消息推送。

> 消息总线，这个有点类似路由机制，发送方把消息丢给总线模块，由总线寻找合适的接收方，并所消息转发过去。由于大部分的消息总线都是用消息队列来实现，这也带来一个附加的好处，就是可以很方便做负载均衡和负载缓冲，犹其由于大量的提交请求，更容易保证提交的时序性。

