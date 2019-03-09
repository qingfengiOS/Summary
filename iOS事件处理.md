## 一、起始阶段
1、cpu处于睡眠状态，等待事件的发生  
2、手指触摸屏幕
## 二、系统响应阶段
1、屏幕硬件感应到事件，并且将感应到的事件传递给输入输出驱动IOKit  
2、IOKit.framework封装整个触摸事件为IOHIDEvent对象  
3、IOKit.framework通过IPC将事件转发给SpringBoard.app  

此阶段是系统层的响应，系统感应到外界的输入，并将响应的输入封装成比较概括的IOHITEvent对象，然后UIKit通过IOHITEvent的类型，判断出相应事件应该由SpringBoard.app处理，通过mach port（IPC进程间通信）转发给SpringBoard.app。  

SpringBoard.app就是iOS的系统桌面，当触摸事件发生时，只有负责管桌面的SpringBoard.app才知道如何正确地相应。因为触摸发生时，有可能用户正在桌面翻页找App，也有可能正处于在微信中刷朋友圈。
## 三、桌面响应阶段
1、SpringBoard.app主线程Runloop收到IOKit.framework转发来的消息苏醒，并触发对应mach port的Source1 回调__IOHIDEvebtSystemClientQueueCallback()
2、如果SpringBoard.app监测到有APP（记录为：xxxxx.app）在前台活动，SpringBoard.app通过mach port（IPC 进程间通信）转发给xxxxx.app，如果SpringBoard.app监测到无前台APP,则SpringBoard.app进入APP内部相应的第二阶段，触发source0回调  
## 四、内部响应阶段
1、前台app主线程Runloop收到SpringBoard.app转发来的消息苏醒，触发mach port的source1回调__IOHIDEnventSystemClientQueueCallback()。
2、soure1回调内部触发source0，回调_UIApplicationHandleEventQueue()  
3、source0回调内部，封装IOHIDEvent为UIEvent  
4、source0回调内部，调用UIApplication的sendEvent:方法，将方法传递给UIWindow  
5、通过递归调用UIView层的hitTest(:with:),结合point(inside:with)找到UIEvent中每一个UITouch所属的UIView（其实是想找到离触摸事件点最近的那个UIView）这个过程是从UIView层级的最顶层往最底层递归查询，但这不是UIResponder响应链，事件响应是在UIEvent中每一个UITouch所属的UIView都确定之后方才开始。

## 五、响应过程
### 寻找最佳响应者  
这一过程主要来确定由哪个视图来首先处理 UITouch 事件。当你点击一个 view，事件传到 UIWindow 这一步之后，会去遍历 view 层级，直至找到那个合适的 view 来处理这个事件，这一过程也叫做 Hit-Testing。    

系统会根据添加 view 的前后顺序，确定 view 在 subviews 数组中的顺序。然后根据这个顺序将视图层级转化为图层树，针对这个树，使用倒着进行前序深度遍历的算法，进行遍历。

### [hitTest:withEvent:] 方法实现原理
UIWindow 拿到事件之后，会先将事件传递给图层树中距离最靠近 UIWindow 那一层最后一个 view，然后调用其 [hitTest:withEvent:]。注意这里是**先将视图传递给 view，再调用其 [hitTest:withEvent:] 方法。并遵循这样的原则：

- 如果点不在这个视图内，则去遍历其他视图。
- 如果点击在这个视图内，但是其还有自视图，那么将事件传递给自视图，并且调用自视图的 [hitTest:withEvent:].
- 如果点击在这个视图内，并且这个视图没有子视图，那么 return self，即它就是那个最合适的视图。
- 如果点击在这个视图内，并且这个视图没有子视图，但是不想作为处理事件的 view，可以 return nil，事件由父视图处理。  

但需要注意，以下三种情况UIView以及其子View的hitTest(_:with:)不会被调用，而之后响应事件是下向上传递的，这直接导致以下三种情况的UIView及其子UIView不接收任何触摸事件：

userInteractionEnabled = NO  
hidden = YES  
alpha = 0.0~0.01之间  
>UIImageView的userInteractionEnabled默认为NO,因此UIImageView以及它的子控件默认是不接收触摸事件的。 


### 事件响应链 
touch 事件处理的传递过程与 Hit-Testing 过程正好相反。Hit-Tesing 过程是从上向下（从父视图到子视图）遍历；touch 事件处理传递是从下向上（从子视图到父视图）传递。这也就是传说中的 Response Chain。最有机会处理事件的对象就是通过 Hit-Testing 找到的视图或者第一响应者，如果两者都能处理，则传递给下一个响应者，之后依次传递。

总流程图：  
![uitouchflow.png](https://upload-images.jianshu.io/upload_images/2598795-382dd4e18c4251ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
[更多请看](http://qingmo.me/2017/03/04/FlowOfUITouch/)