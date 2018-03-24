### 分包的目的：
* 拆分业务代码，不同模块的业务代码应保存独立性；
* 复用框架代码，不同的业务模块可以共享框架代码；
* 在开发过程中，支持单页面(业务)调试；
* 减小首页加载时的时间损耗和性能损耗。

### 分包基本思路
* 拆分业务代码到多个bundle，各业务之间没有代码耦合，按照业务的相关性控制bundle的拆分力度；
* 通过bundleId区分业务；
* 拆分公共组件，将共用组件拆分到RNCommonKit中；
* 相同业务的页面共用一个bridge；
* 不同业务的页面使用不同的bridge。

### 对LDReactNative改造如下
###### LDRNRootViewController
包含RN页面的默认VC，通过页面所在的bundleId、页面的moduleName创建页面。每个页面所在的bundleId唯一，moduleName唯一。

* 删除原初始化方法，增加通过bundleId和moduleName初始化方法：

```
/**
 创建空白RN页面

 @param bundleId 页面所在bundle对应的bundleId
 @param moduleName 页面的moduleName
 @param initialProperties 在初始化时由native传递给RN的参数
 @return 空白RN页面
 */
- (instancetype)initWithBundleId:(NSString *)bundleId
                      moduleName:(NSString *)moduleName
               initialProperties:(NSDictionary *)initialProperties;
```

###### LDRNRootView
RCTRootView的封装，监听了bridge的reload通知，在bridge重新加载时，删除contentView，并重新创建。否则，会因为重复创建contentView导致重复注册某个module的问题。

###### LDRNRootViewCreator
提供创建rootView的类方法。

```
/**
 创建对应bundleId的rootView
 
 @param bundleId 模块的bundleId
 @param moduleName 页面的moduleName
 @param initialProperties 在初始化时由native传递给RN的参数
 @param completionHandler 回调处理
 */
+ (void)createViewWithBundleId:(NSString *)bundleId
                    moduleName:(NSString *)moduleName
             initialProperties:(NSDictionary *)initialProperties
             completionHandler:(void(^)(RCTRootView *rootView, NSError *error))completionHandler;
```

###### LDRNBridgeManager
获取bridge的单例，该单例中保存了以bundleId
为key、相应bridge为value的字典。对于处于同一个bundle的页面，公用一个bridge。另外该单例中，缓存了一个只加载了base代码的bridge，供创建新的bridge时使用。

* 提供创建/获取bridge的类方法
```
/**
 通过bundleId创建bridge

 @param bundleId 页面所在的bundleId
 @param completionHandler 回调
 */
+ (void)getBridgeWithBundleId:(NSString *)bundleId
            completionHandler:(void (^)(RCTBridge *bridge, NSError *error))completionHandler;

```

* 在创建bridge的时候，区分使用local server的调试模式，和使用离线包的线上模式。具体区别如下：

```
1、调试模式下全部代码走local server，没有缓存的bridge；
2、线上模式下，先加载base代码，再加载增量的业务代码。
```
* 缓存bridge时，为了避免占用本次bridge创建时的资源，延迟2秒缓存bridge。

整体流程图如下:

![image](https://note.youdao.com/yws/public/resource/9121d7803f4b34863ce6cbf59bdf6be0/xmlnote/WEBRESOURCE318af11a55ba32f675e72c71d8de9efe/1450)

###### LDRNBridgeWrapper

RCTBridge的封装，提供创建bridge、增量加载业务代码的方法。

* 提供创建bridge的方法。调试模式下，加载local server代码。离线包模式下，仅加载base代码，不加载业务代码

```
/**
 根据bundleId创建bridge，离线包先加载base框架，再加载增量代码；调试包直接走local server

 @param bundleId 页面所在的bundleId
 @return instancetype
 */
- (instancetype)initWithBundleId:(NSString *)bundleId;
```

* 提供加载增量代码的方法，内部设置有等待队列，增量代码等待base代码加载完成后再加载。

```
/**
 使用离线包时加载增量代码

 @param bundleId 页面所在的bundleId
 @param completionHandler 回调
 */
- (void)loadBundle:(NSString *)bundleId completionHandler:(void (^)(RCTBridge *bridge, NSError *error))completionHandler;

```

* 对外不暴露的RCTBridge的分类，主要是增加暴露了RTCBridge中处理动态加载JS代码的方法，另外对该方法通过修改返回参数统一处理了RCTBatchedBridge和RCTCxxBridge。

整体流程图如下：
![image](https://note.youdao.com/yws/public/resource/9121d7803f4b34863ce6cbf59bdf6be0/xmlnote/WEBRESOURCE4c468de8ff3f3a09e3078846c7f4a10a/1451)

###### LDRNBroadCastManager

* 提供桥接方法，供RN页面之间发送通知。支持同bundle、跨bundle页面之间的通知。参数包括：

```
bundleId(string): 通知接受页面所在包的bundleId
eventName(string): 通知名称
data(dictionary): 通知内容

RCT_EXPORT_METHOD(sendBroadCastToBundleId:(NSString *)bundleId eventName:(NSString *)eventName data:(NSDictionary *)data) {
    dispatch_async(dispatch_get_main_queue(), ^{
        [LDRNEventSender sendEventToBundleId:bundleId eventName:eventName withData:data];
    });
}
```

###### LDRNEventSender

* 通知的具体实现，调用了bridge的内部方法，向目标发送通知。支持native向RN页面发送通知。参数与LDRNBroadCastManager相同。
* ==由于使用了bridge的私有方法，将来随着RN版本的升级，需要回归验证方法是否有效。==
