#NSUInteger造成的Bug
前两天接到测试同学的反映一个一级bug：点击了某个按钮之后，整个APP卡死，点击任何按钮都没反应，第一反应就是主线程有耗时操作，导致主线程被阻塞，不响应事件，紧接着一步一步排查代码，结果发现完全不是耗时操作阻塞主线程的锅，万恶之源是这个： 
``` 
NSInteger row = ((self.hotArray.count - 1) / 3) + 1;
```
```
self.selectTriperView.slwy_height = (row * 30 + row * 5 + 5) + 25;  
```
```
[self.selectTriperView refreshView:(row * 30 + row * 5 + 5) selectArray:self.hotArray fromType:2];
```
这里是通过接口获取一个数组，把数组值展示在collectionView之上，由于数组数量是动态的，所以需要根据具体具体个数计算出整个collectionView的高度。好了，问题就在这个self.hotArray.count - 1，当这个数组为空的时候，取出来的count和预期中的0大相径庭，结果是：18446744073709551615 ，（它等于2的64次方减一,也就是64位系统的能表示的最大数减一）， .count是调用了方法：  
```- (NSUInteger)count;```  
它获取数组元素数量，但是它的返回值是NSUInteger，当一个无符号数0减一时，结果就是无法预期的，这里由于计算出的数值大于了NSInteger能表示的最大值导致卡死。
参见：[BugByNSUInteger](https://github.com/qingfengiOS/BugByNSUInteger).  
最后，在此提醒广大同仁，使用.count时注意对空数组的处理!!!!!!