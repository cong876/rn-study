## React-native 使用native第三方sdk

#### ios(以阿里百川用户反馈为例)

1. 首先安装cocopods(类似于npm，ios开发的依赖管理工具，教程：http://www.code4app.com/article/cocoapods-install-usage)

2. 在ios根目录下创建Podfile文件，添加如下代码(使用的是百川feedback1.1.1版本)，然后执行pod install命令

    target ‘Xss’ do 
     pod 'YWFeedbackFMWK', '~> 1.1.1'
  end
  
3. pod安装完成后，使用xcode打开Xss.xcworkspace(我的项目名是Xss)，在项目中创建BCBridge.h以及BCBridge.m文件，以建立js和原生的bridge，.h文件只是个头文件，.m文件代码如下
在这里简要介绍下ios下的controllerView切换机制，controllerView 切换主要有两种，push和present，其中，push必须在同一UINavigationController发生，push的动画表现为横向切入，present的动画为底部向上切入(类似于弹窗)，由于react-native本身处于一个UINavigationController中，然后我目前还没有找到能向这个UINavigationController中push的方法，所以这里采用的是present的方式。
由于这种controller切换在oc里限制比较多，且使用别人的viewController可自定义的部分太受限，所以不是很推荐这种方式。

```objectivec

#import "RCTBridge.h"
#import "RCTEventDispatcher.h"
#import "RCTRootView.h"
#import "BCBridge.h"
#import "YWFeedbackFMWK/YWFeedbackKit.h"

static YWFeedbackKit *feedbackKit;    // 声明一个阿里百川feedback对象

@interface BCBridge ()
@property (nonatomic, strong) UINavigationController *navi;

@end

@implementation BCBridge

+(void)initialize {
  // 使用在百川后台申请的appkey来初始化feedbackKit
  feedbackKit = [[YWFeedbackKit alloc] initWithAppKey: @"yourappkey"];
}

// 建立Bridge，在js中直接使用
RCT_EXPORT_MODULE(BCBridge);

// 在js中调用时函数名为BCFeedback
RCT_EXPORT_METHOD(BCFeedback: (NSDictionary *)style) {
  // 自定义的样式注入，style变量为NSDictionary类型的，有js方法调用时传入，js中表现为Object
  feedbackKit.customUIPlist = style;
  // 将present操作提升到主进程来做(这里我也不太懂oc)，这里百川1.0的feedback必须这样做才能切换过去，2.0不存在这个问题
  dispatch_async(dispatch_get_main_queue(), ^{
    // 调用阿里百川提供的初始化方法，此方法接受一个回调函数，默认参数为初始化后的viewController
    [feedbackKit makeFeedbackViewControllerWithCompletionBlock:^(YWLightFeedbackViewController *viewController, NSError *error) {
      // 创建一个新的UINavigationController以阿里百川返回的viewController为RootViewController
      UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:viewController];
      // 将此controller设为当前域，可以退出
      self.navi = nav;
      // 设置title
      viewController.title = @"意见反馈";
      // 设置关闭按钮
      viewController.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] initWithTitle:@"关闭" style:UIBarButtonItemStylePlain target:self action:@selector(back)];
      // 执行present操作(此view将从屏幕下方向上切入)
      [[UIApplication sharedApplication].delegate.window.rootViewController presentViewController:nav animated:YES completion:nil];
    }];
  });
}

// 声明退出函数
- (void)back
{
  [self.navi dismissViewControllerAnimated:YES completion:nil];
}
@end

```

js中调用

``` javascript
import {
  NativeModules
} from 'react-native'

NativeModules.BCBridge.BCFeedback(options)

```

4. 至此，封装完毕，但是这种方式并不友好，而且也不符合react-native统一ui的思想，所以建议使用此种方式来封装第三方sdk的方法(获取数据)，然后使用react-native实现一套统一的ui(既可用于android也可用于ios)。但阿里百川并没有提供直接获取数据的方法，所以选择sdk时一定要慎重。


#### android

1. 依据官方文档下载对应版本的sdk(这里使用的是1.1.3版本的)
2. 在app下建立libs文件夹(如果没有的话)，将sdk中文件放进去，将项目根目录下的build.gradle文件对应位置添加如下语句

``` java
allprojects {
    repositories {
      ...
        flatDir {
            dirs 'libs'
        }
        ...
    }
}
```
在app目录下的build.gradle文件对应位置添加如下语句
有个大坑是因为阿里百川feedbackSdk默认使用multidex模式编译，如果不在项目中做对应设置，会导致一直编译不通过，看了无数种解决办法才解决此问题，泪崩~~~~

``` java
    defaultConfig {
    ...
        multiDexEnabled true                         // 开启multidex模式编译，此处为大坑，否则编译不过
    }
dependencies {
  ...
    compile 'com.android.support:multidex:1.0.0'     // 此依赖用于开启mulidex模式编译
    compile(name: 'feedbackSdk', ext: 'aar')
    compile files('libs/securityguard-3.1.27.jar')
    compile files('libs/utdid4all-1.0.4.jar')
    compile files('libs/alisdk-ut-5.jar')
}
```
3. 初始化
在MainActivity类中的onCreate方法中添加如下语句(如果FeedbackAPI无法引入，说明sdk依赖为添加成功，请检查上一步)
```java
        MultiDex.install(this);      // 同样是开启multidex模式编译，网上大部分解决方案都没提这个设置，泪崩~~~~
        FeedbackAPI.initAnnoy(this.getApplication(), "yourappkey");    // 初始化阿里百川意见反馈
```
4. 封装activity切换方法
创建BCBridge类(注意引入对应依赖)
具体代码如下
``` java
public class BCBridge extends ReactContextBaseJavaModule {

    public BCBridge(final ReactApplicationContext reactContext) {
        super(reactContext);
    }

    @Override
    public String getName() {
      // 设置在js中调用的类名
        return "BCBridge";
    }

  // 在js中调用的方法名同样为BCFeedback，readableMap对应js中的Object
    @ReactMethod
    public void BCFeedback(ReadableMap map) {
        ReadableNativeMap middleMap = (ReadableNativeMap) map;
        // 将ReadableMap转化为hashMap
        Map nativeMap = middleMap.toHashMap();
        // 设置部分ui样式
        FeedbackAPI. setUICustomInfo(nativeMap);
        // 切换到阿里百川反馈界面
        FeedbackAPI.openFeedbackActivity(getReactApplicationContext());
    }
}
```
5. 建立BCBridgePackage
  将上一步封装的方法集成到应用中(我是这样理解的)
  ``` java
  public class BCBridgePackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        return Arrays.<NativeModule>asList(
                new BCBridge(reactContext)
        );
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}
  ```
同时在MainApplication中对应位置添加如下代码(如果引用一些别人封装好的rn-原生组件，通过rn link 也能实现此操作，但是手动更改此文件时可能会导致一些情况下rn link失效，请注意检查)
``` java
    @Override
    protected List<ReactPackage> getPackages() {
      return Arrays.<ReactPackage>asList(
          ...
          new BCBridgePackage()
      );
    }
```

6. 对比于oc，java的代码好理解些，但是使用android的activity同样会有ios中提到的问题。


#### 总结(个人心得)
由于上面提到的封装原生的页面(ios中体现为viewController，android中体现为activity)，所以不提倡直接去使用别人集成好的viewController和activity，比较提倡使用这类方式来集成原生中的方法或者是组件，然后用rn来实现整体的ui布局，这样在开发成本上以及性能上都能得到很大的提高。
