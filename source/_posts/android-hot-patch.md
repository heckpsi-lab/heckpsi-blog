---
title: 也谈 Android 应用的热更新
date: 2016-02-06 00:29:00
dsc: 我正在考虑我们是不是有一个其它的更加通用的路径在 Java 上来实现动态的修改方法从而进行热更新。
tags: [Android,JavaScript,架构]
---

## 起因

这几天支付宝的所谓「隐私门」的火热程度非同一般。至于支付宝有没有监控用户的隐私数据，双方至今也没有争论出一个结果出来。但我倒是被阿里官方所发的声明所惊讶到了，它们在声明中提到，他们可以在云端开关你手上支付宝的功能，这一点是违背对 Java 作为静态语言的常识的。而网友更是发现，这一点并不是用 API 来简单控制的，而是可以从云端下发代码下来来实现「热更新」，随时修改 apk 的本身。稍加研究，就发现了手机淘宝团队曾经在 QCon 上做过有关于相关主题的 [演讲](http://www.infoq.com/cn/presentations/mobile-phone-taobao-hotpatch-technology-introduction) 。

手机淘宝的热更新技术基于他们所开发的 Dexposed 框架，而 Dexposed 框架其实就是一个应用内的 Xposed 框架，而手机淘宝团队也大方承认了他们的许多代码也是直接从 Xposed 的开源项目里拿的。通过这个技术，框架对应用内的类都实现了 hook，可以通过云端下发的 dex 对这些类进行动态的修改，并且只损失相当较小的性能。而这个技术脱离了传统想要动态修改代码必须把整个开发框架都更换为运行于脚本语言之上的尴尬。

然而当我细究这个框架的时候发现了一些问题。阿里虽然对这一框架进行了 [开源](http://github.com/alibaba/dexposed) 。但已经很久没有更新过新版本了。当前的版本只支持到了 Android 4.4。由于 5.0 起新的 ART 虚拟机、更严格的 SELinux 策略以及对 64 位的支持之类的事，使得 Xposed 都在开发上做了很多调整。我不知道 Dexposed 现在是否支持，但至少阿里没有开源。

考虑到这些情况，我正在考虑我们是不是有一个其它的更加通用的路径在 Java 上来实现动态的修改方法从而进行热更新。

<!--more-->

## 警告

**在本地动态执行远端下发的代码是极度危险的行为。利用此方法执行非法代码等或用于绕过 Google Play 等市场的审查是违反相关协议的，也是对用户极度不负责任的行为。**

## ART 虚拟机带来的挑战

Xposed 面对 ART 虚拟机的时候究竟是遇到了什么样的问题呢？这要从 ART 虚拟机的原理说起。Java 是一门编译成 ByteCode，并由本地的虚拟机进一步动态地解释成机器码的语言。在旧版本的 Android 上所使用的 Dalvik 虚拟机其运行原理与 Oracle 的官方虚拟机是非常接近的。这样的解释对性能的消耗，虽然有 JIT 对其运行的优化，比起像 Objective-C 这样的纯编译语言来说还是差上一些性能的。

为了解决性能上的颓势，Android 在 4.4 版本上首次引入了默认不开启的测试版本的 ART 虚拟机，并在 Android 5.0 上成为了默认的虚拟机。其重要的变化是在 apk 的安装过程中，进行所谓 AOT(Ahead-of-Time) 的优化。即在安装过程中尽可能地将 ByteCode 静态编译了，并进行代码优化。这使得运行时注入的难度更高了。

Xposed 框架通过修改了 libart.so 等相关虚拟机文件关闭了一系列的优化，才使得框架终在 Android 5.0 上运行。而相对的，Dexposed 并不能修改系统虚拟机文件，毕竟这只是应用内的框架，这就使得难度变得很高。代码一旦被静态编译甚至被优化后再做 Java 层面上的 hook 确实难度很高。那么我们能不能在 Java 本身上找到一种被语言本身所支持的 hook 方式以使得更好的兼容呢？

## Java 的馈赠

显然，Java 是没有 eval() 函数的，也就是没有语言本身所支持的可以动态运行 Java 代码的方法。但当我在 Java 的文档里搜索 eval 的时候，我还真发现了一些什么。自 JDK 1.6 起，Java 内置了一个执行脚本的包 javax.script。目前支持的语言只有 JavaScript。也就是说 Java 内置了一个「动态语言」 JavaScript 的解释器！

等一下，我并不是希望你用 JavaScript 来写 Android 应用，因为这毕竟不那么快，我们之所以写原生应用，性能是我们考虑的一大原因。但是如果我们仅在热更新时「临时地」插入一段 JavaScript 代码也并不是一件坏事。但这样的话就存在一个问题，那就是 Java 的变量和 JavaScript 变量如何绑定的问题了。没关系，ScriptEngine 早已实现了这一功能。你可以通过 ScriptEngine.put(String, Object) 的方法在运行前写入 JavaScript 的变量，在运行后通过 ScriptEngine.get(String) 的方法来获得变量。

示例的 Java 代码：

``` java
try {
  ScriptEngine engine = new ScriptEngineManager().getEngineByName("javascript");
  engine.put("a", 1);
  engine.put("b", 2);
  engine.eval("var ans_1 = a + b; var ans_2 = a - b;");
  System.out.println(engine.get("ans_1"));
  System.out.println(engine.get("ans_2"));
} catch (Exception e){
  e.printStackTrace();
}
```

打印如下：

``` 
3.0
-1.0
```

## 利弊

这样做的好处是很显然的，这是 Java 语言层的支持，兼容性好得惊人。而且，这不会影响到 ART 虚拟机的 AOT 优化，你的代码依然可以在 Android 5.0+ 上和原来一样快。

但弊端也是很明显的。插入的代码由于是脚本语言，初始化脚本语言引擎和变量传递都是一个比较消耗性能的东西。但是你大可只在热更新的地方才初始化脚本语言引擎，在没有被更新到的地方依然正常地运行下去。不过在一些访问非常密集的地方使用热更新可能会对效率产生相对比较大的影响，应该避免使用。

## 示例封装

说完这些，我们可以对 Java 的 ScriptEngine 进行一些封装成为一个 HotPatch 类使得它更适合做热更新的工作。

Hot Patch 需要注入的地方分为三个类型

1. 入口 Activity
2. 类方法

### 入口 Activity

对于入口 Activity，我们希望它能发送一个异步的网络请求检测是否有新的热补丁，如果有，那么下载。下载后将对应的需要 hook 的地方的名称和对应的代码以 key-value 的形式保存下来就行。出于方便，我们可以直接使用 Android 内置的 `SharedPreferences` 来存储。这样我们只需要在 Activity 的 `onCreate` 中通过一个 Annotation 来插入。

``` java
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  @HotPatchUpdate("https://www.hotpatch.com/getUpdate", this);
}
```

需要注意的是，这个 Update 的实现需要有几个注意点。这个管道一定要建立在 https 上，因为下发代码是极其危险的，如果被劫持，后果是无法想象的。其次请求时最好自动带上 Android 版本、手机型号、地区、版本号等信息，以方便更精确地下发，千万不能下发错。然后每次执行到这里时应先检查是否有已经下载好的补丁。如果有，请校验其是否合法，以避免用户升级后，相关的补丁已经过期却仍然被运行导致更大的问题。

### 类方法

类方法里主要有两处做 hook 是最为有效的，一个是方法的一开始，而另一处是 return 处。只要将方法的 i/o hook 了，就能解决所有问题。

所以我们分别在这两处插入两个 Annotation 的方法，在尽可能不影响优雅度地情况下插入代码。

``` java
public class Test{
  public android.content.Context context;
  public Test (android.content.Context context){this.context = context;}
  public int add(int a, int b){
    @HotPatchHook("addStart", context, a, b)
    int c = a + b;
    @HotPatchHook("addEnd", context, c)
    return c;
  }
}
```

其中 `"addStart"` 和 `"addEnd"` 分别是两个索引，分别对应去寻找热补丁中对应的代码。而 context 必须传入是因为要找 `SharedPreferences`，如果你通过其它方式来实现存储的话，则不需要。`a` 和 `b` 还有 `c` 是被 Hook 的变量。有一点特别注意，当参数被 JavaScript 引擎传回时请校验其是否是 null。至于如果一个类型在 JavaScript 中找不到的话，Java 中的 ScriptEngine 所支持的 JavaScript 是可以插入 Java 的 import 包的，所以可以避开这样的问题。形如  `String jsCode = "importPackage(java.util);var list2 = Arrays.asList(['A', 'B', 'C']); "; `  的代码是可以被解释的。当然你也可以只传入 `int`、`double`、`boolean` 类型，然后手动写 `set` 和 `get` 方法。这取决于你自己想要什么样的代码风格。

## 总结

至此，我们通过实现两个简单的 Annotation 的方法，实现在 Android 上 Java 语言层面所支持的热更新的方法。当然，这样的实现并不是太过完善，还需要做很多细节上的调整。但是对于简单的热更新来说已经是足够好用了，实现一遍的话所需要的代码数也很少。同时，稍作修改，我们可以将这样的代码运行在任何 Java 程序上，实现 Java 客户端的热更新，而不只局限于 Android。

这样的热更新虽然会带来少许的性能问题，但比起将整个程序都跑在脚本上，这样的解决方案已经好上很多，更重要的是，当你没有热更新时，并不会对性能造成影响。同时，其 Java 语言层面的支持，也使得其兼容性非常良好，可以作为对于作为线上、线下调试的比较好的工具。

相比 Dexposed，这样的方法还显得比较低级，也无法 hook 系统类，没有办法做更多更底层的操作，也无法支持 ndk。但一般意义上来说，这已经足够好用了。

但是，最后还是要提醒一句：

**在本地动态执行远端下发的代码是极度危险的行为。利用此方法执行非法代码等或用于绕过 Google Play 等市场的审查是违反相关协议的，也是对用户极度不负责任的行为。**

