# iOS仿滴滴预约用车时间选择器

## 从需求说起
前几天接到一个版本，里面包含了一个和滴滴预约用车选择时间的picker一样，需要选择当前时间的后面几天内的时间，包含了日期，小时和分钟数，分钟数的间隔是以10分钟为单位，如下图所示：   
![timerPicker](https://user-gold-cdn.xitu.io/2018/11/23/1673f5bbf8c1817a?w=856&h=528&f=png&s=92805)

当接到这个需求时，我的心里是有点小拒绝的，看着就是一个pickerView但是里面东西还是有的东西的，包含：      

1. 时间数据源获取，获取当前时间到3天后。
2. 自定义时间数据源，分钟时间刻度单位为10分钟，不足10分钟的向上取整。
3. 选择当天对当前小时数据和分钟数据的处理。
4. 选择当前小时情况下对分钟数据源的处理。
5. pickerView自定义展示（颜色，字体大小）

个人认为，能自己做的尽量都少用三方库，减少对三方库的依赖，（PS:目前项目用了百度地图，iOS12删除了百度SDK用到的系统库，各种麻烦），所以决定自己造一个轮子。

## 过程
#### 获取天数
这里采用NSDate的```dateWithTimeIntervalSinceNow```函数再转成字符串，值得一提的是```NSDateFormatter```,根据[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/DataFormatting/Articles/dfDateFormatting10_4.html)的描述：  
>Creating a date formatter is not a cheap operation. If you are likely to use a formatter frequently, it is typically more efficient to cache a single instance than to create and dispose of multiple instances. One approach is to use a static variable.

创建一个formatter实例的代价是比较高，频繁使用时要考虑缓存，个人的做法是：  

```
+ (void)load {
    if (!_dateFormatter) {
        _dateFormatter = [[NSDateFormatter alloc] init];
        [_dateFormatter setDateFormat:@"YYYY-MM-dd HH:mm"];
    }
}
```
保证只创建一个NSDateFormatter实例，关于```+load```不做多说，想了解更多的可以看看之前的[Runtime源码 +load 和 +initialize](https://juejin.im/post/5bda5a02e51d45685f4435e6)  

```
for (NSInteger i = 0; i < kDays; i++) {
        NSString *dateString = [self distanceDate:beginTime aDay:i];//获取第i天的日期
        NSString *week = [self currentWeek:dateString type:NO];//获取星期几     
}
```  
这里用到了  
```
static NSInteger const kDays = 3;//从今天起能选择多少天 默认3天
``` 
因为#define在编译的预处理阶段有一个宏替换操作，大量地使用#define会拖慢编译速度，而且宏没有类型，不做任何类型检查。Apple官方也是使用了更多的const。

### 分钟数向上取整  
需求是当分钟数不为整10分钟时，向上取整，比如,16->20,41->50,所以对初始的数据源还有一步向上取整的操作：  

```
NSString *beginTime = [self getTimerAfterCurrentTime:kBenginTimeDely];//开始时间（也就是当前时间20分钟后）
    NSInteger currentMin = [self getMString:beginTime];
    if (currentMin % kTimeInterval != 0) {
        beginTime = [self getTimerAfterTime:beginTime periodMin:(kTimeInterval - currentMin % kTimeInterval)];//开始时间向上取整
}
```   
这里把这三个数据抽取出来，提高灵活性，比如天数要5天之后or时间间隔要改成5分钟or最早时间是30分钟后这里只需要修改对应常量即可。

### 选中数据的处理
第一次的做法是在```- (void)pickerView:(UIPickerView *)pickerView didSelectRow:(NSInteger)row inComponent:(NSInteger)component```中切换日期或者小时的时候重新计算数据源，但是发现这样的效果并不好，有明显的卡顿现象，想起来这样的真的是很愚蠢的办法。应该初始化的时候计算好数据源，而不是每次都重新计算。  

在切换日期或者小时数的时候切换数据源，具体实现：  

```
- (NSInteger)pickerView:(UIPickerView *)pickerView numberOfRowsInComponent:(NSInteger)component {
    if (component == 0) {
        return self.dataSourceModel.dateArray.count;
    } else if (component == 1) {
        if (self.selectedDateIndex == 0) {//选中的今天
            return self.dataSourceModel.todayHourArray.count;
        } else {
            return self.dataSourceModel.hourArray.count;
        }
    } else {
        if (self.selectedHourIndex == 0 && self.selectedDateIndex == 0) {//选中的当天的第一个小时
            return self.dataSourceModel.todayMinuteArray.count;
        } else {
            return self.dataSourceModel.minuteArray.count;
        }
    }
}
```  
关于数据源的计算，比较直观，这里就不贴出来了，详情请看[QFDatePickerView](https://github.com/qingfengiOS/QFDatePickerView)中```QFTimerUtil```文件```+ (QFTimerDataSourceModel *)configDataSource```方法

### pickerView自定义展示
可以直接通过默认的代理方法```- (nullable NSString *)pickerView:(UIPickerView *)pickerView titleForRow:(NSInteger)row forComponent:(NSInteger)component __TVOS_PROHIBITED```实现日期显示，但是这样的展示效果却和设计图差距较大，所以实现了```- (UIView *)pickerView:(UIPickerView *)pickerView viewForRow:(NSInteger)row forComponent:(NSInteger)component reusingView:(UILabel *)recycledLabel ```自定义展示：  

```
- (UIView *)pickerView:(UIPickerView *)pickerView viewForRow:(NSInteger)row forComponent:(NSInteger)component reusingView:(UILabel *)recycledLabel {
    if (!recycledLabel) {
        recycledLabel = [[UILabel alloc] init];
    }
    recycledLabel.textAlignment = NSTextAlignmentCenter;
    [recycledLabel setFont:[UIFont systemFontOfSize:18]];
    recycledLabel.textColor = [UIColor colorWithRed:34.0f / 255.0f green:34.0f / 255.0f blue:34.0f / 255.0f alpha:1.0f];
    ...
   	recycledLabel.text = minModel.showMinuteString;
    return recycledLabel;
}
```

## 使用方式
手动拖入文件夹 或者 ```pod 'QFDatePicker'```  
导入QFTimerPicker头文件，在对应的地方调用picker的初始化方法和show方法：  

```
/**
 初始化时间选择

 @param block 回调block 参数即是选择的日期
 @return 时间选择器实例
 */
- (instancetype)initWithResponse:(ReturnBlock)block;

/**
 初始化时间选择
 
 @param superView 时间选择器的父View，若为空，将时间选择器加载在window上面
 @param block 回调block 参数即是选择的日期
 @return 时间选择器实例
 */
- (instancetype)initWithSuperView:(UIView *)superView response:(ReturnBlock)block;
```
注释比较清楚了，通过```superView```参数，控制这个picker加载在什么视图上，当其为空的时候加载在window上。  

选中的时间再block中回调（PS：这里如果把picker设置为属性时，考虑循环强引用的问题）
 
具体调用案例:  

```
QFTimerPicker *picker = [[QFTimerPicker alloc]initWithSuperView:self.view response:^(NSString *selectedStr) {
        NSLog(@"%@",selectedStr);
        [sender setTitle:selectedStr forState:UIControlStateNormal];
    }];
[picker show];
```

## 总结
1. 数据源的预加载处理
2. 对define和const取舍
3. NSDateFormatter的缓存
4. 日期类的计算，比如获取当前时间，计算星期几等   

[演示Demo](https://github.com/qingfengiOS/QFDatePickerView)  
[cocoaPods安装](https://github.com/qingfengiOS/QFDatePicker)
