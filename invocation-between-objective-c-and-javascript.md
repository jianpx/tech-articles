#Objective C 和 Javascript 之间的互相调用
有些时候出于某些原因（web界面更新迭代快、工作分离），我们需要让手机里面的网页能调用Objective C的原生代码或者是反过来。
所以这时候就需要解决Objective C 和 页面代码（一般就是Javascript）之间互相调用、通信的问题了。

那么， 首先我们从简单的方面开始讲吧，Objective C如何调用Javascript代码

Objective C如何调用Javascript代码?
================================

这个很简单， UIWebView有个方法是:

```- (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script```

就能直接调用了。例如你想获取页面document的title属性，那么只需要：
```NSString *title = [webview stringByEvaluatingJavaScriptFromString:@"document.title"]];```

又或者想调用页面的一个叫`test`的函数，则只需要
```[webview stringByEvaluatingJavaScriptFromString:@"test()"]```

注意事项: JS代码占用的内存使用量不能超过10M。

Javascript如何调用Objective C代码?它们怎么通信呢?
==============================================
iOS里面加载一个网页用的是UIWebView，而关于页面加载的情况是通过UIWebView的一个Delegate：[UIWebViewDelegate](https://developer.apple.com/library/ios/documentation/uikit/reference/UIWebViewDelegate_Protocol/Reference/Reference.html)来通知
对应的webview的。而每次点击页面上的链接（或者是加载本页面的地址时）都会在加载前调用UIWebViewDelegate的一个方法：
```- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType```
如果这个方法的返回值是YES的话就继续加载这个请求，如果是NO的话就不加载了。
所以Javascript调用Objective C代码的秘诀就在这个方法里面了。

Step 1. 匹配url格式
---------------------

伪代码一般如下：

    - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:   (UIWebViewNavigationType)navigationType
    {
      if (request.URL.absoluteString match urlSchemePattern) {
        [self executeSomeObjectiveCCode];
        return NO;
      } else {
        return YES;
      }
    }


```request.URL.absoluteString match urlSchemePattern``` 这里就是如果页面的url的格式是满足某种特定格式的话就不加载那个请求，而是执行Objective C的代码。

Step 2. 协商url格式以及参数传递方式
------------------------------------

现在很明显的是， 一般Javascript想要调用Objective C代码时，Javascript代码就需要和Objective C协商一个请求的协议，例如：凡是请求的url scheme
是```"js-call://"``` 这样格式开头的就是Javascript需要调用Objective C的代码，再具体点，比如"js-call://user/get" 就是要调用Objective C
代码中一个getUser的方法的。（这里的js-call只是样例，实际中你可以自定义其他的字符串和格式，例如myjs:///也是可以的）
那么如果Javascript需要传递参数给Objective C呢？ 很自然的，这里我们想到最简单的方法是像http的query string一样传参数（网上也有实现用json传的），
例如:"js-call://user/set?uid=1&name=jpx"，然后在分析url的时候将query string提取出来传给Objective C的方法即可。
伪代码如下:

    - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
    {
      if ([request.URL.absoluteString hasPrefix:@"js-call://user/set"]) {
        NSDictionary *parameters = [self parseQueryString:request.URL.absoluetString];
        [self executeSomeObjectiveCCodeWithParameters:parameters];
        return NO;
      } else if ([request.URL.absoluteString hasPrefix:@"js-call://user/get"]) {
        NSDictionary *parameters = [self parseQueryString:request.URL.absoluetString];
        [self executeSomeObjectiveCCodeWithParameters:parameters];
        return NO;
      }
      return YES;
    }

OK, 那么现在Javascript调用Objective C的方法就讲完了。不过认真思考的同学可能会想到，如果我的Javascript需要调用好几个
Objective C的接口，那么在shouldStartLoadWithRequest的delegate方法里面不就很多if ... elif ... else ...的代码！！！
而且解析query string的那部分代码也是重复的！！！别的ViewController要实现这样的一套Javascript调用Objective C机制时
有得重复写这些代码？！So 作为程序员的我，也思考了这样的问题，我的解决办法是将这一切封装起来，于是[JPXUIWebViewJSBridge](https://github.com/jianpx/JPXUIWebViewJSBridge)就诞生了。

介绍JPXUIWebViewJSBridge
-------------------------
由于我不想对url的格式分析是很多的if else 代码，而且希望这部分代码能重用。所以我参考了以前用过的某Web框架的url dispatch的机制优化。
大概的思想就是我们定义url格式应该匹配到哪个cgi handler，大概是这样的：

    //r'login' 就是正则表达式匹配请求的url的，后面的login对应的是cgi handler的login函数。
    url_config = [
         (r'login', 'login'),
         (r'user/set', 'update_user'),
     ]

所以我的JPXUIWebViewJSBridge也是可以这样做的：

    self.bridge = [[JPXUIWebViewJSBridge alloc] initWithHandler:self];
    self.bridge.routines = @[@[@"^js-call://user/set.*$", @"setUser"],
                             @[@"^js-call://user/get.*$", @"getUser"]
                             ];

是不是很方便呢？定义了这套规则之后，只需要比如说在ViewController里面实现一个叫setUser的方法即可：
```- (void)setUser:(NSDictionary *)parametersFromWeb``` , 其中parametersFromWeb就是query string对应的字典！

然后在
```- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType```
只需要这样写就可以了:

    - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
    {
        NSError *error;
        BOOL canHandleRequest = [self.bridge canHandleRequest:request error:&error];
        if (canHandleRequest) {
            [self.bridge handleRequest:request error:&error];
            NSLog(@"error1:%@", [error localizedDescription]);
            return NO;
        } else {
            NSLog(@"error2:%@", [error localizedDescription]);
        }
        return YES;
    }

具体可以看这里的[Demo](https://github.com/jianpx/JPXUIWebViewJSBridge/tree/master/Demo) 。 欢迎有需要的同学使用！


一些注意事项
--------------
Javascript调用Objective C时，很多人第一反应就是在a标签里面的href写url调用，例如:
```<a href="js-call://user/set?uid=1&name=jpx" >测试</a>```, 但是这样的调用会如下的一些问题：
```
There is weird but apprehensible bugs with this practice:
a lot of javascript/html stuff get broken when we cancel a location change:
```

* All setInterval and setTimeout immediatly stop on location change
* Every innerHTML won’t work after a canceled location change!
* You may get other really weird bugs, really hard to diagnose...

而更加合理的做法应该是通过加载一个iframe：

    function execute(url)
    {
         var iframe = document.createElement("IFRAME");
         iframe.setAttribute("src", url);
         document.documentElement.appendChild(iframe);
         iframe.parentNode.removeChild(iframe);
         iframe = null;
    }

Finally, Have Fun! Guys:)
