# HttpClientUrl
使用libcurl+openssl，实现的http请求库

在https://github.com/yutianzuo/android-curl 基础上，修改接口，改造成iOS可以使用的http请求库！
其中：建议使用YBNetwork的框架对外提供网络请求框架！


# 包括头文件等：

#define USE_LIBCUR
#ifdef USE_LIBCUR
#include "HttpClientUrl.h"

//封装调用方传递过来的callback方法：
@interface LibcurlCallback : NSObject
@property(nonatomic, copy, nullable) YBRequestProgressBlock uploadProgressCallback;
@property(nonatomic, copy, nullable) YBRequestProgressBlock downloadProgressCallback;
@property(nonatomic, copy, nullable) YBRequestCompletionBlock completionCallback;
@end

@implementation LibcurlCallback
-(id)init {
    self = [super init];
    if (self) {
        self.uploadProgressCallback = nil;
        self.downloadProgressCallback = nil;
        self.completionCallback = nil;
    }
    return self;
}

-(id)initWithUploadProgressBlock:(YBRequestProgressBlock)upload dowload:(YBRequestProgressBlock)download completion:(YBRequestCompletionBlock)completion{
    self = [super init];
    if (self) {
        self.uploadProgressCallback = upload;
        self.downloadProgressCallback = download;
        self.completionCallback = completion;
    }
    return self;
}

@end

#endif

# 发起网络请求
#ifdef USE_LIBCUR
    HttpClientUrl *client = HttpClientUrl::sharedInstance();
    
    client->setCallback(urlRequestCallback);
    const char * url =[URLString UTF8String];
    if ([URLRequest.HTTPMethod caseInsensitiveCompare:@"get"] == NSOrderedSame){
        client->get("12345", 1, url, NULL, NULL, 0, NULL, NULL, 0);
    }else{
        client->postFromData("12345", 1, url, NULL, NULL, 0, NULL, NULL, 0);
    }
    NSNumber *taskIdentifier = [[NSNumber alloc] initWithInt:client->getCurrentSeq()];
    NSLog(@"dispatch_async request %s, seq:%d End.", url, client->getCurrentSeq());
 
    LibcurlCallback *callback = [[LibcurlCallback alloc] initWithUploadProgressBlock:nil dowload:downloadProgress completion:completion];
    YBNM_TASKRECORD_LOCK(self.liburlTaskRecord[taskIdentifier] = callback;)
    
    return taskIdentifier;//;
#endif//    

//在封装的请求类中，用seq和回调方法对象字典方式记录回调对象
#ifdef USE_LIBCUR
@property (nonatomic, strong) NSMutableDictionary<NSNumber *, LibcurlCallback *> *liburlTaskRecord;
#endif


# 请求的回调方法：
#ifdef USE_LIBCUR
static void urlRequestCallback(int result,
                 const char *respones,
                 float persent,
                 size_t seq,
                 int errcode,
                 void *extra){
    NSString *resp= nil;
    if (respones != nullptr){
        resp=[NSString stringWithCString:respones  encoding:NSUTF8StringEncoding];
    }
    
    [[YBNetworkManager sharedManager] notifylibCurlCallback:result response:resp persent:persent seq:seq errcode:errcode];
    NSLog(@"urlRequestCallback,seq:%d, result:%d, responses:%s, persent:%f", seq, result, respones==nullptr?"":respones, persent);
}

- (void)notifylibCurlCallback:(int)result response:(NSString *)response persent:(float)persent seq:(int)seq errcode:(int)errcode{
    NSNumber *identifier = [[NSNumber alloc] initWithInt:seq];
    
    YBNM_TASKRECORD_LOCK(LibcurlCallback *curltask = self.liburlTaskRecord[identifier];)
    
    if (curltask){
        if(curltask.completionCallback && (persent == 0 || persent == 100)) {
            NSError *error = nil;
            if (result != 0 || errcode != 0){
                NSDictionary * userInfo = [NSDictionary dictionaryWithObject:[NSNumber numberWithInt:result] forKey:NSLocalizedDescriptionKey];
                error = [NSError errorWithDomain:@"An Error Has Occurred" code:result userInfo:userInfo];
            }
            dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
            dispatch_async(globalDispatchQueueBackground, ^{
                NSMutableDictionary *responseObj = [[NSMutableDictionary alloc] init];
                responseObj[@"error_code"] = [NSNumber numberWithInt:errcode];
                responseObj[@"response"] = response;
                curltask.completionCallback([YBNetworkResponse responseWithSessionTask:nil responseObject:responseObj error:error]);
                YBNM_TASKRECORD_LOCK([self.taskRecord removeObjectForKey:identifier];)
            });
            
        }else{
            if (curltask.downloadProgressCallback){
                
            }else if (curltask.uploadProgressCallback){
                
            }
        }
    }
}
#endif


