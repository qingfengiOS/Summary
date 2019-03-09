#SDWebImage源码解析
### 概览
说到 iOS界的图片加载库，[SDWebImage](https://github.com/rs/SDWebImage.git)可谓无人不知，其简介的接口以及异步下载与缓存的强大功能，深受青睐，github上20K+的Star数说明一切。项目的大体框架结构如下：  
![结构.png](https://upload-images.jianshu.io/upload_images/2598795-80afa520741d3987.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>UIImageView+WebCache和UIButton+WebCache直接为表层的 UIKit框架提供接口,   
>SDWebImageManger负责处理和协调SDWebImageDownloader和SDWebImageCache, 并与UIKit层进行交互。  
>SDWebImageDownloaderOperation真正执行下载请求；最底层的两个类为高层抽象提供支持。

整个流程大体如下所示:  
![调用流程.png](https://upload-images.jianshu.io/upload_images/2598795-0bc8fc36a02b270b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面我们从一个具体案例来一步一步分析，以UIimageView为例，最外层调用：

```
[_imageView sd_setImageWithURL:url placeholderImage:nil options:SDWebImageProgressiveDownload];
```
值得一说的是：这里SDWebImage对外提供多种接口，相当于convince方法，达到了对于可选参数的多种支持

```
- (void)sd_setImageWithURL:(nullable NSURL *)url {
    [self sd_setImageWithURL:url placeholderImage:nil options:0 progress:nil completed:nil];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:0 progress:nil completed:nil];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:options progress:nil completed:nil];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_setImageWithURL:url placeholderImage:nil options:0 progress:nil completed:completedBlock];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:0 progress:nil completed:completedBlock];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:options progress:nil completed:completedBlock];
}

- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                 completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_internalSetImageWithURL:url
                    placeholderImage:placeholder
                             options:options
                        operationKey:nil
                       setImageBlock:nil
                            progress:progressBlock
                           completed:completedBlock];
}               
```
提供多种接口，到底层调用到同一个方法，减少调用方对可选参数的传递，这样的设计思路在日常的开发中很有借鉴意义。  
经过层层传递最终的加载方法如下：

```
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context {
    NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    if (!(options & SDWebImageDelayPlaceholder)) {
        if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
            dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
            dispatch_group_enter(group);
        }
        dispatch_main_async_safe(^{
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock];
        });
    }
    
    if (url) {
        // check if activityView is enabled or not
        if ([self sd_showActivityIndicatorView]) {
            [self sd_addActivityIndicator];
        }
        
        // reset the progress
        self.sd_imageProgress.totalUnitCount = 0;
        self.sd_imageProgress.completedUnitCount = 0;
        
        SDWebImageManager *manager;
        if ([context valueForKey:SDWebImageExternalCustomManagerKey]) {
            manager = (SDWebImageManager *)[context valueForKey:SDWebImageExternalCustomManagerKey];
        } else {
            manager = [SDWebImageManager sharedManager];
        }
        
        __weak __typeof(self)wself = self;
        SDWebImageDownloaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            wself.sd_imageProgress.totalUnitCount = expectedSize;
            wself.sd_imageProgress.completedUnitCount = receivedSize;
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            __strong __typeof (wself) sself = wself;
            if (!sself) { return; }
            [sself sd_removeActivityIndicator];
            // if the progress not been updated, mark it to complete state
            if (finished && !error && sself.sd_imageProgress.totalUnitCount == 0 && sself.sd_imageProgress.completedUnitCount == 0) {
                sself.sd_imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                sself.sd_imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
            }
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!sself) { return; }
                if (!shouldNotSetImage) {
                    [sself sd_setNeedsLayout];
                }
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, error, cacheType, url);
                }
            };
            
            // case 1a: we got an image, but the SDWebImageAvoidAutoSetImage flag is set
            // OR
            // case 1b: we got no image and the SDWebImageDelayPlaceholder is not set
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }
            
            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                targetImage = placeholder;
                targetData = nil;
            }
            
            // check whether we should use the image transition
            SDWebImageTransition *transition = nil;
            if (finished && (options & SDWebImageForceTransition || cacheType == SDImageCacheTypeNone)) {
                transition = sself.sd_imageTransition;
            }
            if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
                dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
                dispatch_group_enter(group);
                dispatch_main_async_safe(^{
                    [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
                });
                // ensure completion block is called after custom setImage process finish
                dispatch_group_notify(group, dispatch_get_main_queue(), ^{
                    callCompletedBlockClojure();
                });
            } else {
                dispatch_main_async_safe(^{
                    [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
                    callCompletedBlockClojure();
                });
            }
        }];
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } else {
        dispatch_main_async_safe(^{
            [self sd_removeActivityIndicator];
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                completedBlock(nil, error, SDImageCacheTypeNone, url);
            }
        });
    }
}
``` 
这个方法有点长，下面我们来一步一步地分析 
### 具体流程
##### 一、获取validOperationKey
```
NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
```
首先判断调用方是否传递了operationKey，如果没有则直接取对应调用类的class名字  
##### 二、取消当前正在下载的任务
```
[self sd_cancelImageLoadOperationWithKey:validOperationKey];
```  
这里会调用到__UIView+WebCacheOperation__类里面的sd_cancelImageLoadOperationWithKey方法：

```
- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key {
    // Cancel in progress downloader from queue
    SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
    id<SDWebImageOperation> operation;
    @synchronized (self) {
        operation = [operationDictionary objectForKey:key];
    }
    if (operation) {
        if ([operation conformsToProtocol:@protocol(SDWebImageOperation)]){
            [operation cancel];
        }
        @synchronized (self) {
            [operationDictionary removeObjectForKey:key];
        }
    }
}
```
[self sd_operationDictionary] 实现细节如下：

```
- (SDOperationsDictionary *)sd_operationDictionary {
    @synchronized(self) {
        SDOperationsDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
        if (operations) {
            return operations;
        }
        operations = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
        objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        return operations;
    }
}
```  

1. 这里首先获取通过loadOperationKey来通过关联对象获取operationsDictionary（这里会有一次加锁）   
	- 如果非空直接返回operationDictionary  
	- 如果为空那么使用**NSMapTable**创建并返回,这里会通过关联对象以loadOperationKey保存这个operations

	>NSMapTable和字典类似，不过它的key不是一定需要遵守NSCoding协议并且他可以指定key和value的内存管理语义    

2. 通过上一步获取到了operationDictionary，这里通过key在operationDictionary里面取出对应的operation；
	- 如果operation非空并且遵守了SDWebImageOperation协议那么调用cancle取消任务，然后从operationDictionary中通过key移除这个operation

##### 三、保存url并设置占位图片

```
objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    if (!(options & SDWebImageDelayPlaceholder)) {
        if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
            dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
            dispatch_group_enter(group);
        }
        dispatch_main_async_safe(^{
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock];
        });
    }
```

1. 通过关联对象以imageURLKey为key保存url   
2. 如果是多图下载则使用dispatch_group
3. 判断如果不是延迟设置展位图，使用dispatch_main_async_safe宏设置占位图

##### 四、判断url是否为空  
如果url为空直接在主线程移除加载指示器并回调加载失败(原因为url为nil)  
如果url非空进入下一步
##### 五、处理加载进度条  

1. 通过关联对象添加的属性**TAG_ACTIVITY_SHOW**判断时候开启加载进度activityIndicator  
2. 通过sd_imageProgress的两个属性totalUnitCount和completedUnitCount保存加载的进度 

```
SDWebImageDownloaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            wself.sd_imageProgress.totalUnitCount = expectedSize;
            wself.sd_imageProgress.completedUnitCount = receivedSize;
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
```
##### 六、获取SDWebImageManager对象

```
SDWebImageManager *manager;
if ([context valueForKey:SDWebImageExternalCustomManagerKey]) {
    manager = (SDWebImageManager *)[context valueForKey:SDWebImageExternalCustomManagerKey];
} else {
    manager = [SDWebImageManager sharedManager];
}
```
先从context中获取manager，如果没有则创建SDWebImageManager单例对象
##### 七、调用SDWebImageManager的loadImageWithURL方法获取一个加载任务

```
id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) 
```
这里为什么要获取一个SDWebImageOperation对象呢?所有的结果不是都在block里面回调的吗？  
我们看下源码,在方法在末尾调用了方法   

```
[self sd_setImageLoadOperation:operation forKey:validOperationKey];
```
其实现细节如下：

```
- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key {
    if (key) {
        [self sd_cancelImageLoadOperationWithKey:key];
        if (operation) {
            SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
            @synchronized (self) {
                [operationDictionary setObject:operation forKey:key];
            }
        }
    }
}
```
这里的目的其实是将这个任务（operation）通过validOperationKey保存到operationsDictionary中，和第二步的取消任务相呼应。  
##### 八、判断url的合法性以及有效性

```
if ([url isKindOfClass:NSString.class]) {
    url = [NSURL URLWithString:(NSString *)url];
}

// Prevents app crashing on argument type error like sending NSNull instead of NSURL
if (![url isKindOfClass:NSURL.class]) {
    url = nil;
}
```

1. 如注释的一样，有时候可能传入的url并不是url而是一个string，这里做一层包装，如果url类型不是NSURL那么把url置空，防止崩溃。  
2. 接下来创建一个SDWebImageCombinedOperation对象，并且设置其manager为self  
3. 接下来判断url是否是加载失败的url(这里用到了NSMutableSet和加锁操作)，
	- 如果是失败的url或者url无效则直接回调失败  
	- 如果url有效则通过url获取key

##### 九、查找缓存
经过上一步的url验证之后会调用到**SDImageCache**里的queryCacheOperationForKey方法查找缓存，首先判断这里的传入的key是否为空，如果为nil了直接return,这里的缓存查找分为两步，内存缓存和磁盘缓存  

```
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock {
    if (!key) {
        if (doneBlock) {
            doneBlock(nil, nil, SDImageCacheTypeNone);
        }
        return nil;
    }
    
    // First check the in-memory cache...
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    NSOperation *operation = [NSOperation new];
    void(^queryDiskBlock)(void) =  ^{
        if (operation.isCancelled) {
            // do not call the completion if cancelled
            return;
        }
        
        @autoreleasepool {
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage;
            SDImageCacheType cacheType = SDImageCacheTypeDisk;
            if (image) {
                // the image is from in-memory cache
                diskImage = image;
                cacheType = SDImageCacheTypeMemory;
            } else if (diskData) {
                // decode image data only if in-memory cache missed
                diskImage = [self diskImageForKey:key data:diskData];
                if (diskImage && self.config.shouldCacheImagesInMemory) {
                    NSUInteger cost = SDCacheCostForImage(diskImage);
                    [self.memCache setObject:diskImage forKey:key cost:cost];
                }
            }
            
            if (doneBlock) {
                if (options & SDImageCacheQueryDiskSync) {
                    doneBlock(diskImage, diskData, cacheType);
                } else {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        doneBlock(diskImage, diskData, cacheType);
                    });
                }
            }
        }
    };
    
    if (options & SDImageCacheQueryDiskSync) {
        queryDiskBlock();
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}
```

1. 内存缓存查找  
 
```
- (nullable UIImage *)imageFromMemoryCacheForKey:(nullable NSString *)key {
    return [self.memCache objectForKey:key];
}
```  
这里直接通过key去cache里面查找，这里的self.memCache是继承自NSCache的SDMemoryCache关于Cache的介绍请看[NSCache](https://nshipster.cn/nscache/)  
如果这里找到了，并且没有强制指明查询磁盘的话直接回调

```
BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
if (shouldQueryMemoryOnly) {
    if (doneBlock) {
        doneBlock(image, nil, SDImageCacheTypeMemory);
    }
    return nil;
}
/**
 * By default, we do not query disk data when the image is cached in memory. This mask can force to query disk data at the same time.
 */
SDImageCacheQueryDataWhenInMemory = 1 << 0,
```  

2. 磁盘缓存查找  
上一步如果没有找到的话，就会进如磁盘查找阶段，此时会新建一个NSOperation，执行queryDiskBlock代码块  
	- 2.1 通过key查找磁盘获取diskData
	- 2.2 如果上一步内存找到image直接赋值，否则解码diskData，并且把磁盘的image写入内存
	
	```
	[self.memCache setObject:diskImage forKey:key cost:cost];
	```

##### 十、下载图片
在方法  
```
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock
```  
的回调block中，首先判断是否需要下载  

```
__strong __typeof(weakOperation) strongOperation = weakOperation;
if (!strongOperation || strongOperation.isCancelled) {
    [self safelyRemoveOperationFromRunning:strongOperation];
    return;
}
    
// Check whether we should download image from network
BOOL shouldDownload = (!(options & SDWebImageFromCacheOnly))
    && (!cachedImage || options & SDWebImageRefreshCached)
    && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
```  
如果需要下载，调用**SDWebImageDownloader**的  

```
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    __weak SDWebImageDownloader *wself = self;

    return [self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^SDWebImageDownloaderOperation *{
        __strong __typeof (wself) sself = wself;
        NSTimeInterval timeoutInterval = sself.downloadTimeout;
        if (timeoutInterval == 0.0) {
            timeoutInterval = 15.0;
        }

        // In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
        NSURLRequestCachePolicy cachePolicy = options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData;
        NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url
                                                                    cachePolicy:cachePolicy
                                                                timeoutInterval:timeoutInterval];
        
        request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
        request.HTTPShouldUsePipelining = YES;
        if (sself.headersFilter) {
            request.allHTTPHeaderFields = sself.headersFilter(url, [sself allHTTPHeaderFields]);
        }
        else {
            request.allHTTPHeaderFields = [sself allHTTPHeaderFields];
        }
        SDWebImageDownloaderOperation *operation = [[sself.operationClass alloc] initWithRequest:request inSession:sself.session options:options];
        operation.shouldDecompressImages = sself.shouldDecompressImages;
        
        if (sself.urlCredential) {
            operation.credential = sself.urlCredential;
        } else if (sself.username && sself.password) {
            operation.credential = [NSURLCredential credentialWithUser:sself.username password:sself.password persistence:NSURLCredentialPersistenceForSession];
        }
        
        if (options & SDWebImageDownloaderHighPriority) {
            operation.queuePriority = NSOperationQueuePriorityHigh;
        } else if (options & SDWebImageDownloaderLowPriority) {
            operation.queuePriority = NSOperationQueuePriorityLow;
        }
        
        if (sself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
            [sself.lastAddedOperation addDependency:operation];
            sself.lastAddedOperation = operation;
        }

        return operation;
    }];
}
```  
方法，在此内部调用到了**SDWebImageDownloader**的
                                                 
``` 
- (nullable SDWebImageDownloadToken *)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock
                                           completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock
                                                   forURL:(nullable NSURL *)url
                                           createCallback:(SDWebImageDownloaderOperation *(^)(void))createCallback {
    // The URL will be used as the key to the callbacks dictionary so it cannot be nil. If it is nil immediately call the completed block with no image or data.
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return nil;
    }
    
    LOCK(self.operationsLock);
    SDWebImageDownloaderOperation *operation = [self.URLOperations objectForKey:url];
    if (!operation) {
        operation = createCallback();
        __weak typeof(self) wself = self;
        operation.completionBlock = ^{
            __strong typeof(wself) sself = wself;
            if (!sself) {
                return;
            }
            LOCK(sself.operationsLock);
            [sself.URLOperations removeObjectForKey:url];
            UNLOCK(sself.operationsLock);
        };
        [self.URLOperations setObject:operation forKey:url];
        // Add operation to operation queue only after all configuration done according to Apple's doc.
        // `addOperation:` does not synchronously execute the `operation.completionBlock` so this will not cause deadlock.
        /*------------这里添加下载任务到downQueue----------------*/
        [self.downloadQueue addOperation:operation];
    }
    UNLOCK(self.operationsLock);

    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    
    SDWebImageDownloadToken *token = [SDWebImageDownloadToken new];
    token.downloadOperation = operation;
    token.url = url;
    token.downloadOperationCancelToken = downloadOperationCancelToken;

    return token;
}
``` 
在这里把operation添加到下载队列self.downloadQueue中，self.downloadQueue是NSOperationQueue的实例，它默认的最大并发数为6  

```
_downloadQueue.maxConcurrentOperationCount = 6;
```  
当然，可以通过属性修改这个默认值  

```
- (void)setMaxConcurrentDownloads:(NSInteger)maxConcurrentDownloads {
    _downloadQueue.maxConcurrentOperationCount = maxConcurrentDownloads;
}
```
##### 十一、图片下载回调处理
下载使用的NSURLSession，所以在**SDWebImageDownloader**里面有着NSURLSessionDataDelegate代理方法  

```
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:dataTask];
    if ([dataOperation respondsToSelector:@selector(URLSession:dataTask:didReceiveResponse:completionHandler:)]) {
        [dataOperation URLSession:session dataTask:dataTask didReceiveResponse:response completionHandler:completionHandler];
    } else {
        if (completionHandler) {
            completionHandler(NSURLSessionResponseAllow);
        }
    }
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:dataTask];
    if ([dataOperation respondsToSelector:@selector(URLSession:dataTask:didReceiveData:)]) {
        [dataOperation URLSession:session dataTask:dataTask didReceiveData:data];
    }
}

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:dataTask];
    if ([dataOperation respondsToSelector:@selector(URLSession:dataTask:willCacheResponse:completionHandler:)]) {
        [dataOperation URLSession:session dataTask:dataTask willCacheResponse:proposedResponse completionHandler:completionHandler];
    } else {
        if (completionHandler) {
            completionHandler(proposedResponse);
        }
    }
}

#pragma mark NSURLSessionTaskDelegate

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    
    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:task];
    if ([dataOperation respondsToSelector:@selector(URLSession:task:didCompleteWithError:)]) {
        [dataOperation URLSession:session task:task didCompleteWithError:error];
    }
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURLRequest * _Nullable))completionHandler {
    
    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:task];
    if ([dataOperation respondsToSelector:@selector(URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:)]) {
        [dataOperation URLSession:session task:task willPerformHTTPRedirection:response newRequest:request completionHandler:completionHandler];
    } else {
        if (completionHandler) {
            completionHandler(request);
        }
    }
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler {

    // Identify the operation that runs this task and pass it the delegate method
    SDWebImageDownloaderOperation *dataOperation = [self operationWithTask:task];
    if ([dataOperation respondsToSelector:@selector(URLSession:task:didReceiveChallenge:completionHandler:)]) {
        [dataOperation URLSession:session task:task didReceiveChallenge:challenge completionHandler:completionHandler];
    } else {
        if (completionHandler) {
            completionHandler(NSURLSessionAuthChallengePerformDefaultHandling, nil);
        }
    }
}
```
在每个方法中都调用了

```

- (SDWebImageDownloaderOperation *)operationWithTask:(NSURLSessionTask *)task {
    SDWebImageDownloaderOperation *returnOperation = nil;
    for (SDWebImageDownloaderOperation *operation in self.downloadQueue.operations) {
        if (operation.dataTask.taskIdentifier == task.taskIdentifier) {
            returnOperation = operation;
            break;
        }
    }
    return returnOperation;
}
```  
通过task获取对应的**SDWebImageDownloaderOperation**实例，将具体的操作放到了这个类里面

```
#pragma mark NSURLSessionDataDelegate

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
    NSURLSessionResponseDisposition disposition = NSURLSessionResponseAllow;
    NSInteger expected = (NSInteger)response.expectedContentLength;
    expected = expected > 0 ? expected : 0;
    self.expectedSize = expected;
    self.response = response;
    NSInteger statusCode = [response respondsToSelector:@selector(statusCode)] ? ((NSHTTPURLResponse *)response).statusCode : 200;
    BOOL valid = statusCode < 400;
    //'304 Not Modified' is an exceptional one. It should be treated as cancelled if no cache data
    //URLSession current behavior will return 200 status code when the server respond 304 and URLCache hit. But this is not a standard behavior and we just add a check
    if (statusCode == 304 && !self.cachedData) {
        valid = NO;
    }
    
    if (valid) {
        for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            progressBlock(0, expected, self.request.URL);
        }
    } else {
        // Status code invalid and marked as cancelled. Do not call `[self.dataTask cancel]` which may mass up URLSession life cycle
        disposition = NSURLSessionResponseCancel;
    }
    
    __weak typeof(self) weakSelf = self;
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadReceiveResponseNotification object:weakSelf];
    });
    
    if (completionHandler) {
        completionHandler(disposition);
    }
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
    if (!self.imageData) {
        self.imageData = [[NSMutableData alloc] initWithCapacity:self.expectedSize];
    }
    [self.imageData appendData:data];

    if ((self.options & SDWebImageDownloaderProgressiveDownload) && self.expectedSize > 0) {
        // Get the image data
        __block NSData *imageData = [self.imageData copy];
        // Get the total bytes downloaded
        const NSInteger totalSize = imageData.length;
        // Get the finish status
        BOOL finished = (totalSize >= self.expectedSize);
        
        if (!self.progressiveCoder) {
            // We need to create a new instance for progressive decoding to avoid conflicts
            for (id<SDWebImageCoder>coder in [SDWebImageCodersManager sharedInstance].coders) {
                if ([coder conformsToProtocol:@protocol(SDWebImageProgressiveCoder)] &&
                    [((id<SDWebImageProgressiveCoder>)coder) canIncrementallyDecodeFromData:imageData]) {
                    self.progressiveCoder = [[[coder class] alloc] init];
                    break;
                }
            }
        }
        
        // progressive decode the image in coder queue
        dispatch_async(self.coderQueue, ^{
            UIImage *image = [self.progressiveCoder incrementallyDecodedImageWithData:imageData finished:finished];
            if (image) {
                NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
                image = [self scaledImageForKey:key image:image];
                if (self.shouldDecompressImages) {
                    image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&imageData options:@{SDWebImageCoderScaleDownLargeImagesKey: @(NO)}];
                }
                
                // We do not keep the progressive decoding image even when `finished`=YES. Because they are for view rendering but not take full function from downloader options. And some coders implementation may not keep consistent between progressive decoding and normal decoding.
                
                [self callCompletionBlocksWithImage:image imageData:nil error:nil finished:NO];
            }
        });
    }

    for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
        progressBlock(self.imageData.length, self.expectedSize, self.request.URL);
    }
}

- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler {
    
    NSCachedURLResponse *cachedResponse = proposedResponse;

    if (!(self.options & SDWebImageDownloaderUseNSURLCache)) {
        // Prevents caching of responses
        cachedResponse = nil;
    }
    if (completionHandler) {
        completionHandler(cachedResponse);
    }
}

#pragma mark NSURLSessionTaskDelegate

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    @synchronized(self) {
        self.dataTask = nil;
        __weak typeof(self) weakSelf = self;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:weakSelf];
            if (!error) {
                [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadFinishNotification object:weakSelf];
            }
        });
    }
    
    // make sure to call `[self done]` to mark operation as finished
    if (error) {
        [self callCompletionBlocksWithError:error];
        [self done];
    } else {
        if ([self callbacksForKey:kCompletedCallbackKey].count > 0) {
            /**
             *  If you specified to use `NSURLCache`, then the response you get here is what you need.
             */
            __block NSData *imageData = [self.imageData copy];
            if (imageData) {
                /**  if you specified to only use cached data via `SDWebImageDownloaderIgnoreCachedResponse`,
                 *  then we should check if the cached data is equal to image data
                 */
                if (self.options & SDWebImageDownloaderIgnoreCachedResponse && [self.cachedData isEqualToData:imageData]) {
                    // call completion block with nil
                    [self callCompletionBlocksWithImage:nil imageData:nil error:nil finished:YES];
                    [self done];
                } else {
                    // decode the image in coder queue
                    dispatch_async(self.coderQueue, ^{
                        UIImage *image = [[SDWebImageCodersManager sharedInstance] decodedImageWithData:imageData];
                        NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
                        image = [self scaledImageForKey:key image:image];
                        
                        BOOL shouldDecode = YES;
                        // Do not force decoding animated GIFs and WebPs
                        if (image.images) {
                            shouldDecode = NO;
                        } else {
#ifdef SD_WEBP
                            SDImageFormat imageFormat = [NSData sd_imageFormatForImageData:imageData];
                            if (imageFormat == SDImageFormatWebP) {
                                shouldDecode = NO;
                            }
#endif
                        }
                        
                        if (shouldDecode) {
                            if (self.shouldDecompressImages) {
                                BOOL shouldScaleDown = self.options & SDWebImageDownloaderScaleDownLargeImages;
                                image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&imageData options:@{SDWebImageCoderScaleDownLargeImagesKey: @(shouldScaleDown)}];
                            }
                        }
                        CGSize imageSize = image.size;
                        if (imageSize.width == 0 || imageSize.height == 0) {
                            [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Downloaded image has 0 pixels"}]];
                        } else {
                            [self callCompletionBlocksWithImage:image imageData:imageData error:nil finished:YES];
                        }
                        [self done];
                    });
                }
            } else {
                [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Image data is nil"}]];
                [self done];
            }
        } else {
            [self done];
        }
    }
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler {
    
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;
    
    if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
        if (!(self.options & SDWebImageDownloaderAllowInvalidSSLCertificates)) {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        } else {
            credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
            disposition = NSURLSessionAuthChallengeUseCredential;
        }
    } else {
        if (challenge.previousFailureCount == 0) {
            if (self.credential) {
                credential = self.credential;
                disposition = NSURLSessionAuthChallengeUseCredential;
            } else {
                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
            }
        } else {
            disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
        }
    }
    
    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}
```
这里面对需要解码的图片做了解码操作和判断是否图片需要缩放等处理

##### 十二、图片缓存  
经过层层回调，结果会回到到第七步的**SDWebImageManager**的```- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock ```方法中``` - (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock```的completedBlock中，这里调用了```[self.imageCache storeImage:transformedImage imageData:cacheData forKey:key toDisk:cacheOnDisk completion:nil];``` 方法做图片缓存操作，我们来看一下实现细节  
                                                 
```
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    if (!image || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    // if memory cache is enabled
    /*-------------------这里存内存------------------------*/
    if (self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(image);
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    /*-------------------这里存磁盘------------------------*/
    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            @autoreleasepool {
                NSData *data = imageData;
                if (!data && image) {
                    // If we do not have any data to detect image format, check whether it contains alpha channel to use PNG or JPEG format
                    SDImageFormat format;
                    if (SDCGImageRefContainsAlpha(image.CGImage)) {
                        format = SDImageFormatPNG;
                    } else {
                        format = SDImageFormatJPEG;
                    }
                    data = [[SDWebImageCodersManager sharedInstance] encodedDataWithImage:image format:format];
                }
                [self _storeImageDataToDisk:data forKey:key];
            }
            
            if (completionBlock) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionBlock();
                });
            }
        });
    } else {
        if (completionBlock) {
            completionBlock();
        }
    }
}
```
可以看到

1. 这里使用了self.memCache直接存入内存
 
	```
	if (self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(image);
        [self.memCache setObject:image forKey:key cost:cost];
    }
    
    //下面取cost
	FOUNDATION_STATIC_INLINE NSUInteger SDCacheCostForImage(UIImage *image) {
#if SD_MAC
    return image.size.height * image.size.width;
#elif SD_UIKIT || SD_WATCH
    return image.size.height * image.size.width * image.scale * image.scale;
#endif
}
	```
2. 判断是否需要写入磁盘，如果需要的话在异步队列self.ioQueue中调用方法写入磁盘，实现如下：
	
	```
	- (void)_storeImageDataToDisk:(nullable NSData *)imageData forKey:(nullable NSString *)key {
    if (!imageData || !key) {
        return;
    }
    
    if (![self.fileManager fileExistsAtPath:_diskCachePath]) {
        [self.fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }
    
    // get cache Path for image key
    NSString *cachePathForKey = [self defaultCachePathForKey:key];
    // transform to NSUrl
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    
    [imageData writeToURL:fileURL options:self.config.diskCacheWritingOptions error:nil];
    
    // disable iCloud backup
    if (self.config.shouldDisableiCloud) {
        [fileURL setResourceValue:@YES forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}
	```

##### 十三、清理策略
所有的缓存都有时效和大小的限制，在SDImageCache中有着一些列的清理策略
默认图片的有效缓存期为7天 

```
static const NSInteger kDefaultCacheMaxCacheAge = 60 * 60 * 24 * 7; // 1 week

```

```
#pragma mark - Cache clean Ops

- (void)clearMemory {
    [self.memCache removeAllObjects];
}

- (void)clearDiskOnCompletion:(nullable SDWebImageNoParamsBlock)completion {
    dispatch_async(self.ioQueue, ^{
        [self.fileManager removeItemAtPath:self.diskCachePath error:nil];
        [self.fileManager createDirectoryAtPath:self.diskCachePath
                withIntermediateDirectories:YES
                                 attributes:nil
                                      error:NULL];

        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion();
            });
        }
    });
}

- (void)deleteOldFiles {
    [self deleteOldFilesWithCompletionBlock:nil];
}

- (void)deleteOldFilesWithCompletionBlock:(nullable SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        NSURL *diskCacheURL = [NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
        NSArray<NSString *> *resourceKeys = @[NSURLIsDirectoryKey, NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey];

        // This enumerator prefetches useful properties for our cache files.
        NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtURL:diskCacheURL
                                                   includingPropertiesForKeys:resourceKeys
                                                                      options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                 errorHandler:NULL];

        NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.config.maxCacheAge];
        NSMutableDictionary<NSURL *, NSDictionary<NSString *, id> *> *cacheFiles = [NSMutableDictionary dictionary];
        NSUInteger currentCacheSize = 0;

        // Enumerate all of the files in the cache directory.  This loop has two purposes:
        //
        //  1. Removing files that are older than the expiration date.
        //  2. Storing file attributes for the size-based cleanup pass.
        NSMutableArray<NSURL *> *urlsToDelete = [[NSMutableArray alloc] init];
        for (NSURL *fileURL in fileEnumerator) {
            NSError *error;
            NSDictionary<NSString *, id> *resourceValues = [fileURL resourceValuesForKeys:resourceKeys error:&error];

            // Skip directories and errors.
            if (error || !resourceValues || [resourceValues[NSURLIsDirectoryKey] boolValue]) {
                continue;
            }

            // Remove files that are older than the expiration date;
            NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
            if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }

            // Store a reference to this file and account for its total size.
            NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
            currentCacheSize += totalAllocatedSize.unsignedIntegerValue;
            cacheFiles[fileURL] = resourceValues;
        }
        
        for (NSURL *fileURL in urlsToDelete) {
            [self.fileManager removeItemAtURL:fileURL error:nil];
        }

        // If our remaining disk cache exceeds a configured maximum size, perform a second
        // size-based cleanup pass.  We delete the oldest files first.
        if (self.config.maxCacheSize > 0 && currentCacheSize > self.config.maxCacheSize) {
            // Target half of our maximum cache size for this cleanup pass.
            const NSUInteger desiredCacheSize = self.config.maxCacheSize / 2;

            // Sort the remaining cache files by their last modification time (oldest first).
            NSArray<NSURL *> *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                                     usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                         return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                                     }];

            // Delete files until we fall below our desired cache size.
            for (NSURL *fileURL in sortedFiles) {
                if ([self.fileManager removeItemAtURL:fileURL error:nil]) {
                    NSDictionary<NSString *, id> *resourceValues = cacheFiles[fileURL];
                    NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                    currentCacheSize -= totalAllocatedSize.unsignedIntegerValue;

                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}

#if SD_UIKIT
- (void)backgroundDeleteOldFiles {
    Class UIApplicationClass = NSClassFromString(@"UIApplication");
    if(!UIApplicationClass || ![UIApplicationClass respondsToSelector:@selector(sharedApplication)]) {
        return;
    }
    UIApplication *application = [UIApplication performSelector:@selector(sharedApplication)];
    __block UIBackgroundTaskIdentifier bgTask = [application beginBackgroundTaskWithExpirationHandler:^{
        // Clean up any unfinished task business by marking where you
        // stopped or ending the task outright.
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];

    // Start the long-running task and return immediately.
    [self deleteOldFilesWithCompletionBlock:^{
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];
}
```
总体分类内存清理和磁盘清理

1. 内存清理 

	```
	[self.memCache removeAllObjects];
	
	```  

2. 磁盘清理
	SDWebImage监听了app进入后台的通知，当进入后台时，如果当前缓存量大于等于设置的最大缓存量，那么按时间顺序递归删除一般的内容，直到缓存量满足条件。   
	
	```
	// If our remaining disk cache exceeds a configured maximum size, perform a second
    // size-based cleanup pass.  We delete the oldest files first.
    if (self.config.maxCacheSize > 0 && currentCacheSize > self.config.maxCacheSize) {
        // Target half of our maximum cache size for this cleanup pass.
        const NSUInteger desiredCacheSize = self.config.maxCacheSize / 2;

        // Sort the remaining cache files by their last modification time (oldest first).
        NSArray<NSURL *> *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                                 usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                     return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                                 }];

        // Delete files until we fall below our desired cache size.
        for (NSURL *fileURL in sortedFiles) {
            if ([self.fileManager removeItemAtURL:fileURL error:nil]) {
                NSDictionary<NSString *, id> *resourceValues = cacheFiles[fileURL];
                NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                currentCacheSize -= totalAllocatedSize.unsignedIntegerValue;

                if (currentCacheSize < desiredCacheSize) {
                    break;
                }
            }
        }
    }
	```
	
### 后记
整个框架还有很多值得分析的地方，这里仅仅是通过一个调用案例简要分析，其中还有很多知识点  
 
- dispatch_barrier_sync函数：该方法用于对操作设置屏幕，确保在执行完任务后才会执行后续操作。该方法常用于确保类的线程安全性操作。  
- NSMutableURLRequest：用于创建一个网络请求对象，我们可以根据需要来配置请求报头等信息。
- NSOperation及NSOperationQueue：操作队列是Objective-C中一种高级的并发处理方法，现在它是基于GCD来实现的。相对于GCD来说，操作队列的优点是可以取消在任务处理队列中的任务，另外在管理操作间的依赖关系方面也容易一些。对SDWebImage中我们就看到了如何使用依赖将下载顺序设置成后进先出的顺序。
- NSURLConnection：用于网络请求及响应处理。在iOS7.0后，苹果推出了一套新的网络请求接口，即NSURLSession类。
- 开启一个后台任务。
- NSCache类：一个类似于集合的容器。它存储key-value对，这一点类似于NSDictionary类。我们通常用使用缓存来临时存储短时间使用但创建昂贵的对象。重用这些对象可以优化性能，因为它们的值不需要重新计算。另外一方面，这些对象对于程序来说不是紧要的，在内存紧张时会被丢弃。
- 清理缓存图片的策略：特别是最大缓存空间大小的设置。如果所有缓存文件的总大小超过这一大小，则会按照文件最后修改时间的逆序，以每次一半的递归来移除那些过早的文件，直到缓存的实际大小小于我们设置的最大使用空间。
- 对图片的解压缩操作：这一操作可以查看SDWebImageDecoder.m中+decodedImageWithImage方法的实现。
- 对GIF图片的处理
- 对WebP图片的处理  
 
参考  
[SDWebImage实现分析](http://southpeak.github.io/2015/02/07/sourcecode-sdwebimage/)  
[SDWebImage源码](https://huang303513.github.io/2017/05/01/SDWebImage源码解析(一).html)