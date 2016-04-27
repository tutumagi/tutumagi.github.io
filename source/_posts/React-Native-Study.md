---
title: React Native 学习记录 － JS和OC通信流程
date: 2016-04-26 22:01:34
categories:
- Coding
tags:
- React Native
- iOS
---

## 什么是React Native
[React Native](https://facebook.github.io/react-native/) 是Facebook出的一套开源框架，框架的作用是开发者特别是前端开发者可以使用[JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript) + [React](http://facebook.github.io/react/)编写，但是会被“翻译”成原生代码执行。体验比纯粹的使用WebView去运行H5+JS语言更好，效率更高。  
这是[官方](https://facebook.github.io/react-native/)的介绍，可以让你构建出**世界级**的应用哦，是不是很心动？！
> React Native enables you to build world-class application experiences on native platforms using a consistent developer experience based on JavaScript and React. The focus of React Native is on developer efficiency across all the platforms you care about - **learn once, write anywhere**. Facebook uses React Native in multiple production apps and will continue investing in React Native.

## React Native给我们带来了什么
1. **热更新**
2. 开发效率的提高
3. 组件的复用
4. 跨平台!?(官方提供的ui组件中也分*Android， *IOS，查阅了一下，有人说可以做到80%到90%共用，这个应该看具体应用场景了)

## React Native现状
现有APP:  
[国内](http://reactnative.cn/cases.html)  
[国外](https://facebook.github.io/react-native/releases/next/showcase.html)  

[携程的“我”的部分模块](http://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=403412672&idx=1&sn=2cceb873ee4640830aad3261ae177df5&scene=23&srcid=0421T6qLv5d2yM24EQs2hOZN#rd)

## React Native原理
### JavaScript 和 Objective-C 通信原理

参考这两篇文章学习[React native Bridge](http://tadeuzagallo.com/blog/react-native-bridge/) 和 [JS和OC通信原理](http://blog.cnbang.net/tech/2698/)  
#### Native Module 核心代码  
```objc
//导出module符号给js使用
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }
...
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });
	
  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);
	
  // Register module
  [RCTModuleClasses addObject:moduleClass];
	
  objc_setAssociatedObject(moduleClass, &RCTBridgeModuleClassIsRegistered,
                           @NO, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```
我们在查看｀RCT_EXTERN｀宏时可以学到一个小技巧：如果我们开发组件提供给其他模块使用，当我们不想暴露方法的实现时，可以使用这种方式，在方法名前面`extern __attribute__((visibility("default")))`	，告诉编译器这个符号不要在编译时解决，而是在链接时去解决。

```objc
//导出方法符号给js使用
RCT_EXPORT_METHOD(drawSomethingWithCGRect:(id )rectId)
{
	...
}
	
...
#define RCT_EXTERN_REMAP_METHOD(js_name, method) \
  + (NSArray<NSString *> *)RCT_CONCAT(__rct_export__, \
    RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    return @[@#js_name, @#method]; \
  }
  
/**
 * 宏展开后的形式，主要是导出全局唯一的符号供js调用  
 + (NSArray<NSString *> *)__rct_export__290{
    return @[@"", @"drawSomethingWithCGRect:(id )rectId"];
 }
 - (void)drawSomethingWithCGRect:(id )rectId
 */
```

#### Runtime
程序加载时做的事情
![](http://7xsw3s.com2.z0.glb.clouddn.com/image%2Fjpg%2F1460484959926)      

1. Register All Export Modules 就是上面我们Native Code里面做的事情
2. 获取js，可能本地，可能远程；同时通过所有ExportModuleClasses去初始化RCTModuleData: 
	1. RCTBridget存储ModuleData的引用
	2. 判断ModuleClass的方法执行是否需要在主线程，如果不需要则新建一个串行队列 提供给该moduleCLass的方法执行 
	3. 判断ModuleClass是否有需要导出的常量
3. 初始化JavaScript Executor，这里如果我们不在Chrome里面调试，就是RCTJSCExecutor，否则就是RCTWebSocketExecutor。
	1. 初始化JavaScriptCore;
	2. 添加一些方法供Js调用，比如Js在调用我们的Native Code的方法时:  
	
		```objc		
		[self addSynchronousHookWithName:@"nativeFlushQueueImmediate" usingBlock:^(NSArray<NSArray *> *calls){
		    RCTJSCExecutor *strongSelf = weakSelf;
		    if (!strongSelf.valid || !calls) {
		      return;
		    }
			
		    RCT_PROFILE_BEGIN_EVENT(0, @"nativeFlushQueueImmediate", nil);
		    [strongSelf->_bridge handleBuffer:calls batchEnded:NO];
		    RCT_PROFILE_END_EVENT(0, @"js_call", nil);
		}];
		```
 	同时: ModuleClass对应的Json格式的config数据:
 	![导出的json格式](http://7xsw3s.com2.z0.glb.clouddn.com/1460904823619)

    ```json
    [
      "CalendarManager",
      {
        "firstDayOfTheWeek": "Monday"
      },
      [
        "addEvent",
        "findEvents"
      ]
    ]
    ```
4. 注入JavaScript:  

	```objc	
	- (void)injectJSONConfiguration:(NSString *)configJSON
	                     onComplete:(void (^)(NSError *))onComplete
	{
	  if (!_valid) {
	    return;
	  }
		
	  [_javaScriptExecutor injectJSONText:configJSON
	                  asGlobalObjectNamed:@"__fbBatchedBridgeConfig"
	                             callback:onComplete];
	}
	```
	就是将4拿到的`jsonConfig`赋值给`__fbBatchedBridgeConfig`，在5中执行JS会用到
5. 执行2中获取的Js

#### 通信
参考前面提到的博文（[JS和OC通信原理](http://blog.cnbang.net/tech/2698/) ），这篇介绍得很详细了。
### 渲染机制
```
JSX  ->   Virtual DOM   ->   RCTUIManager(shadow queue)  -> Native View (main queue)
```

首先`babel`会翻译`JSX`为标准的`ES5`, React的组件模型实现了一套`Virtual DOM`，会比对当前正在显示的`Virtual DOM` 和将要显示的`Virtual DOM`，做一个diff，然后将diff后的数据组装成消息结构通过`JSBridge`发送给`Native`端，这时Native在`shadow queue`(非主线程)将消息组装成一个`block`并push到队列里去，然后在一个恰当的时机flush所有的`UI block`，此时是在主线程中调用。

## 看看别人怎么说
[一个前端工程师的看法](http://div.io/topic/851?page=1#3453)  
[知乎上面的讨论](https://www.zhihu.com/question/27852694)

## 学习曲线
[JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript)  
[JSX](https://facebook.github.io/react/docs/jsx-in-depth.html)  
[React](http://facebook.github.io/react/index.html)  
[CSS-Layout](https://github.com/facebook/css-layout)  
[Flux](https://facebook.github.io/flux/docs/overview.html)  
Github上的中英文有关ReactNative的学习仓库：  
[中文整理资料](https://github.com/ele828/react-native-guide)  
[英文整理资料](https://github.com/jondot/awesome-react-native)  
前端开发者还得了解常用的iOS以及Android的View结构以及移动设备的体验习惯。


**完**
