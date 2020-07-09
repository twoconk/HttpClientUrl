# HttpClientUrl
使用libcurl+openssl (交叉编译openssl和libcurl，可以参考：https://github.com/jasonacox/Build-OpenSSL-cURL 或者直接使用文华编译好的库 )，实现的http请求库 

在https://github.com/yutianzuo/android-curl 基础上，修改接口，改造成iOS可以使用的http请求库！
其中：建议使用YBNetwork的框架对外提供网络请求框架！


# 包括头文件等：
```c
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
```
    
# 发起网络请求
```c
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
```

# HttpClientUrl头文件和.mm文件的实现，其他文件可以直接参考android的库，目录结构不变
```c
//
//  HttpClientUrl.h
//  HttpClientUrl
//  
//

#ifndef HTTPCLIENTURL__
#define HTTPCLIENTURL__
#include <stdio.h>
#include <stdlib.h>

typedef void (*CALLBACK)(int result,
                         const char *respones,
                         float persent,
                         size_t seq,
                         int errcode,
                         void *extra);

class HttpClientUrl{
private:
    CALLBACK mCallback;
    static volatile unsigned int g_req_seq;
private:
    HttpClientUrl(){
        mCallback = NULL;
        init(5, NULL);
    };
    ~HttpClientUrl(){
        unInit();
    };
    void init(int threadPoolSize, CALLBACK callBack);
    void unInit() ;
public:
    static HttpClientUrl* sharedInstance(){
        static HttpClientUrl globalHttpClientUrl;
        return &globalHttpClientUrl;
    }
public:
    void setCallback(CALLBACK callback){
        mCallback = callback;
    }
    CALLBACK getCallback(){
        return mCallback;
    }
    int getCurrentSeq(){return g_req_seq;}
    void addBasicHeader(const char * strHash_,
                        const char * strKey_,
                        const char * strValue_);
    void addBasicURLParam(const char * strHash_, const char * strKey_, const char * strValue_) ;
    void setHost(const char * strHash_, const char * strHost_);
    void get(const char * strHash_,int requestSeq,
             const char * strPath_,
             const char ** headers_keys,
             const char ** headers_values,
             int hearder_length,
             const char ** params_keys,
             const char ** params_values,
             int params_length) ;
    void setCertPath(const char * strHash_, const char * strCertPath_);
    void setProxy(const char * strHash_, const char * proxy_) ;
    void postFromData(const char * strHash_,
                      int requestSeq,
                      const char * strPath_,
                      const char ** headers_keys,
                      const char ** headers_values,
                      int hearder_length,
                      const char ** params_keys,
                      const char ** params_values,
                      int params_length);
    void postJson(const char * strHash_,
                  int requestSeq,
                  const char * strPath_,
                  const char ** headers_keys,
                  const char ** headers_values,
                  int hearder_length,
                  const char * strJson_) ;
    void putJson(const char * strHash_,
                 int requestSeq,
                 const char * strPath_,
                 const char ** headers_keys,
                 const char ** headers_values,
                 int hearder_length,
                 const char * strJson_) ;
    void postFile(const char * strHash_,
                  int requestSeq,
                  const char * strPath_,
                  const char ** headers_keys,
                  const char ** headers_values,
                  int hearder_length,
                  const char * strFormName_,
                  const char ** params_keys,
                  const char ** params_values,
                  int params_length,
                  const char * strJsonName_,
                  const char * strJson_,
                  const char * strFileKeyName_,
                  const char * strFilePath_,
                  const char * strFileName_) ;
    void download(const char * strHash_,
                  int requestSeq,
                  const char * strPath_,
                  const char ** headers_keys,
                  const char ** headers_values,
                  int hearder_length,
                  const char ** params_keys,
                  const char ** params_values,
                  int params_length,
                  const char * strFilePath_);
};

#endif


//HttpClientUrl.mm

#include <stdlib.h>
#include <stdio.h>
#include <string>

#include "manager/httpmanager.h"
#include "sha.h"
#include "aes_cbc.h"

#include "HttpClientUrl.h"
#import <Foundation/Foundation.h>

#ifdef DEBUG
#define DLog(FORMAT, ...) fprintf(stderr,"%s:%d\t%s\n",[[[NSString stringWithUTF8String:__FILE__] lastPathComponent] UTF8String], __LINE__, [[NSString stringWithFormat:FORMAT, ##__VA_ARGS__] UTF8String]);
#endif


volatile unsigned HttpClientUrl::g_req_seq = 1;

void HttpClientUrl::init(int threadPoolSize, CALLBACK callBack) {
    DLog(@"[curl] init, threadPoolSize:%d", threadPoolSize);
    HttpManager::init(threadPoolSize);
}

void HttpClientUrl::unInit() {
    DLog(@"[curl] unInit");
    HttpManager::uninit();
}

void HttpClientUrl::addBasicHeader(const char * strHash_,
                    const char * strKey_,
                    const char * strValue_) {
    const char *strHash = strHash_;
    const char *strKey = strKey_;
    const char *strValue = strValue_;
    
    if (strHash && strKey && strValue) {
        DLog(@"[curl] addBasicHeader, hash:%s, key:%s, value:%s", strHash, strKey, strValue);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->add_basic_headers(strKey, strValue);
    }
}

void HttpClientUrl::addBasicURLParam(const  char * strHash_, const char * strKey_, const char * strValue_) {
    const char *strHash = strHash_;
    const char *strKey = strKey_;
    const char *strValue = strValue_;

    if (strHash && strKey && strValue) {
        DLog(@"[curl] addBasicURLParam, hash:%s, key:%s, value:%s", strHash, strKey, strValue);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->add_basic_url_params(strKey, strValue);
    }
}

void HttpClientUrl::setHost(const char * strHash_, const char * strHost_) {
    const char *strHash = strHash_;
    const char *strHost = strHost_;

    if (strHash && strHost) {
        DLog(@"[curl] addBasicURLParam, hash:%s, strHost_:%s\n", strHash, strHost);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->set_host(strHost);
    }
}

using HEADERMAP = std::map<std::string, std::string>;

static HEADERMAP gen_map(const char ** headers_keys, const char ** headers_values, int hearder_length) {
    HEADERMAP map_ret;
    if (hearder_length <= 0 || headers_keys == NULL || headers_values == NULL || *headers_keys == NULL || *headers_values == NULL){
      return map_ret;
    }
    
    for (int i= 0; i<hearder_length; i++){
        map_ret[headers_keys[i]] = headers_values[i];
    }
    return map_ret;
}

static void GlobalCallBackFunc(int result,
                               const std::string &respones,
                               float persent,
                               size_t seq,
                               int errcode,
                               void *extra) {
    DLog(@"[curl] GlobalCallBackFunc seq:%d, result:%d:%s\n", seq, result, respones.c_str());
    CALLBACK callback = HttpClientUrl::sharedInstance()->getCallback();
    if (callback != NULL){
        callback(result, respones.c_str(), persent, seq, errcode, extra);
    }
}

void HttpClientUrl::get(const char * strHash_,int seq,
         const char * strPath_,
         const char ** headers_keys,
         const char ** headers_values,
         int hearder_length,
         const char ** params_keys,
         const char ** params_values,
         int params_length) {

    const char *strHash = strHash_;
    const char *strPath = strPath_;

    if (strHash && strPath) {
        DLog(@"[curl] get, hash:%s, strPath:%s", strHash, strPath);
        HEADERMAP inner_headers = gen_map(headers_keys, headers_values, hearder_length);
        HEADERMAP inner_url_params = gen_map( params_keys, params_values, params_length);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->get(strPath, inner_headers, inner_url_params, GlobalCallBackFunc, ++g_req_seq);
        DLog(@"[curl] get end, hash:%s, strPath:%s", strHash, strPath);
    }
}

void HttpClientUrl::setCertPath(const char * strHash_, const char * strCertPath_)
{
    const char *strHash = strHash_;
    const char *strCertPath = strCertPath_;

    if (strHash && strCertPath) {
        DLog(@"[curl] setCertPath, hash:%s, strPath:%s", strHash, strCertPath);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->set_cert_path(strCertPath);
    }
}

void HttpClientUrl::setProxy(const char * strHash_, const char * proxy_) {
    const char *strHash = proxy_;
    const char *proxy = strHash_;


    if (strHash && proxy) {
        DLog(@"[curl] setProxy, hash:%s, proxy:%s", strHash, proxy);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->set_proxy_path(proxy);
    }
}

void HttpClientUrl::postFromData(const char * strHash_,
                  int seq,
                  const char * strPath_,
                  const char ** headers_keys,
                  const char ** headers_values,
                  int hearder_length,
                  const char ** params_keys,
                  const char ** params_values,
                  int params_length) {
    const char *strHash = strHash_;
    const char *strPath = strPath_;


    if (strHash && strPath) {
        DLog(@"[curl] postFromData, hash:%s, strPath:%s", strHash, strPath);
        HEADERMAP inner_headers = gen_map( headers_keys, headers_values, hearder_length);
        HEADERMAP inner_url_params = gen_map( params_keys, params_values, params_length);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->post_form(strPath, inner_headers, inner_url_params, GlobalCallBackFunc, ++g_req_seq);
    }
}

void HttpClientUrl::postJson(const char * strHash_,
                             int seq,
                             const char * strPath_,
                             const char ** headers_keys,
                             const char ** headers_values,
                             int hearder_length,
                             const char * strJson_) {
    const char *strJson = strJson_;
    const char *strHash = strHash_;
    const char *strPath = strPath_;


    std::string str_json;
    if (strJson) {
        str_json = strJson;
    }


    if (strHash && strPath) {
        DLog(@"[curl] postJson, hash:%s, strPath:%s", strHash, strPath);
        HEADERMAP inner_headers = gen_map( headers_keys, headers_values, hearder_length);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->post_json(strPath, inner_headers, str_json, GlobalCallBackFunc, ++g_req_seq);
    }
}

void HttpClientUrl::putJson(const char * strHash_,
             int seq,
             const char * strPath_,
             const char ** headers_keys,
             const char ** headers_values,
             int hearder_length,
             const char * strJson_) {
    const char *strJson = strJson_;
    const char *strHash = strHash_;
    const char *strPath = strPath_;
 
    std::string str_json;
    if (strJson) {
        str_json = strJson;
    }

    if (strHash && strPath) {
        DLog(@"[curl] putJson, hash:%s, strPath:%s", strHash, strPath);
        HEADERMAP inner_headers = gen_map( headers_keys, headers_values, hearder_length);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->put(strPath, str_json, inner_headers, GlobalCallBackFunc, ++g_req_seq);
    }

}

void HttpClientUrl::postFile(const char * strHash_,
              int seq,
              const char * strPath_,
              const char ** headers_keys,
              const char ** headers_values,
              int hearder_length,
              const char * strFormName_,
              const char ** params_keys,
              const char ** params_values,
              int params_length,
              const char * strJsonName_,
              const char * strJson_,
              const char * strFileKeyName_,
              const char * strFilePath_,
              const char * strFileName_) {
    const char *strHash = strHash_;
    const char *strPath = strPath_;
    const char *strFormName = strFormName_;
    const char *strJsonName = strJsonName_;
    const char *strJson = strJson_;
    const char *strFileKeyName = strFileKeyName_;
    const char *strFilePath = strFilePath_;
    const char *strFileName = strFileName_;

    std::string str_formname;
    std::string str_jsonname;
    std::string str_json;
    std::string str_filekeyname;
    std::string str_filename;
    std::string str_filepath;

    if (strFormName) {
        str_formname = strFormName;
    }
    if (strJsonName) {
        str_jsonname = strJsonName;
    }
    if (strJson) {
        str_json = strJson;
    }
    if (strFileKeyName) {
        str_filekeyname = strFileKeyName;
    }
    if (strFilePath) {
        str_filepath = strFilePath;
    }
    if (strFileName) {
        str_filename = strFileName;
    }

    if (strHash && strPath) {
        DLog(@"[curl] postFile, hash:%s, strPath:%s", strHash, strPath);
        HEADERMAP inner_headers = gen_map( headers_keys, headers_values, hearder_length);
        HEADERMAP inner_form_params = gen_map( params_keys, params_values, params_length);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->post_file(strPath, inner_headers, str_formname, inner_form_params, str_jsonname,
                     str_json,
                     str_filekeyname, str_filename, str_filepath, GlobalCallBackFunc,
                     ++g_req_seq);
    }

}

void HttpClientUrl::download(const char * strHash_,
              int seq,
              const char * strPath_,
              const char ** headers_keys,
              const char ** headers_values,
              int hearder_length,
              const char ** params_keys,
              const char ** params_values,
              int params_length,
              const char * strFilePath_) {
    const char *strFilePath = nullptr;
    const char *strHash = nullptr;
    const char *strPath = nullptr;

    std::string str_filepath;
    if (strFilePath) {
        str_filepath = strFilePath;
    }

    if (strHash && strPath) {
        DLog(@"[curl] download, hash:%s, strPath:%s", strHash, strPath);
        HEADERMAP inner_headers = gen_map( headers_keys, headers_values, hearder_length);
        HEADERMAP inner_url_params = gen_map( params_keys, params_values, params_length);
        RequestManager *p = HttpManager::get_instance()->get_request_manager(strHash);
        p->download(strPath, inner_headers, inner_url_params, str_filepath, GlobalCallBackFunc,
                    ++g_req_seq);
    }


}
 

```

# 修改request.h

```C
    // modify skip_ssl from false to  true.
    void set_url(const std::string &str_url, bool skip_ssl = true)
   
```
