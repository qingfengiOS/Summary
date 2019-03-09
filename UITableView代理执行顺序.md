#iOS不同系统版本tableView代理方法的执行顺序  
今天意外发现在低版本(9.3)的iOS系统上出现崩溃问题，最后定位到的问题是UITableView的代理方法执行顺序导致的,特此用Demo对tableView的代理方法的调用顺序做了测试。  

###一、前置条件：
tableView有2个section，每个section包含3个row 
###二、结果：
####iOS9.3系统：
```
641 CoreAnimation系列[55359:1224886] -[ViewController numberOfSectionsInTableView:]
642 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
642 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
642 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
642 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
642 CoreAnimation系列[55359:1224886] -[ViewController tableView:numberOfRowsInSection:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
643 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
644 CoreAnimation系列[55359:1224886] -[ViewController tableView:numberOfRowsInSection:]
644 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
644 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
644 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
645 CoreAnimation系列[55359:1224886] -[ViewController numberOfSectionsInTableView:]
645 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
645 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
645 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
645 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:numberOfRowsInSection:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
646 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
647 CoreAnimation系列[55359:1224886] -[ViewController tableView:numberOfRowsInSection:]
647 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
647 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
647 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
651 CoreAnimation系列[55359:1224886] -[ViewController numberOfSectionsInTableView:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:numberOfRowsInSection:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
651 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForHeaderInSection:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForFooterInSection:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:numberOfRowsInSection:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
652 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
653 CoreAnimation系列[55359:1224886] -[ViewController tableView:cellForRowAtIndexPath:]
653 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
654 CoreAnimation系列[55359:1224886] -[ViewController tableView:cellForRowAtIndexPath:]
654 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
654 CoreAnimation系列[55359:1224886] -[ViewController tableView:cellForRowAtIndexPath:]
654 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
655 CoreAnimation系列[55359:1224886] -[ViewController tableView:cellForRowAtIndexPath:]
655 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
655 CoreAnimation系列[55359:1224886] -[ViewController tableView:cellForRowAtIndexPath:]
655 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
655 CoreAnimation系列[55359:1224886] -[ViewController tableView:cellForRowAtIndexPath:]
656 CoreAnimation系列[55359:1224886] -[ViewController tableView:heightForRowAtIndexPath:]
656 CoreAnimation系列[55359:1224886] -[ViewController tableView:viewForHeaderInSection:]
656 CoreAnimation系列[55359:1224886] -[ViewController tableView:viewForFooterInSection:]
656 CoreAnimation系列[55359:1224886] -[ViewController tableView:viewForHeaderInSection:]
656 CoreAnimation系列[55359:1224886] -[ViewController tableView:viewForFooterInSection:]

```
可以看到在iOS9.3系统上: 
#####1）、轮回执行  
1、执行一次```numberOfSectionsInTableView```；  
2、依次执行```heightForHeaderInSection ```和```heightForFooterInSection ```各两次；  
3、执行一次```numberOfRowsInSection```；  
4、执行三次```heightForRowAtIndexPath```；  
5、执行```heightForHeaderInSection ```和```heightForFooterInSection ```各两次；  
6、执行一次```numberOfRowsInSection ```；  
7、执行三次```heightForRowAtIndexPath```； 
#####2）、组合执行
以上7步为一个轮回；执行三个轮回之后，  
8、执行```cellForRowAtIndexPath```和```heightForRowAtIndexPath```两个方法一起共6次；  
9、最后```viewForHeaderInSection```和```viewForFooterInSection ```一起两次
####iOS11.3:
```
.827590+0800 CoreAnimation系列[54215:1200550] -[ViewController numberOfSectionsInTableView:]
.827974+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:numberOfRowsInSection:]
.828114+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:numberOfRowsInSection:]
.828288+0800 CoreAnimation系列[54215:1200550] -[ViewController numberOfSectionsInTableView:]
.828375+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:numberOfRowsInSection:]
.828476+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:numberOfRowsInSection:]
.830575+0800 CoreAnimation系列[54215:1200550] -[ViewController numberOfSectionsInTableView:]
.830683+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:numberOfRowsInSection:]
.830762+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:numberOfRowsInSection:]
.831455+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:cellForRowAtIndexPath:]
.831948+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.832127+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForFooterInSection:]
.836103+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:cellForRowAtIndexPath:]
.836396+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.836750+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:cellForRowAtIndexPath:]
.836994+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.837325+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:cellForRowAtIndexPath:]
.837518+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.837595+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForFooterInSection:]
.837889+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:cellForRowAtIndexPath:]
.838063+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.838374+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:cellForRowAtIndexPath:]
.838609+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.838796+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForHeaderInSection:]
.838888+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:viewForHeaderInSection:]
.839054+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:viewForFooterInSection:]
.839197+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForHeaderInSection:]
.839270+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:viewForHeaderInSection:]
.839377+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:viewForFooterInSection:]
.839478+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.839566+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.839638+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.839720+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.839797+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
.839871+0800 CoreAnimation系列[54215:1200550] -[ViewController tableView:heightForRowAtIndexPath:]
```
在iOS11.3系统上:  
1、执行一次```numberOfSectionsInTableView```+两次```numberOfRowsInSection ```，一起轮回三次  
2、执行```cellForRowAtIndexPath``` + ```heightForRowAtIndexPat``` + ```heightForFooterInSection```各一次  
3、执行```cellForRowAtIndexPath``` + ```heightForRowAtIndexPat```各一次，轮回三次    
4、执行一次```heightForFooterInSection```  
5、执行```cellForRowAtIndexPath``` + ```heightForRowAtIndexPat```各一次，轮回两次  
6、执行```heightForHeaderInSection ```+```viewForHeaderInSection ```+```viewForFooterInSection```各一次，轮回两次
7、执行```heightForRowAtIndexPath```六次
###三、结论：
iOS11.0之后对tableview的数据源调用有了很大改变，改变了调用顺序，所以如果你出现在不同版本的iOS系统上出现数据源问题的话，可以考虑代理方法的调用顺序，从而查找原因。
