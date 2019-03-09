#带年月和至今以及设置分钟间隔的时间选择器
在平时的开发过程中，肯定遇到过一些需求，比如一些场景中，只需要选择年、月不需要天，并且包含“至今”，系统默认的pickerView呢，并不能满足，只能自定义数据源，当然这并不困难，但如果有一个现成的时间选择器，何乐不为呢。

使用起来也比较方便，导入头文件头文件，在需要的地方调用```initDatePackerWithResponse```方法和```show```方法，在block里面处理得到的时间字符串，时间字符串在block里面，注意循环引用的问题。  
效果如下：
![datePicker](https://upload-images.jianshu.io/upload_images/2598795-ca8facdd816d008b.png)  
上面我们已经完成了对日期的选择，接下来就是对时间的选择，苹果原生的DatePicker支持以每一分钟为单位的时间选择，但是在实际项目中常常需求并非如此，并不需要选到每一分钟，可能需要以5分钟、10分钟等为单位选择，好了，话不多说，看效果：  
![timepicker](https://upload-images.jianshu.io/upload_images/2598795-4ffe057b67124b5c.png)  
调用方法和日期选择一样，直接调用```initDatePackerWithStartHour:endHour: period: response:(void (^)(NSString *))block```方法，传入开始时间，结束时间，时间间隔三个参数，返回结果在block里处理。  
[Github地址:](github.com/qingfengiOS/QFDatePickerView)如果有用呢，希望能顺手给个star，有什么意见和建议希望各路大神不吝赐教！
