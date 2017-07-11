# iOS下JS与OC互相调用（一）--UIWebView 拦截URL
##1.创建UIWebView，并加载本地HTML

加载本地HTML的目的是便于自己写JS调用做测试，最终肯定还是加载网络HTML。

```
self.webView = [[UIWebView alloc] initWithFrame:self.view.frame];
    self.webView.delegate = self;
    NSURL *htmlURL = [[NSBundle mainBundle] URLForResource:@"index.html" withExtension:nil];
//    NSURL *htmlURL = [NSURL URLWithString:@"http://www.baidu.com"];
    NSURLRequest *request = [NSURLRequest requestWithURL:htmlURL];

    // 如果不想要webView 的回弹效果
    self.webView.scrollView.bounces = NO;
    // UIWebView 滚动的比较慢，这里设置为正常速度
    self.webView.scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
    [self.webView loadRequest:request];
    [self.view addSubview:self.webView];
```
本地的HTML里，我定义了几个按钮，来触发调用原生的方法，然后再将执行结果回调到js 里。

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>iOS下JS与OC互相调用（一）--UIWebView 拦截URL</title>
    <script type="text/javascript">
        function loadURL(url) {
            var iFrame;
            iFrame = document.createElement("iframe");
            iFrame.setAttribute("src", url);
            iFrame.setAttribute("style", "display:none;");
            iFrame.setAttribute("height", "0px");
            iFrame.setAttribute("width", "0px");
            iFrame.setAttribute("frameborder", "0");
            document.body.appendChild(iFrame);
            // 发起请求后这个iFrame就没用了，所以把它从dom上移除掉
            iFrame.parentNode.removeChild(iFrame);
            iFrame = null;
        }

        function scanClick() {
            loadURL("haleyAction://scanClick");
        }
        function locationClick() {
            loadURL("haleyAction://locationClick");
        }
        function colorClick() {
            loadURL("haleyAction://colorClick");
        }
        function shareClick() {
            loadURL("haleyAction://shareClick?title=测试分享的标题&content=测试分享的内容&url=http://www.baidu.com");
        }
        function payClick() {
            loadURL("haleyAction://payClick");
        }
        function goBack() {
            loadURL("haleyAction://goBack");
        }
        function asyncAlert(content) {
            setTimeout(function(){
                alert(content);
            },1);
        }
        function setLocation(location) {
            asyncAlert(location);
            document.getElementById("returnValue").value = location;
        }
        function shareResult(title,content,url) {
            var str = title+content+url;
            asyncAlert(str);
            document.getElementById("returnValue").value = str;
        }

    </script>
</head>
<body>
<h1>iOS下JS与OC互相调用（一）--UIWebView 拦截URL</h1>
<input type="button" value="扫一扫" onclick="scanClick()" />
<input type="button" value="获取定位" onclick="locationClick()" />
<input type="button" value="修改背景色" onclick="colorClick()" />
<input type="button" value="分享" onclick="shareClick()" />
<input type="button" value="支付" onclick="payClick()" />
<input type="button" value="返回" onclick="goBack()" />
</body>
</html>
```

```
1.为什么自定义一个loadURL 方法，不直接使用window.location.href?
答：因为如果当前网页正使用window.location.href加载网页的同时，调用window.location.href去调用OC原生方法，会导致加载网页的操作被取消掉。
同样的，如果连续使用window.location.href执行两次OC原生调用，也有可能导致第一次的操作被取消掉。所以我们使用自定义的loadURL，来避免这个问题。
loadURL的实现来自关于UIWebView和PhoneGap的总结一文。
2.为什么loadURL 中的链接，使用统一的scheme?
答:便于在OC 中做拦截处理，减少在JS中调用一些OC 没有实现的方法时，webView 做跳转。因为我在OC 中拦截URL 时，根据scheme (即haleyAction)来区分是调用原生的方法还是正常的网页跳转。然后根据host（即//后的部分getLocation）来区分执行什么操作。
3.为什么自定义一个asyncAlert方法？
答：因为有的JS调用是需要OC 返回结果到JS的。stringByEvaluatingJavaScriptFromString是一个同步方法，会等待js 方法执行完成，而弹出的alert 也会阻塞界面等待用户响应，所以他们可能会造成死锁。导致alert 卡死界面。如果回调的JS 是一个耗时的操作，那么建议将耗时的操作也放入setTimeout的function 中。
```
##2.拦截 URL

```
#pragma mark - UIWebViewDelegate
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    NSLog(@"%@",request);
    NSURL *URL = request.URL;
    NSLog(@"%@",URL);
    NSString *scheme = [URL scheme];
    NSLog(@"%@",scheme);
    if ([scheme isEqualToString:@"haleyaction"]) {
        [self handleCustomAction:URL];
        return NO;
    }
    return YES;
}
#pragma mark - private method
- (void)handleCustomAction:(NSURL *)URL
{
    NSString *host = [URL host];
    NSLog(@"%@",host);
    if ([host isEqualToString:@"scanClick"]) {
        NSLog(@"扫一扫");
    } else if ([host isEqualToString:@"locationClick"]) {
        NSLog(@"获取定位");
        [self getLocation];
    } else if ([host isEqualToString:@"colorClick"]) {
        NSLog(@"修改背景色");
    } else if ([host isEqualToString:@"shareClick"]) {
        NSLog(@"分享");
        [self share:URL];
    } else if ([host isEqualToString:@"payClick"]) {
        NSLog(@"支付");
    } else if ([host isEqualToString:@"goBack"]) {
        NSLog(@"返回");
    }
}
- (void)getLocation
{
    // 获取位置信息
    // 将结果返回给js
    NSString *jsStr = [NSString stringWithFormat:@"setLocation('%@')",@"广东省深圳市南山区学府路XXXX号"];
    [self.webView stringByEvaluatingJavaScriptFromString:jsStr];
}
- (void)share:(NSURL *)URL
{
    NSLog(@"%@",URL.query);
    NSArray *params =[URL.query componentsSeparatedByString:@"&"];
    NSMutableDictionary *tempDic = [NSMutableDictionary dictionary];
    for (NSString *paramStr in params) {
        NSArray *dicArray = [paramStr componentsSeparatedByString:@"="];
        if (dicArray.count > 1) {
            NSString *decodeValue = [dicArray[1] stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
            [tempDic setObject:decodeValue forKey:dicArray[0]];
        }
    }
    
    NSString *title = [tempDic objectForKey:@"title"];
    NSString *content = [tempDic objectForKey:@"content"];
    NSString *url = [tempDic objectForKey:@"url"];
    // 在这里执行分享的操作
    
    // 将分享结果返回给js
    NSString *jsStr = [NSString stringWithFormat:@"shareResult('%@','%@','%@')",title,content,url];
    [self.webView stringByEvaluatingJavaScriptFromString:jsStr];
}
```
###3.OC调用JS方法

```
关于将OC 执行结果返回给JS 需要注意的是：

如果回调执行的JS 方法带参数，而参数不是字符串时，不要加单引号,否则可能导致调用JS 方法失败。比如我这样的：

NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userProfile options:NSJSONWritingPrettyPrinted error:nil];
NSString *jsonStr = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
NSString *jsStr = [NSString stringWithFormat:@"loginResult('%@',%@)",type, jsonStr];
[_webView stringByEvaluatingJavaScriptFromString:jsStr];
```
如果第二个参数用单引号包起来，就会导致JS端的loginResult不会调用。另外，利用
```
[webView stringByEvaluatingJavaScriptFromString:@"var arr = [3, 4, 'abc'];"];
```
可以往HMTL的JS环境中插入全局变量、JS方法等。


