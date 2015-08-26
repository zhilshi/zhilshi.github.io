---
layout: post
title:  "视图控制器跳转"
date:   2015-08-26 09:43:00
categories: iOS
---

最近在项目中，遇到导航跳转相关的各种复杂逻辑。下面介绍一些个人在开发中的一些思路，希望有所帮助。

###获取当前的视图控制器
大多时候，我们需要知道当前的视图控制器是啥。通常在点击推送消息跳转、session异常退出登录等情况下，需要知道当前在哪一个视图控制器上，才可以进行线框跳转。那么，如何获取呢？第一反应，这不是很简单？直接从rootViewController找起来，找到topViewController.
比如:
{% highlight objective-c linenos %}
+ (UIViewController *)currentTopViewController
{
    //获取根视图控制器
    UIViewController *rootViewController = [UIApplication sharedApplication].keyWindow.rootViewController;
    //tabbar
    if ([rootViewController isKindOfClass:[UITabBarController class]])
    {
        UINavigationController *nav =  (UINavigationController *)[(UITabBarController *)rootViewController selectedViewController];        
        return nav.topViewController;
    }
    //导航条
    if ([rootViewController isKindOfClass:[UINavigationController class]])
    {
        return ((UINavigationController *)rootViewController).topViewController;
    }
    //视图控制器
    return rootViewController;
}
{% endhighlight %}	
**等等,考虑这么简单？模态呢？**,我们知道，可以连续有多个模态的导航控制器,因此大概是这样子的。
{% highlight objective-c linenos %}
//方法一 通过rootViewController 找
+ (UIViewController *)szl_currentTopViewController
{
    //获取根视图控制器
    UIViewController *rootViewController = [UIApplication sharedApplication].keyWindow.rootViewController;
    
    //tabbar
    if ([rootViewController isKindOfClass:[UITabBarController class]])
    {
        UINavigationController *selectNavController =  (UINavigationController *)[(UITabBarController *)rootViewController selectedViewController];
        return [[self class] szl_findCurrentTopViewControllerFromViewController:selectNavController];
    }
    
    return [[self class] szl_findCurrentTopViewControllerFromViewController:rootViewController];
}

// 开始查找
+ (UIViewController *)szl_findCurrentTopViewControllerFromViewController:(UIViewController *)viewController
{
    if (!viewController)
    {
        return nil;
    }
    NSLog(@"start find from : %@",viewController);
    // 查找模态
    UIViewController *topPresentedViewController = [[self class] szl_topPresentedViewControllerFromViewController:viewController];
    
    // 查找最前视图
    UIViewController *currentTopViewController = [[self class] szl_topViewControllerFromViewController:topPresentedViewController];
    
    return currentTopViewController;
}

// 从fromViewController开始查找,一直找到最后的模态视图；
// 如果无模态视图，返回fromViewController
+ (UIViewController *)szl_topPresentedViewControllerFromViewController:(UIViewController *)fromViewController
{
    NSLog(@"presented viewController : %@",fromViewController);

    // 判断是否存在模态视图, 不存在,返回自身
    if (!fromViewController.presentedViewController)
    {
        return fromViewController;
    }
    
    // 递归查找
    return [[self class] szl_topPresentedViewControllerFromViewController:fromViewController.presentedViewController];
}

//从fromViewController获取最前面的视图控制器
+ (UIViewController *)szl_topViewControllerFromViewController:(UIViewController *)fromViewController
{
    NSLog(@"fromViewController: %@",fromViewController);

    //当前模态视图是一个导航控制器
    if ([fromViewController isKindOfClass:[UINavigationController class]])
    {
        return ((UINavigationController *)fromViewController).topViewController;
    }
    
    //视图控制器
    return fromViewController;
}
{% endhighlight %}	
这里的思路其实很简单，从根视图控制器开始查找,根视图控制器存在两种情况:

1. 根视图控制器是**UITabBarController**
2. 根视图控制器是**UINavigationController** 或 **UIViewController**

我们针对**UITabBarController**进行处理,通过**selectedViewController**获取当前的视图控制器。

其次,根据这时获取的视图控制器,可能是一个导航视图控制器(基本上是导航控制器),但也有可能是视图控制器。通过**szl_topPresentedViewControllerFromViewController**方法寻找最顶层的模态视图控制器。可以看到这是一个递归方法,天知道复杂的应用里会有多少层模态呢？这里注意,如果一开始就不存在模态,还是会返回出传入的对象。

最后,根据获取到的视图控制器,有可能是一个导航视图控制器,也有可能是一个视图控制器,因此通过方法**szl_topViewControllerFromViewController**获取最终的当前视图控制器。

需要注意的是,由于**presentedViewController**取值操作,因此获取方法需要在视图控制器**viewWillAppear**之后调用.
回头一看，看上去是没什么问题了?基本上确实是可以实现功能。

但是,我们看到整个方法的实现,最主要的还是基于视图控制器**presentedViewController**的属性。

那么该属性是在什么时候赋值的呢?我们基本可以断定属性赋值的时机肯定会影响到我们对当前视图控制器的判断。
我们先来看看模态方法**presentViewController:animated:completion:**的**appleAPI**说明:

`Presents a view controller modally.
In a horizontally compact environment, the presented view is always full screen. In a horizontally regular environment, the presentation depends on the value in the modalPresentationStyle property.
**This method sets the presentedViewController property to the specified view controller, resizes that view controller's view based on the presentation style and then adds the view to the view hierarchy**. The view is animated onscreen according to the transition style specified in the modalTransitionStyle property of the presented view controller.
**The completion handler is called after the viewDidAppear: method is called on the presented view controller.**
`

**presentedViewController**属性是在该方法中设置的,并且是在**viewDidAppear**之后完成的。经过测试就发现,只有在**viewDidLoad**完成之后,模态视图完成加载后,才会对指定的视图控制器设置该属性。

因此调用的时候需要注意该情况:**如果在新的视图模态加载未完成前,获取当前的视图控制器可能就会存在问题**

此外,同样的,如果该视图正在调用**dismissViewControllerAnimated:completion**方法呢？其实我们可以看下下面的几个API:
{% highlight objective-c %}
- (BOOL)isBeingPresented NS_AVAILABLE_IOS(5_0);
- (BOOL)isBeingDismissed NS_AVAILABLE_IOS(5_0);
- (BOOL)isMovingToParentViewController NS_AVAILABLE_IOS(5_0);
- (BOOL)isMovingFromParentViewController NS_AVAILABLE_IOS(5_0);
{% endhighlight %}	

对于视图的模态状态,显而易见,这几个方法是中间过渡的状态表示。比如我们可以通过判断
**([self isBeingDismissed]||[self isMovingFromParentViewController])**表达式,来确定视图是将会被dismiss掉。

因此,我们可以修改下**szl_findCurrentTopViewControllerFromViewController:**方法,如下
{% highlight objective-c %}
+ (UIViewController *)szl_findCurrentTopViewControllerFromViewController:(UIViewController *)viewController
{
    if (!viewController)
    {
        return nil;
    }
    NSLog(@"start find from : %@",viewController);
    // 查找模态
    UIViewController *topPresentedViewController = [[self class] szl_topPresentedViewControllerFromViewController:viewController];
    // 查找最前视图
    UIViewController *currentTopViewController = [[self class] szl_topViewControllerFromViewController:topPresentedViewController];
	// 当前页面正在移除的情况下 即将消失
	if ([currentTopViewController isBeingDismissed] || [currentTopViewController isMovingFromParentViewController])
	{
		UIViewController *presetingViewController = presentedVC.presentingViewController;
		if ([presetingViewController isKindOfClass:[UINavigationController class]])
		{
			 return ((UINavigationController *)presetingViewController).topViewController;
		}
		else
		{
			 return presetingViewController;
		}
	}
    return currentTopViewController;
}
{% endhighlight %}	

你看,简单的获取当前最前页,写了这么多的代码,但我们依然不能完全肯定没有遗落之处。比如session异常的通知,很有可能在任何状态下发生!那么,有没有更好的方式？

其实视图的跳转,都是在主线程上一个视图控制器被添加到当前窗口上,当前的视图,会调用**viewWillAppear**和**viewDidAppear**,那可以直接用一全局变量指定当前的视图控制器?为了避免每个类中进行调用,可以考虑使用**Method Swizzling**来获取当前的视图控制器.其次再根据模态情况下进行处理。这样避免了方法一种繁琐的递归查找,效率更高。
{% highlight objective-c %}
- (void)szl_viewDidAppear:(BOOL)animated
{
    NSString *classname = NSStringFromClass([self class]);
    // 需要过滤 筛选
    if ([classname hasPrefix:@"SZL"])
    {
        curViewController = self;
        NSLog(@"curViewController is %@",curViewController);
    }
    [self szl_viewDidAppear:animated];
}

+ (void)load
{
    SEL originalSelector  =  @selector(viewDidAppear:);
    SEL overrideSelector  =  @selector(szl_viewDidAppear:);
    Method originalMethod = class_getInstanceMethod(self, originalSelector);
    Method overrideMethod = class_getInstanceMethod(self, overrideSelector);
    if (class_addMethod(self, originalSelector, method_getImplementation(overrideMethod), method_getTypeEncoding(overrideMethod)))
    {
        class_replaceMethod(self, overrideSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }
    else
    {
        method_exchangeImplementations(originalMethod, overrideMethod);
    }
}
{% endhighlight %}	
该load实现参考stackoverflow上的实现。在交互方法中,增加了一步前缀判断。此外,对于特殊的系统视图控制器比如相册、拍照等也需要考虑进来。当然,获取到后,还是需要再根据方法一的后续判断进行处理,这就不同的业务规则了。

###跳转
关于跳转,很多时候我们需要处理复杂的页面跳转。比如session异常的情况,业务设计需要不管在任何页面,你需要先回到UITabbarViewConroller的当前页,再模态一个登录视图控制器提示用户进行登录操作。该怎么设计呢??基本上思路是这样子的:

1. 所有的模态视图控制器移除,直到selectedViewController
2. 在selectedViewController上popToRootViewController
3. 创建视图控制器,进行模态展示

第一步操作其实很简单,在apple API里对于**dismissViewControllerAnimated:completion**说明:

`if you present several view controllers in succession, thus building a stack of presented view controllers, calling this method on
 a view controller lower in the stack dismisses its immediate child view controller and all view controllers above that child on
  the stack.`
	
意识是说当存在多个模态的视图控制器构成的堆栈里,只要在这个堆栈里找到低层级的模态视图调用**dismiss**,在它之上的缘由视图控制器都会移除。
这里需要注意的是,如果使用**UITabbarViewController**,比如在它的**selectedViewController**控制器A上,模态一个视图控制器B,那么A的**presentedViewController**指向B是没有错的,但B的**presentingViewController**指向的不是A,而是**UITabbarViewController**。因此这里的方法大概是这样子的:
{% highlight objective-c %}

UITabBarController *tabbarViewController = [UIApplication sharedApplication].keyWindow.rootViewController;
UIViewController   *selectViewController = root.selectedViewController;
[tabbarViewController dismissViewControllerAnimated:YES completion:^{
       dispatch_async(dispatch_get_main_queue(), ^{
           [(UINavigationController*)selectViewController popToRootViewControllerAnimated:^(){
            [selectViewController presentViewController:[SZLAssemblingFactory YSNavigationwithLoginViewController] 
           									      animated:YES 
           									    completion:nil];
           }];          
       });
}];

{% endhighlight %}

