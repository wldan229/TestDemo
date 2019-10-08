# TestDemo
Ghg
基于Selenium和ChromeDriver的自动化页面性能测试

原创 Linux操作系统 zhuyiquan90 2018-06-28 17:48:07
由于最近工作一直很紧张，拖了很久才在五一假期将Selenium实现自动化页面性能测试的代码实现部分补上，希望今后自己能更勤勉，多一些知识产出。 
Selenium WebDriver（以下简称SW）提供了一套用于Web应用程序的自动化测试工具。SW按其应用场景不同可以分为（1）基于HtmlUnit的无界面实现，并非驱动真实浏览器进行测试；（2）模拟真实输入，对多浏览器的支持和测试，包括FirefoxDriver、InternetExplorerDriver、OperaDriver和ChromeDriver；（3）对移动应用的测试，包括AndroidDriver和iPhoneDriver。 
针对SW进行功能性测试的文章和书已经很多了，比如如何操作获取页面元素内容。而本文所要写的是如何基于Selenium和ChromeDriver做页面性能测试，比如获取页面请求的加载时间、获取页面的DOM元素加载完成时间等等。类似于一些成熟的拨测产品的实现原型（这也是笔者正在做的项目）。我想这是非常有意义的一次探索。



1. Maven依赖
2、ChromeDriver使用详解
2.1、DesiredCapabilities & ChromeOptions
2.2、Performance Log
2.3、Chrome DevTools Protocol View
2.3.1、Network
Network.requestWillBeSent
Network.responseReceived
Network.loadingFailed
Network.loadingFinished
2.3.2、Page
Page.domContentEventFired
Page.loadEventFired
3、持久化ChromeDriverService的使用
4、一个应用实例的实现


1. Maven依赖

首先，项目需要引入依赖的相关selenium包：selenium-api和selenium-java，要考虑不同版本和JDK版本的兼容性，笔者是JDK 1.8。

 <dependency> <groupId>org.seleniumhq.seleniumgroupId> <artifactId>selenium-apiartifactId> <version>3.5.3version> dependency>
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


		
 <dependency> <groupId>org.seleniumhq.seleniumgroupId> <artifactId>selenium-javaartifactId> <version>3.5.3version> dependency>
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


		
2、ChromeDriver使用详解

本节内容参考https://sites.google.com/a/chromium.org/chromedriver/home，另外ChromeDriver的安装，笔者在《CentOS 7.x环境下搭建: Headless chrome + Selenium + ChromeDriver 实现自动化测试》中有详述。

2.1、DesiredCapabilities & ChromeOptions

Capabilities属性可以定义和配置你的ChromeDriver会话，以满足对应功能和需求。 
在Java实现中，类ChromeOptions和类DesiredCapabilities都可以用于具体定义Capabilities。 
比如以下代码，通过ChromeOptions来定义Chrome的window-size属性：

// 设置chromedriver路径 System.setProperty("webdriver.chrome.driver","/opt/drivers/chromedriver");

ChromeOptions options = new ChromeOptions(); // 设置chrome启动时size大小 options.addArguments("--window-size=1980,1000"); // 根据ChromeOptions实例化ChromeDriver WebDriver driver = new ChromeDriver(options); try { // 打开苏宁易购 driver.get("https://www.suning.com");
    } catch (Exception e) {
    e.printStackTrace();
    } finally { // 关闭浏览器 driver.quit();
}
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


		
当然，以上例子也可以改写为通过DesiredCapabilities来实现：

// 设置chromedriver路径 System.setProperty("webdriver.chrome.driver","/opt/drivers/chromedriver");

ChromeOptions options = new ChromeOptions(); // 设置chrome启动时size大小 options.addArguments("--window-size=1980,1000");
DesiredCapabilities cap = DesiredCapabilities.chrome();
cap.setCapability(ChromeOptions.CAPABILITY, options); // 根据DesiredCapabilities实例化ChromeDriver WebDriver driver = new ChromeDriver(cap); try { // 打开苏宁易购 driver.get("https://www.suning.com");
    } catch (Exception e) {
    e.printStackTrace();
    } finally { // 关闭浏览器 driver.quit();
}
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


			

				18
			


			

				19
			


		
2.2、Performance Log

ChromeDriver支持性能日志（Performance Log）数据的采集。想想看Chrome的F12控制台，我们能够采集到”Network”、Page”等，而这些是实现页面性能测试的基础。 
Performance Log并非是默认开启的属性，所以我们可以通过上节说的DesiredCapabilities在创建新会话的时候开启Performance Log。 
而采集到的日志，我们可以通过LogEntry对象输出到Console。具体代码实现如下：

package com.suning.webdrivertest.chromedemo; import org.openqa.selenium.WebDriver; import org.openqa.selenium.chrome.ChromeDriver; import org.openqa.selenium.logging.LogEntry; import org.openqa.selenium.logging.LogType; import org.openqa.selenium.logging.LoggingPreferences; import org.openqa.selenium.remote.CapabilityType; import org.openqa.selenium.remote.DesiredCapabilities; import java.util.logging.Level; /**
 *
 * Created by zhuyiquan90 on 2018/1/3.
 */ public class ChromeDriverDemo1 { public static void main(String[] args) { // 设置chromedriver路径 System.setProperty("webdriver.chrome.driver", "/opt/drivers/chromedriver");

        DesiredCapabilities cap = DesiredCapabilities.chrome();
        LoggingPreferences logPrefs = new LoggingPreferences(); // 启用Performance Log日志采集 logPrefs.enable(LogType.PERFORMANCE, Level.ALL);
        cap.setCapability(CapabilityType.LOGGING_PREFS, logPrefs); // 根据DesiredCapabilities实例化ChromeDriver WebDriver driver = new ChromeDriver(cap); try { // 打开苏宁易购 driver.get("https://www.suning.com"); for (LogEntry entry : driver.manage().logs().get(LogType.PERFORMANCE)) { // 输出采集到的性能日志 System.out.println(Thread.currentThread().getName() + entry.toString());
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally { // 关闭浏览器 driver.quit();
        }
    }
} 
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


			

				18
			


			

				19
			


			

				20
			


			

				21
			


			

				22
			


			

				23
			


			

				24
			


			

				25
			


			

				26
			


			

				27
			


			

				28
			


			

				29
			


			

				30
			


			

				31
			


			

				32
			


			

				33
			


			

				34
			


			

				35
			


			

				36
			


			

				37
			


			

				38
			


			

				39
			


			

				40
			


			

				41
			


			

				42
			


			

				43
			


			

				44
			


			

				45
			


			

				46
			


			

				47
			


		
其输出结果如下：

Starting ChromeDriver 2.34.522932 (4140ab217e1ca1bec0c4b4d1b148f3361eb3a03e) on port 29777 Only local connections are allowed.
四月 30, 2018 3:06:27 下午 org.openqa.selenium.remote.ProtocolHandshake createSession
信息: Detected dialect: OSS
main[2018-04-30T15:06:27+0800] [INFO] {"message":{"method":"Page.frameAttached","params":{"frameId":"49C70573CE1145CEB5B38A270213A48","parentFrameId":"28DAFE9FE90E9292F1B8EDB3315608EC","stack":{"callFrames":[{"columnNumber":240,"functionName":"","lineNumber":0,"scriptId":"21","url":""}]}}},"webview":"28DAFE9FE90E9292F1B8EDB3315608EC"}
main[2018-04-30T15:06:27+0800] [INFO] {"message":{"method":"Page.frameStartedLoading","params":{"frameId":"49C70573CE1145CEB5B38A270213A48"}},"webview":"28DAFE9FE90E9292F1B8EDB3315608EC"}
main[2018-04-30T15:06:27+0800] [INFO] {"message":{"method":"Page.frameNavigated","params":{"frame":{"id":"49C70573CE1145CEB5B38A270213A48","loaderId":"EE699DC52C8ACA226069D24DC92E16","mimeType":"text/html","name":"chromedriver dummy frame","parentId":"28DAFE9FE90E9292F1B8EDB3315608EC","securityOrigin":"://","url":"about:blank"}}},"webview":"28DAFE9FE90E9292F1B8EDB3315608EC"} 
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


		
2.3、Chrome DevTools Protocol View

这一节，我们来讲讲Network和Page包含的内容，即针对上一节输出的内容，我们如何有效利用，通过它们来计算页面性能（参考Chrome DevTools Protocol）。

2.3.1、Network

Network中我们用到的事件主要是requestWillBeSent、responseReceived、loadingFailed和loadingFinished四种：

Network.requestWillBeSent

当页面即将发送HTTP请求时触发，其Json格式为：

{
    "message": {
        "method": "Network.requestWillBeSent",
        "params": {
            "documentURL": "about:blank",
            "frameId": "C80F96297F4216E35079CFD86251AB8B",
            "initiator": {
                "lineNumber": 0,
                "type": "parser",
                "url": "https://www.suning.com/" },
            "loaderId": "58DDB2CF16600EAE484A541DF9440089",
            "redirectResponse": {
                "connectionId": 639,
                "connectionReused": false,
                "encodedDataLength": 497,
                "fromDiskCache": false,
                "fromServiceWorker": false,
                "headers": {
                    "Cache-Control": "no-cache",
                    "Connection": "keep-alive",
                    "Content-Length": "0",
                    "Date": "Mon, 30 Apr 2018 07:06:42 GMT",
                    "Expires": "Thu, 01 Jan 1970 00:00:00 GMT",
                    "Location": "https://cm.g.doubleclick.net/pixel?google_nid=ipy&google_cm",
                    "P3P": "CP=\"NON DSP COR CURa ADMa DEVa TAIa PSAa PSDa IVAa IVDa CONa HISa TELa OTPa OUR UNRa IND UNI COM NAV INT DEM CNT PRE LOC\"",
                    "Pragma": "no-cache",
                    "Server": "nginx/1.10.2",
                    "Set-Cookie": "CMBMP=IWl; Domain=.ipinyou.com; Expires=Thu, 10-May-2018 07:06:42 GMT; Path=/" },
                "headersText": "HTTP/1.1 302 Found\r\nServer: nginx/1.10.2\r\nDate: Mon, 30 Apr 2018 07:06:42 GMT\r\nContent-Length: 0\r\nConnection: keep-alive\r\nCache-Control: no-cache\r\nPragma: no-cache\r\nExpires: Thu, 01 Jan 1970 00:00:00 GMT\r\nP3P: CP=\"NON DSP COR CURa ADMa DEVa TAIa PSAa PSDa IVAa IVDa CONa HISa TELa OTPa OUR UNRa IND UNI COM NAV INT DEM CNT PRE LOC\"\r\nSet-Cookie: CMBMP=IWl; Domain=.ipinyou.com; Expires=Thu, 10-May-2018 07:06:42 GMT; Path=/\r\nLocation: https://cm.g.doubleclick.net/pixel?google_nid=ipy&google_cm\r\n\r\n",
                "mimeType": "",
                "protocol": "http/1.1",
                "remoteIPAddress": "127.0.0.1",
                "remotePort": 1086,
                "requestHeaders": {
                    "Accept": "image/webp,image/apng,image/*,*/*;q=0.8",
                    "Accept-Encoding": "gzip, deflate, br",
                    "Accept-Language": "zh-CN,zh;q=0.9",
                    "Connection": "keep-alive",
                    "Cookie": "sessionId=I4UF6b1WcgGMC; PYID=I4UF6b1Wcg99; CMTMS=p7Ik3Ve; CMSTMS=p7Ik3Ve; CMPUB=ADV-DefaultAdv; CMBMP=IW2",
                    "Host": "cm.ipinyou.com",
                    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36" },
                "requestHeadersText": "GET /baidu/cms.gif?baidu_error=1×tamp=1525072001 HTTP/1.1\r\nHost: cm.ipinyou.com\r\nConnection: keep-alive\r\nUser-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36\r\nAccept: image/webp,image/apng,image/*,*/*;q=0.8\r\nAccept-Encoding: gzip, deflate, br\r\nAccept-Language: zh-CN,zh;q=0.9\r\nCookie: sessionId=I4UF6b1WcgGMC; PYID=I4UF6b1Wcg99; CMTMS=p7Ik3Ve; CMSTMS=p7Ik3Ve; CMPUB=ADV-DefaultAdv; CMBMP=IW2\r\n",
                "securityDetails": {
                    "certificateId": 0,
                    "cipher": "AES_256_GCM",
                    "issuer": "RapidSSL SHA256 CA",
                    "keyExchange": "ECDHE_RSA",
                    "keyExchangeGroup": "P-256",
                    "protocol": "TLS 1.2",
                    "sanList": ["*.ipinyou.com", "ipinyou.com"],
                    "signedCertificateTimestampList": [{
                        "hashAlgorithm": "SHA-256",
                        "logDescription": "Symantec log",
                        "logId": "DDEB1D2B7A0D4FA6208B81AD8168707E2E8E9D01D55C888D3D11C4CDB6ECBECC",
                        "origin": "Embedded in certificate",
                        "signatureAlgorithm": "ECDSA",
                        "signatureData": "3045022024364934CBC90A8529E327E6EF853E3EF5E48B7F1598414E0F10059DC92685FC022100A74F93A8CF23D6572D7597C072368D69EC43AFB6A9EDAA4B01B43921AADEFDC2",
                        "status": "Verified",
                        "timestamp": 1511173770857.0 }, {
                        "hashAlgorithm": "SHA-256",
                        "logDescription": "Google 'Pilot' log",
                        "logId": "A4B90990B418581487BB13A2CC67700A3C359804F91BDFB8E377CD0EC80DDC10",
                        "origin": "Embedded in certificate",
                        "signatureAlgorithm": "ECDSA",
                        "signatureData": "3046022100F319D0F56F27C82228E2B01934A1C7F46915A1509F094EE91508F08C3B5AE2B2022100B0D94DD6FD00CB435EC33B916B52EC76FE5FFCC5D5BD8CB559248243AEDFE3CE",
                        "status": "Verified",
                        "timestamp": 1511173770923.0 }],
                    "subjectName": "*.ipinyou.com",
                    "validFrom": 1511136000,
                    "validTo": 1547942399 },
                "securityState": "secure",
                "status": 302,
                "statusText": "Found",
                "timing": {
                    "connectEnd": 772.852999994939,
                    "connectStart": 0.566999995498918,
                    "dnsEnd": -1,
                    "dnsStart": -1,
                    "proxyEnd": -1,
                    "proxyStart": -1,
                    "pushEnd": 0,
                    "pushStart": 0,
                    "receiveHeadersEnd": 1226.29800000141,
                    "requestTime": 42129.997749,
                    "sendEnd": 773.012999998173,
                    "sendStart": 772.960999995121,
                    "sslEnd": 772.844999999506,
                    "sslStart": 1.62599999748636,
                    "workerReady": -1,
                    "workerStart": -1 },
                "url": "https://cm.ipinyou.com/baidu/cms.gif?baidu_error=1×tamp=1525072001" },
            "request": {
                "headers": {
                    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36" },
                "initialPriority": "Low",
                "method": "GET",
                "mixedContentType": "none",
                "referrerPolicy": "no-referrer-when-downgrade",
                "url": "https://cm.g.doubleclick.net/pixel?google_nid=ipy&google_cm" },
            "requestId": "20524.247",
            "timestamp": 42131.225431,
            "type": "Image",
            "wallTime": 1525072000.35906 } },
    "webview": "28DAFE9FE90E9292F1B8EDB3315608EC" }
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


			

				18
			


			

				19
			


			

				20
			


			

				21
			


			

				22
			


			

				23
			


			

				24
			


			

				25
			


			

				26
			


			

				27
			


			

				28
			


			

				29
			


			

				30
			


			

				31
			


			

				32
			


			

				33
			


			

				34
			


			

				35
			


			

				36
			


			

				37
			


			

				38
			


			

				39
			


			

				40
			


			

				41
			


			

				42
			


			

				43
			


			

				44
			


			

				45
			


			

				46
			


			

				47
			


			

				48
			


			

				49
			


			

				50
			


			

				51
			


			

				52
			


			

				53
			


			

				54
			


			

				55
			


			

				56
			


			

				57
			


			

				58
			


			

				59
			


			

				60
			


			

				61
			


			

				62
			


			

				63
			


			

				64
			


			

				65
			


			

				66
			


			

				67
			


			

				68
			


			

				69
			


			

				70
			


			

				71
			


			

				72
			


			

				73
			


			

				74
			


			

				75
			


			

				76
			


			

				77
			


			

				78
			


			

				79
			


			

				80
			


			

				81
			


			

				82
			


			

				83
			


			

				84
			


			

				85
			


			

				86
			


			

				87
			


			

				88
			


			

				89
			


			

				90
			


			

				91
			


			

				92
			


			

				93
			


			

				94
			


			

				95
			


			

				96
			


			

				97
			


			

				98
			


			

				99
			


			

				100
			


			

				101
			


			

				102
			


			

				103
			


			

				104
			


			

				105
			


			

				106
			


			

				107
			


			

				108
			


			

				109
			


			

				110
			


			

				111
			


			

				112
			


			

				113
			


			

				114
			


			

				115
			


			

				116
			


			

				117
			


		
参数说明：

参数	类型	说明
requestId	String	唯一请求ID
loaderId	String	加载ID
documentURL	String	页面文档URL
request	Request	请求数据对象
timestamp	float	以过去某个任意时间点为基点，从打开页面开始，以秒为单位单调递增的时间戳
wallTime	float	UTC时间
initiator	Initiator	请求初始化对象
redirectResponse	Response	重定向响应对象
type	String	资源类型
frameId	String	FrameID
hasUserGesture	boolean	Whether the request is initiated by a user gesture. Defaults to false.
其中， 
Request对象：

参数	类型	说明
url	String	请求url
method	String	HTTP请求类型
headers	Object	请求头信息
postData	String	Post请求数据
hasPostData	boolean	如果是Post请求，则为true
mixedContentType	String	是否存在混淆内容问题：blockable, optionally-blockable, none.
initialPriority	String	资源加载优先级：VeryLow, Low, Medium, High, VeryHigh.
referrerPolicy	String	跨域策略：no-referrer-when-downgrade, no-referrer, origin, origin-when-cross-origin, same-origin, strict-origin, strict-origin-when-cross-origin
isLinkPreload	boolean	是否通过预加载方式加载
Response对象：

参数	类型	说明
url	String	请求url
status	int	响应状态码
statusText	String	状态码内容
headers	Object	响应头部，json格式
headersText	String	响应头部，文本格式
mimeType	String	Resource mimeType
requestHeaders	Obeject	请求头部，json格式
requestHeadersText	String	请求头部，文本格式
connectionReused	boolean	连接是否被复用
connectionId	long	物理连接ID
remoteIPAddress	String	Remote IP address
remotePort	int	Remote port
fromDiskCache	boolean	是否直接从浏览器缓存获取资源
fromServiceWorker	boolean	Specifies that the request was served from the ServiceWorker
encodedDataLength	long	响应字节数
timing	ResourceTiming	ResourceTiming对象
protocol	String	协议
securityState	String	Security state of the request resource：unknown, neutral, insecure, secure, info
securityDetails	SecurityDetails	Security details for the request
ResourceTiming对象：

参数	类型	说明
requestTime	float	时间基线
proxyStart	float	Started resolving proxy.
proxyEnd	float	Finished resolving proxy.
dnsStart	float	Started DNS address resolve.
dnsEnd	float	Finished DNS address resolve.
connectStart	float	Started connecting to the remote host.
connectEnd	float	Connected to the remote host.
sslStart	float	Started SSL handshake.
sslEnd	float	Finished SSL handshake.
workerStart	float	Started running ServiceWorker.
workerReady	float	Finished Starting ServiceWorker.
sendStart	float	Started sending request.
sendEnd	float	Finished sending request.
pushStart	float	Time the server started pushing request.
pushEnd	float	Time the server finished pushing request.
receiveHeadersEnd	float	Finished receiving response headers.
Network.responseReceived

当HTTP响应可用时触发，其Json格式为：

{
    "message": {
        "method": "Network.responseReceived",
        "params": {
            "frameId": "28DAFE9FE90E9292F1B8EDB3315608EC",
            "loaderId": "44DBCD0BEBFCEE5AED6388366BCB719B",
            "requestId": "20524.277",
            "response": {
                "connectionId": 468,
                "connectionReused": true,
                "encodedDataLength": 439,
                "fromDiskCache": false,
                "fromServiceWorker": false,
                "headers": {
                    "Cache-Control": "no-cache, max-age=0, must-revalidate",
                    "Connection": "keep-alive",
                    "Content-Length": "43",
                    "Content-Type": "image/gif",
                    "Date": "Mon, 30 Apr 2018 07:06:42 GMT",
                    "Expires": "Fri, 01 Jan 1980 00:00:00 GMT",
                    "Last-Modified": "Mon, 28 Sep 1970 06:00:00 GMT",
                    "Pragma": "no-cache",
                    "Server": "nginx/1.6.3",
                    "X-Dscp-Value": "0",
                    "X-Via": "1.1 dxun38:1 (Cdn Cache Server V2.0), 1.1 shb115:4 (Cdn Cache Server V2.0), 1.1 ls10:0 (Cdn Cache Server V2.0)" },
                "headersText": "HTTP/1.1 200 OK\r\nDate: Mon, 30 Apr 2018 07:06:42 GMT\r\nServer: nginx/1.6.3\r\nContent-Type: image/gif\r\nContent-Length: 43\r\nLast-Modified: Mon, 28 Sep 1970 06:00:00 GMT\r\nExpires: Fri, 01 Jan 1980 00:00:00 GMT\r\nPragma: no-cache\r\nCache-Control: no-cache, max-age=0, must-revalidate\r\nX-Dscp-Value: 0\r\nX-Via: 1.1 dxun38:1 (Cdn Cache Server V2.0), 1.1 shb115:4 (Cdn Cache Server V2.0), 1.1 ls10:0 (Cdn Cache Server V2.0)\r\nConnection: keep-alive\r\n\r\n",
                "mimeType": "image/gif",
                "protocol": "http/1.1",
                "remoteIPAddress": "127.0.0.1",
                "remotePort": 1086,
                "requestHeaders": {
                    "Accept": "image/webp,image/apng,image/*,*/*;q=0.8",
                    "Accept-Encoding": "gzip, deflate, br",
                    "Accept-Language": "zh-CN,zh;q=0.9",
                    "Connection": "keep-alive",
                    "Cookie": "_snstyxuid=ADFD3F4299718846; _snvd=152507199416958111",
                    "Host": "sa.suning.cn",
                    "Referer": "https://www.suning.com/",
                    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36" },
                "requestHeadersText": "GET /ajaxSiteExpro.gif?oId=152507199969277498&pvId=152507199147454663&expoInfo=index3_homepage1_32618013033_word03,index3_homepage1_32618013033_word04,index3_homepage1_newUser_tankuang&expoType=1&pageUrl=https://www.suning.com/&visitorId=&loginUserName=&memberID=-&sessionId=&pageType=web&hidUrlPattern=&iId=log_1525071999692 HTTP/1.1\r\nHost: sa.suning.cn\r\nConnection: keep-alive\r\nUser-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36\r\nAccept: image/webp,image/apng,image/*,*/*;q=0.8\r\nReferer: https://www.suning.com/\r\nAccept-Encoding: gzip, deflate, br\r\nAccept-Language: zh-CN,zh;q=0.9\r\nCookie: _snstyxuid=ADFD3F4299718846; _snvd=152507199416958111\r\n",
                "securityDetails": {
                    "certificateId": 0,
                    "cipher": "AES_256_GCM",
                    "issuer": "WoSign OV SSL CA",
                    "keyExchange": "ECDHE_RSA",
                    "keyExchangeGroup": "P-256",
                    "protocol": "TLS 1.2",
                    "sanList": ["*.suning.cn", "suning.cn"],
                    "signedCertificateTimestampList": [],
                    "subjectName": "*.suning.cn",
                    "validFrom": 1479721356,
                    "validTo": 1574329356 },
                "securityState": "secure",
                "status": 200,
                "statusText": "OK",
                "timing": {
                    "connectEnd": -1,
                    "connectStart": -1,
                    "dnsEnd": -1,
                    "dnsStart": -1,
                    "proxyEnd": -1,
                    "proxyStart": -1,
                    "pushEnd": 0,
                    "pushStart": 0,
                    "receiveHeadersEnd": 656.157999997959,
                    "requestTime": 42130.56839,
                    "sendEnd": 1.03800000215415,
                    "sendStart": 0.979999997070991,
                    "sslEnd": -1,
                    "sslStart": -1,
                    "workerReady": -1,
                    "workerStart": -1 },
                "url": "https://sa.suning.cn/ajaxSiteExpro.gif?oId=152507199969277498&pvId=152507199147454663&expoInfo=index3_homepage1_32618013033_word03,index3_homepage1_32618013033_word04,index3_homepage1_newUser_tankuang&expoType=1&pageUrl=https://www.suning.com/&visitorId=&loginUserName=&memberID=-&sessionId=&pageType=web&hidUrlPattern=&iId=log_1525071999692" },
            "timestamp": 42131.22618,
            "type": "Image" } },
    "webview": "28DAFE9FE90E9292F1B8EDB3315608EC" }
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


			

				18
			


			

				19
			


			

				20
			


			

				21
			


			

				22
			


			

				23
			


			

				24
			


			

				25
			


			

				26
			


			

				27
			


			

				28
			


			

				29
			


			

				30
			


			

				31
			


			

				32
			


			

				33
			


			

				34
			


			

				35
			


			

				36
			


			

				37
			


			

				38
			


			

				39
			


			

				40
			


			

				41
			


			

				42
			


			

				43
			


			

				44
			


			

				45
			


			

				46
			


			

				47
			


			

				48
			


			

				49
			


			

				50
			


			

				51
			


			

				52
			


			

				53
			


			

				54
			


			

				55
			


			

				56
			


			

				57
			


			

				58
			


			

				59
			


			

				60
			


			

				61
			


			

				62
			


			

				63
			


			

				64
			


			

				65
			


			

				66
			


			

				67
			


			

				68
			


			

				69
			


			

				70
			


			

				71
			


			

				72
			


			

				73
			


			

				74
			


			

				75
			


			

				76
			


			

				77
			


			

				78
			


			

				79
			


			

				80
			


			

				81
			


			

				82
			


			

				83
			


			

				84
			


		
参数说明：

参数	类型	说明
requestId	String	唯一请求ID
timestamp	float	以过去某个任意时间点为基点，从打开页面开始，以秒为单位单调递增的时间戳
type	String	资源类型
response	Response	响应对象
frameId	String	FrameID
应用场景：根据Response可以快速识别请求的各种异常状态码（5XX、4XX），以及时间的分布。 
这里写图片描述

Network.loadingFailed

当HTTP请求无法加载时触发，其Json格式为：

{
    "message": {
        "method": "Network.loadingFailed",
        "params": {
            "canceled": true,
            "errorText": "net::ERR_ABORTED",
            "requestId": "20524.271",
            "timestamp": 42130.877864,
            "type": "Image" } },
    "webview": "28DAFE9FE90E9292F1B8EDB3315608EC" }
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


		
参数说明：

参数	类型	说明
requestId	String	唯一请求ID
loaderId	String	加载ID
timestamp	float	以过去某个任意时间点为基点，从打开页面开始，以秒为单位单调递增的时间戳
type	String	资源类型
errorText	String	错误原因提示
canceled	boolean	如果请求加载被取消，则为true
blockedReason	String	请求被阻塞的原因
应用场景：我们可以通过loadingFailed和requestWillBeSent确定哪些请求加载失败。

Network.loadingFinished

当HTTP请求完成加载时触发，其Json格式为：

{
    "message": {
        "method": "Network.loadingFinished",
        "params": {
            "blockedCrossSiteDocument": false,
            "encodedDataLength": 327,
            "requestId": "20524.262",
            "timestamp": 42130.87542 } },
    "webview": "28DAFE9FE90E9292F1B8EDB3315608EC" }
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


		
参数说明：

参数	类型	说明
requestId	String	唯一请求ID
timestamp	float	以过去某个任意时间点为基点，从打开页面开始，以秒为单位单调递增的时间戳
encodedDataLength	long	响应字节数
blockedCrossSiteDocument	boolean	如果由于跨域阻塞了响应，则为true
应用场景：根据encodedDataLength，计算响应的最终大小。

这里写图片描述针对requestWillBeSent、responseReceived、loadingFailed和loadingFinished四种对象，Java构建Model如下所示：

2.3.2、Page

Page中我们用到的事件主要是domContentEventFired和loadEventFired两种：

Page.domContentEventFired

页面Dom内容加载完成时间。

{
    "message": {
        "method": "Page.domContentEventFired",
        "params": {
            "timestamp": 42124.003701 } },
    "webview": "28DAFE9FE90E9292F1B8EDB3315608EC" }
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


		
Page.loadEventFired

页面加载完成时间。

{
    "message": {
        "method": "Page.loadEventFired",
        "params": {
            "timestamp": 42133.108263 } },
    "webview": "28DAFE9FE90E9292F1B8EDB3315608EC" }
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


		
以上，我们可以根据ChromeDriver来完成对页面加载性能分析的自动化测试了。

3、持久化ChromeDriverService的使用

本节介绍ChromeDriverService，这完全是出于提高测试性能的考虑。我们知道每次创建一个ChromeDriver，完成测试以后再释放掉这个对象，等下次来了一个新的测试，仍要再新建一个对象，如此反复。这相当于每次都打开浏览器，再关闭浏览器，再打开浏览器。这种实现方式并不利于高并发的测试场景。 
我们希望如Java的池化设计思想一样，初始化生成多个持久化的浏览器对象，后面每次测试都用这些浏览器对象进行，这样会极大提升测试性能（想想看，避免了往复创建和关闭进程的过程啊！）。因此引入ChromeDriverService，ChromeDriverService是一个管理ChromeDriver server的的持久化实例：

The purpose of ChromeDriverService is to manage a persistent instance of the ChromeDriver server. 
Standard practice is to use the ChromeDriver class or the Selenium standalone server to obtain Chrome driver instances, but this practice sacrifices performance for convenience. In this scenario, each driver instance is associated with its own instance of the ChromeDriver server, which gets launched when the driver is requested and terminated when the driver exits. This per-instance server management adds overhead to test execution, both in terms of run-time and resource utilization. 
Using ChromeDriverService, this overhead can be reduced to a minimum by enabling your test framework to launch a server instance at the start of the test suite and shut it down when the suite finishes. An example of this approach can be found on the ChromeDriver Getting started page under the heading Controlling ChromeDriver’s lifetime.
其使用可以参考：Java Code Examples for org.openqa.selenium.chrome.ChromeDriverService。 
下面是我实现的一个Demo，我生成了3个线程分别持有一个ChromeDrvierService对象，相当于每个线程管理一个浏览器进程。采用阻塞队列BQ来实现生产者-消费者模式，当队列中有任务时，会分配给一个线程去进行测试。当队列中无任务时，也不会销毁ChromeDrvierService。阻塞队列的深度和线程池的大小可以根据服务器性能动态调整。

package com.suning.webdrivertest.chromedemo; import org.openqa.selenium.WebDriver; import org.openqa.selenium.chrome.ChromeDriver; import org.openqa.selenium.chrome.ChromeDriverService; import org.openqa.selenium.logging.LogEntry; import org.openqa.selenium.logging.LogType; import org.openqa.selenium.logging.LoggingPreferences; import org.openqa.selenium.remote.CapabilityType; import org.openqa.selenium.remote.DesiredCapabilities; import java.io.File; import java.util.concurrent.ArrayBlockingQueue; import java.util.concurrent.BlockingQueue; import java.util.concurrent.CountDownLatch; import java.util.logging.Level; /**
 * Created by zhuyiquan90 on 2018/1/9.
 */ public class ChromeDriverDemo2 { // 阻塞队列长度 private static final int BLOCK_QUEUE_SIZE = 100; // 浏览器driver线程数 private static final int THREAD_SIZE = 3; // chromedriver地址 private static final String chromedriverPath = "opt/drivers/chromedriver"; private static final BlockingQueue reqQuene = new ArrayBlockingQueue(BLOCK_QUEUE_SIZE); static class DriverRunnable implements Runnable { private WebDriver driver;
        CountDownLatch latch; public DriverRunnable(CountDownLatch latch) {
            ChromeDriverService chromeDriverService = new ChromeDriverService.Builder()
                    .usingDriverExecutable(new File(chromedriverPath))
                    .usingAnyFreePort()
                    .build();

            DesiredCapabilities cap = DesiredCapabilities.chrome();
            LoggingPreferences logPrefs = new LoggingPreferences();
            logPrefs.enable(LogType.PERFORMANCE, Level.ALL);
            cap.setCapability(CapabilityType.LOGGING_PREFS, logPrefs);

            driver = new ChromeDriver(chromeDriverService, cap); this.latch = latch;
        } public void run() { while (true) { try {
                    driver.get(reqQuene.take()); for (LogEntry entry : driver.manage().logs().get(LogType.PERFORMANCE)) {
                        System.out.println(Thread.currentThread().getName() + entry.toString());
                    }
                    latch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    } public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(10); for (int i = 0; i < THREAD_SIZE; i++) {
            Thread driverThread = new Thread(new DriverRunnable(latch), "driverThread" + i);
            driverThread.start();
        } for (int i = 0; i < 10; i++) {
            reqQuene.put("https://www.suning.com");
        } // 使用latch.await()的目的仅仅是为Demo的输出顺序直观，没有其他作用 // 可以去掉latch latch.await();
    }
} 
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


			

				18
			


			

				19
			


			

				20
			


			

				21
			


			

				22
			


			

				23
			


			

				24
			


			

				25
			


			

				26
			


			

				27
			


			

				28
			


			

				29
			


			

				30
			


			

				31
			


			

				32
			


			

				33
			


			

				34
			


			

				35
			


			

				36
			


			

				37
			


			

				38
			


			

				39
			


			

				40
			


			

				41
			


			

				42
			


			

				43
			


			

				44
			


			

				45
			


			

				46
			


			

				47
			


			

				48
			


			

				49
			


			

				50
			


			

				51
			


			

				52
			


			

				53
			


			

				54
			


			

				55
			


			

				56
			


			

				57
			


			

				58
			


			

				59
			


			

				60
			


			

				61
			


			

				62
			


			

				63
			


			

				64
			


			

				65
			


			

				66
			


			

				67
			


			

				68
			


			

				69
			


			

				70
			


			

				71
			


			

				72
			


			

				73
			


			

				74
			


			

				75
			


			

				76
			


			

				77
			


			

				78
			


			

				79
			


		
4、一个应用实例的实现

下面是本文的最后一部分，我想通过一个相对完整的应用实例来收官。这个实例来自于真实的应用场景，需求是采集每个页面的如下数据： 
首屏性能，包括：

首屏请求数
首屏大小
首屏DOM总数
首屏DOM加载完成时间
首屏完全加载完成时间
首屏异常响应
首屏失败响应
首屏慢响应
以及全页面性能，即打开页面后完成对整个页面的浏览，包括：

全页面请求数
全页面大小
全页面DOM总数
全页面DOM加载完成时间
全页面完全加载完成时间
全页面异常响应
全页面失败响应
全页面慢响应
最终实现如下：

package com.suning.webdrivertest.chrome; import com.alibaba.fastjson.JSON; import com.alibaba.fastjson.JSONObject; import com.suning.webdrivertest.networkdto.NetworkLoadingFailedDTO; import com.suning.webdrivertest.networkdto.NetworkLoadingFinishedDTO; import com.suning.webdrivertest.networkdto.NetworkRequestWillBeSentDTO; import com.suning.webdrivertest.networkdto.NetworkResponseReceivedDTO; import com.suning.webdrivertest.performancedto.*; import org.openqa.selenium.JavascriptExecutor; import org.openqa.selenium.WebDriver; import org.openqa.selenium.chrome.ChromeDriver; import org.openqa.selenium.chrome.ChromeDriverService; import org.openqa.selenium.chrome.ChromeOptions; import org.openqa.selenium.logging.LogEntry; import org.openqa.selenium.logging.LogType; import org.openqa.selenium.logging.LoggingPreferences; import org.openqa.selenium.remote.CapabilityType; import org.openqa.selenium.remote.DesiredCapabilities; import java.text.MessageFormat; import java.util.ArrayList; import java.util.List; import java.util.concurrent.ArrayBlockingQueue; import java.util.concurrent.BlockingQueue; import java.util.logging.Level; /**
 * Created by zhuyiquan90 on 2018/1/10.
 */ public class ChromeTest { // 阻塞队列长度 private static final int BLOCK_QUEUE_SIZE = 100; // 浏览器driver线程数 private static final int THREAD_SIZE = 1; // Fired when page is about to send HTTP request. public static final String NETWORK_REQUEST_WILL_BE_SENT = "Network.requestWillBeSent"; // Fired when HTTP response is available. public static final String NETWORK_RESPONSE_RECEIVED = "Network.responseReceived"; // Fired when HTTP request has finished loading. public static final String NETWORK_LOADING_FAILED = "Network.loadingFailed"; // Fired when HTTP request has failed to load. public static final String NETWORK_LOADING_FINISHED = "Network.loadingFinished"; // DOM Length JS public static final String JS_DOM_LENGTH = "return document.getElementsByTagName('*').length"; // ScrollingTop JS public static final String JS_SCROLLINGTOP = "return $(window).scrollTop( {0} * 1000)"; // Scrolling Y JS public static final String JS_SCROLLINGY = "return window.scrollY"; // Performance Timing JS public static final String JS_PERFORMANCE_TIMING = "return performance.timing"; private static final BlockingQueue reqQuene = new ArrayBlockingQueue(BLOCK_QUEUE_SIZE); static class DriverRunnable implements Runnable { private WebDriver driver; public DriverRunnable() {

            System.setProperty(ChromeDriverService.CHROME_DRIVER_LOG_PROPERTY,
                    System.getProperty("user.dir") + "/target/chromedriver.log");
            System.setProperty(ChromeDriverService.CHROME_DRIVER_EXE_PROPERTY,
                    System.getProperty("user.dir") + "/drivers/chromedriver");

            ChromeDriverService chromeDriverService = new ChromeDriverService.Builder()
                    .withVerbose(true)
                    .usingAnyFreePort()
                    .build();

            ChromeOptions options = new ChromeOptions(); // options.addArguments("--headless"); options.addArguments("--window-size=1980,1000");
            options.addArguments("--disable-web-security"); // options.addArguments("--start-fullscreen"); // options.addArguments("--screenshot"); // options.addArguments("--golden-screenshots-dir=" + chromedriverPath); DesiredCapabilities cap = DesiredCapabilities.chrome();
            LoggingPreferences logPrefs = new LoggingPreferences();
            logPrefs.enable(LogType.PERFORMANCE, Level.ALL);
            cap.setCapability(CapabilityType.LOGGING_PREFS, logPrefs);
            cap.setCapability(ChromeOptions.CAPABILITY, options);

            driver = new ChromeDriver(chromeDriverService, cap);
        } public void run() { while (true) { try {
                    String url = reqQuene.take();
                    TotalPerformanceDTO totalPerformance = new TotalPerformanceDTO();
                    totalPerformance.setFirstScreenPerformance(
                            detectFirstScreenPerformance(url, driver));
                    totalPerformance.setFullPagePerformance(
                            detectFullPagePerformance(url, driver));

                    System.out.println(totalPerformance.toString()); // 滑动页面,直到页面底部 //                    long scrollStart = 0, scrollEnd = 1; //                    int i = 1; //                    while (scrollStart != scrollEnd) { //                        scrollStart = (Long) js.executeScript("return window.scrollY"); //                        String scrollTo = "return $(window).scrollTop(" + i++ + " * 1000)"; //                        js.executeScript(scrollTo); //                        Thread.sleep(200); //                        scrollEnd = (Long) js.executeScript("return window.scrollY"); //                        System.out.println(scrollTo + ":" + scrollEnd); //                    } //                    System.out.println(Thread.currentThread().getName() + ": " + //                            js.executeScript("return document.getElementsByTagName('*').length")); //                    for (LogEntry entry : driver.manage().logs().get(LogType.PERFORMANCE)) { //                        System.out.println(Thread.currentThread().getName() + entry.getMessage()); //                    } } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    } /**
     * 首屏数据统计
     *
     * @param url
     * @param driver
     * @return */ private static FirstScreenPerformanceDTO detectFirstScreenPerformance(String url, WebDriver driver) {
        driver.get(url); // js操作对象 JavascriptExecutor js = (JavascriptExecutor) driver;

        FirstScreenPerformanceDTO firstScreenPerformance = new FirstScreenPerformanceDTO();
        List firstscreenRequestList = new ArrayList();
        List firstscreenResponseList = new ArrayList();
        List firstscreenFailList = new ArrayList();
        List firstscreenFinishedList = new ArrayList(); int pageRequestNum = 0; double pageSize = 0.0; for (LogEntry entry : driver.manage().logs().get(LogType.PERFORMANCE)) {
            JSONObject jsonObj = JSON.parseObject(entry.getMessage()).getJSONObject("message");
            String method = jsonObj.getString("method");
            String params = jsonObj.getString("params"); if (method.equals(NETWORK_REQUEST_WILL_BE_SENT)) {

                NetworkRequestWillBeSentDTO request = JSON.parseObject(
                        params, NetworkRequestWillBeSentDTO.class);
                pageRequestNum++; //                            System.out.println(method + ":" + request.toString()); firstscreenRequestList.add(request);
            } else if (method.equals(NETWORK_RESPONSE_RECEIVED)) {

                NetworkResponseReceivedDTO response = JSON.parseObject(
                        params, NetworkResponseReceivedDTO.class); //                System.out.println(method + ":" + response.getResponse().getUrl()); firstscreenResponseList.add(response);
            } else if (method.equals(NETWORK_LOADING_FINISHED)) {

                NetworkLoadingFinishedDTO finished = JSON.parseObject(
                        params, NetworkLoadingFinishedDTO.class);
                pageSize += finished.getEncodedDataLength(); //                            System.out.println(method + ":" + finished.toString()); firstscreenFinishedList.add(finished);
            } else if (method.equals(NETWORK_LOADING_FAILED)) {

                NetworkLoadingFailedDTO failed = JSON.parseObject(
                        params, NetworkLoadingFailedDTO.class); //                            System.out.println(method + ":" + failed.toString()); firstscreenFailList.add(failed);
            }
        } // 获取首屏DOM数 firstScreenPerformance.setPageDomNum(executeDomLengthJS(js)); // 获取首屏DOM加载完成时间 和 首屏完全加载完成时间 PerformanceTimingDTO performanceTiming = executePerformanceTimingJS(js);
        firstScreenPerformance.setDomContentLoadedCost(
                performanceTiming.getDomContentLoadedEventEnd() - performanceTiming.getConnectStart());
        firstScreenPerformance.setLoadEventCost(
                performanceTiming.getLoadEventEnd() - performanceTiming.getConnectStart()); // 获取首屏大小 firstScreenPerformance.setPageSize(pageSize / (1000 * 1000)); // 获取首屏请求数 firstScreenPerformance.setPageRequestNum(pageRequestNum);
        System.out.println("页面" + url + "：首屏请求数:" + firstScreenPerformance.getPageRequestNum() + ", 首屏大小:" + firstScreenPerformance.getPageSize() + "MB, 首屏DOM总数:" + firstScreenPerformance.getPageDomNum() + ", 首屏DOM加载完成时间:" + firstScreenPerformance.getDomContentLoadedCost() + "ms, 首屏完全加载完成时间:" + firstScreenPerformance.getLoadEventCost() + "ms"); // 分析异常响应 PageErrorsDTO pageErrorsDTO = new PageErrorsDTO();
        pageErrorsDTO.setCodeErrorResponseList(
                analysisCodeErrorResponse(firstscreenResponseList)); if (!pageErrorsDTO.getCodeErrorResponseList().isEmpty()) {
            System.out.println("首屏异常响应："); for (int i = 0; i < pageErrorsDTO.getCodeErrorResponseList().size(); i++) {
                System.out.println(pageErrorsDTO.
                        getCodeErrorResponseList().get(i).getUrl() + ": " + pageErrorsDTO.
                        getCodeErrorResponseList().get(i).getStatus());
            }
        } // 分析失败响应 pageErrorsDTO.setFailResponseList(
                analysisFailResponse(firstscreenFailList, firstscreenRequestList)); if (!pageErrorsDTO.getFailResponseList().isEmpty()) {
            System.out.println("首屏失败响应："); for (int i = 0; i < pageErrorsDTO.getFailResponseList().size(); i++) {
                System.out.println(pageErrorsDTO.
                        getFailResponseList().get(i).getUrl() + ": " + pageErrorsDTO.
                        getFailResponseList().get(i).getErrorText() + " " + pageErrorsDTO.
                        getFailResponseList().get(i).getBlockedReason());
            }
        } // 分析慢响应 pageErrorsDTO.setSlowReponseList(
                analysisSlowResponse(firstscreenRequestList, firstscreenFinishedList, 3.0)); if (!pageErrorsDTO.getSlowReponseList().isEmpty()) {
            System.out.println("首屏慢响应："); for (int i = 0; i < pageErrorsDTO.getSlowReponseList().size(); i++) {
                System.out.println(pageErrorsDTO.
                        getSlowReponseList().get(i).getUrl() + ": " + pageErrorsDTO.
                        getSlowReponseList().get(i).getCost());
            }
        }
        firstScreenPerformance.setPageErrorsDTO(pageErrorsDTO); return firstScreenPerformance;
    } /**
     * 全页面数据统计
     *
     * @param url
     * @param driver
     * @return */ private static FullPagePerformanceDTO detectFullPagePerformance(String url, WebDriver driver) {
        driver.get(url); // js操作对象 JavascriptExecutor js = (JavascriptExecutor) driver;
        FullPagePerformanceDTO fullPagePerformance = new FullPagePerformanceDTO(); // 滚动到页面底部 scrollToBottom(js);

        List fullPageRequestList = new ArrayList();
        List fullPageResponseList = new ArrayList();
        List fullPageFailList = new ArrayList();
        List fullPageFinishedList = new ArrayList(); int pageRequestNum = 0; double pageSize = 0.0; for (LogEntry entry : driver.manage().logs().get(LogType.PERFORMANCE)) {
            JSONObject jsonObj = JSON.parseObject(entry.getMessage()).getJSONObject("message");
            String method = jsonObj.getString("method");
            String params = jsonObj.getString("params"); if (method.equals(NETWORK_REQUEST_WILL_BE_SENT)) {

                NetworkRequestWillBeSentDTO request = JSON.parseObject(
                        params, NetworkRequestWillBeSentDTO.class);
                pageRequestNum++; //                System.out.println(method + ":" + request.toString()); fullPageRequestList.add(request);
            } else if (method.equals(NETWORK_RESPONSE_RECEIVED)) {

                NetworkResponseReceivedDTO response = JSON.parseObject(
                        params, NetworkResponseReceivedDTO.class); //                System.out.println(method + ":" + response.getResponse().getUrl()); fullPageResponseList.add(response);
            } else if (method.equals(NETWORK_LOADING_FINISHED)) {

                NetworkLoadingFinishedDTO finished = JSON.parseObject(
                        params, NetworkLoadingFinishedDTO.class);
                pageSize += finished.getEncodedDataLength(); //                            System.out.println(method + ":" + finished.toString()); fullPageFinishedList.add(finished);
            } else if (method.equals(NETWORK_LOADING_FAILED)) {

                NetworkLoadingFailedDTO failed = JSON.parseObject(
                        params, NetworkLoadingFailedDTO.class); //                            System.out.println(method + ":" + failed.toString()); fullPageFailList.add(failed);
            }
        } // 获取全页面DOM数 fullPagePerformance.setPageDomNum(executeDomLengthJS(js)); // 获取全页面DOM加载完成时间 和 首屏完全加载完成时间 PerformanceTimingDTO performanceTiming = executePerformanceTimingJS(js);
        fullPagePerformance.setDomContentLoadedCost(
                performanceTiming.getDomContentLoadedEventEnd() - performanceTiming.getConnectStart());
        fullPagePerformance.setLoadEventCost(
                performanceTiming.getLoadEventEnd() - performanceTiming.getConnectStart()); // 获取全页面大小 fullPagePerformance.setPageSize(pageSize / (1000 * 1000)); // 获取全页面请求数 fullPagePerformance.setPageRequestNum(pageRequestNum);
        System.out.println("页面" + url + "：全页面请求数:" + fullPagePerformance.getPageRequestNum() + ", 全页面大小:" + fullPagePerformance.getPageSize() + "MB, 全页面DOM总数:" + fullPagePerformance.getPageDomNum() + ", 全页面DOM加载完成时间:" + fullPagePerformance.getDomContentLoadedCost() + "ms, 全页面完全加载完成时间:" + fullPagePerformance.getLoadEventCost() + "ms"); // 分析异常响应 PageErrorsDTO pageErrorsDTO = new PageErrorsDTO();
        pageErrorsDTO.setCodeErrorResponseList(
                analysisCodeErrorResponse(fullPageResponseList)); if (!pageErrorsDTO.getCodeErrorResponseList().isEmpty()) {
            System.out.println("全页面异常响应："); for (int i = 0; i < pageErrorsDTO.getCodeErrorResponseList().size(); i++) {
                System.out.println(pageErrorsDTO.
                        getCodeErrorResponseList().get(i).getUrl() + ": " + pageErrorsDTO.
                        getCodeErrorResponseList().get(i).getStatus());
            }
        } // 分析失败响应 pageErrorsDTO.setFailResponseList(
                analysisFailResponse(fullPageFailList, fullPageRequestList)); if (!pageErrorsDTO.getFailResponseList().isEmpty()) {
            System.out.println("全页面失败响应："); for (int i = 0; i < pageErrorsDTO.getFailResponseList().size(); i++) {
                System.out.println(pageErrorsDTO.
                        getFailResponseList().get(i).getUrl() + ": " + pageErrorsDTO.
                        getFailResponseList().get(i).getErrorText() + " " + pageErrorsDTO.
                        getFailResponseList().get(i).getBlockedReason());
            }
        } // 分析慢响应 pageErrorsDTO.setSlowReponseList(
                analysisSlowResponse(fullPageRequestList, fullPageFinishedList, 3.0)); if (!pageErrorsDTO.getSlowReponseList().isEmpty()) {
            System.out.println("全页面慢响应："); for (int i = 0; i < pageErrorsDTO.getSlowReponseList().size(); i++) {
                System.out.println(pageErrorsDTO.
                        getSlowReponseList().get(i).getUrl() + ": " + pageErrorsDTO.
                        getSlowReponseList().get(i).getCost());
            }
        }
        fullPagePerformance.setPageErrorsDTO(pageErrorsDTO); return fullPagePerformance;
    } /**
     * 滚动到页面底部
     *
     * @param js
     * @return */ private static long scrollToBottom(JavascriptExecutor js) { long scrollStart = 0, scrollEnd = 1; int i = 1; while (scrollStart != scrollEnd) {
            scrollStart = (Long) js.executeScript(JS_SCROLLINGY);
            String scrollTo = MessageFormat.format(JS_SCROLLINGTOP, i++);
            System.out.println(scrollTo);
            js.executeScript(scrollTo); try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            scrollEnd = (Long) js.executeScript(JS_SCROLLINGY); // System.out.println(scrollTo + ":" + scrollEnd); } return scrollEnd;
    } /**
     * 执行js，获取页面DOM数
     *
     * @param js
     * @return */ private static long executeDomLengthJS(JavascriptExecutor js) { return (Long) js.executeScript(JS_DOM_LENGTH);
    } /**
     * 执行js，获取Performance Timing
     *
     * @param js
     * @return */ private static PerformanceTimingDTO executePerformanceTimingJS(JavascriptExecutor js) {
        String performance = js.executeScript(JS_PERFORMANCE_TIMING).toString();
        performance = performance.replace("unloadEventEnd=", "\"unloadEventEnd\":")
                .replace("responseEnd=", "\"responseEnd\":")
                .replace("responseStart=", "\"responseStart\":")
                .replace("domInteractive=", "\"domInteractive\":")
                .replace("domainLookupEnd=", "\"domainLookupEnd\":")
                .replace("unloadEventStart=", "\"unloadEventStart\":")
                .replace("domComplete=", "\"domComplete\":")
                .replace("domContentLoadedEventStart=", "\"domContentLoadedEventStart\":")
                .replace("domainLookupStart=", "\"domainLookupStart\":")
                .replace("redirectEnd=", "\"redirectEnd\":")
                .replace("redirectStart=", "\"redirectStart\":")
                .replace("connectEnd=", "\"connectEnd\":")
                .replace("toJSON={},", "")
                .replace("connectStart=", "\"connectStart\":")
                .replace("loadEventStart=", "\"loadEventStart\":")
                .replace("navigationStart=", "\"navigationStart\":")
                .replace("requestStart=", "\"requestStart\":")
                .replace("secureConnectionStart=", "\"secureConnectionStart\":")
                .replace("fetchStart=", "\"fetchStart\":")
                .replace("domContentLoadedEventEnd=", "\"domContentLoadedEventEnd\":")
                .replace("domLoading=", "\"domLoading\":")
                .replace("loadEventEnd=", "\"loadEventEnd\":"); //        System.out.println(performance); return JSON.parseObject(
                performance, PerformanceTimingDTO.class);
    } /**
     * 分析异常状态码的响应
     *
     * @param networkResponseReceivedList
     * @return */ private static List analysisCodeErrorResponse (List networkResponseReceivedList) {

        List codeErrorResponseList = new ArrayList(); for (NetworkResponseReceivedDTO r : networkResponseReceivedList) { if (r.getResponse().getStatus() >= 400 && r.getResponse().getStatus() <= 599) {
                CodeErrorResponseDTO codeErrorResponseDTO = new CodeErrorResponseDTO();
                codeErrorResponseDTO.setUrl(r.getResponse().getUrl());
                codeErrorResponseDTO.setStatus(r.getResponse().getStatus());
                System.out.println(r.toString());

                codeErrorResponseList.add(codeErrorResponseDTO);
            }
        } return codeErrorResponseList;
    } /**
     * 分析失败的响应
     *
     * @param networkLoadingFailedList
     * @param networkRequestWillBeSentList
     * @return */ private static List analysisFailResponse(
            List networkLoadingFailedList,
            List networkRequestWillBeSentList) {
        List failResponseList = new ArrayList(); for (NetworkLoadingFailedDTO f : networkLoadingFailedList) { for (NetworkRequestWillBeSentDTO r : networkRequestWillBeSentList) { if (f.getRequestId().equals(r.getRequestId())) {
                    FailResponseDTO failResponseDTO = new FailResponseDTO();
                    failResponseDTO.setUrl(r.getRequest().getUrl());
                    failResponseDTO.setErrorText(f.getErrorText());
                    failResponseDTO.setBlockedReason(f.getBlockedReason());

                    failResponseList.add(failResponseDTO);
                }
            }
        } return failResponseList;
    } /**
     * 分析慢响应，单位s
     *
     * @param networkRequestWillBeSentList
     * @param networkLoadingFinishedList
     * @param slowThreshold
     * @return */ private static List analysisSlowResponse(
            List networkRequestWillBeSentList,
            List networkLoadingFinishedList, double slowThreshold) {
        List slowReponseList = new ArrayList(); for (NetworkRequestWillBeSentDTO r : networkRequestWillBeSentList) { for (NetworkLoadingFinishedDTO f : networkLoadingFinishedList) { if (r.getRequestId().equals(f.getRequestId())) { double cost = f.getTimestamp() - r.getTimestamp(); if (cost >= slowThreshold) {
                        SlowReponseDTO slowReponseDTO = new SlowReponseDTO();
                        slowReponseDTO.setUrl(r.getRequest().getUrl());
                        slowReponseDTO.setCost(cost);

                        slowReponseList.add(slowReponseDTO);
                    }
                }
            }
        } return slowReponseList;
    } public static void main(String[] args) throws InterruptedException { for (int i = 0; i < THREAD_SIZE; i++) {
            Thread driverThread = new Thread(new DriverRunnable(), "driverThread" + i);
            driverThread.start();
        } for (int i = 0; i < 1; i++) {
            reqQuene.put("https://www.suning.com");
        }
    }
}
		
			
			

				1
			


			

				2
			


			

				3
			


			

				4
			


			

				5
			


			

				6
			


			

				7
			


			

				8
			


			

				9
			


			

				10
			


			

				11
			


			

				12
			


			

				13
			


			

				14
			


			

				15
			


			

				16
			


			

				17
			


			

				18
			


			

				19
			


			

				20
			


			

				21
			


			

				22
			


			

				23
			


			

				24
			


			

				25
			


			

				26
			


			

				27
			


			

				28
			


			

				29
			


			

				30
			


			

				31
			


			

				32
			


			

				33
			


			

				34
			


			

				35
			


			

				36
			


			

				37
			


			

				38
			


			

				39
			


			

				40
			


			

				41
			


			

				42
			


			

				43
			


			

				44
			


			

				45
			


			

				46
			


			

				47
			


			

				48
			


			

				49
			


			

				50
			


			

				51
			


			

				52
			


			

				53
			


			

				54
			


			

				55
			


			

				56
			


			

				57
			


			

				58
			


			

				59
			


			

				60
			


			

				61
			


			

				62
			


			

				63
			


			

				64
			


			

				65
			


			

				66
			


			

				67
			


			

				68
			


			

				69
			


			

				70
			


			

				71
			


			

				72
			


			

				73
			


			

				74
			


			

				75
			


			

				76
			


			

				77
			


			

				78
			


			

				79
			


			

				80
			


			

				81
			


			

				82
			


			

				83
			


			

				84
			


			

				85
			


			

				86
			


			

				87
			


			

				88
			


			

				89
			


			

				90
			


			

				91
			


			

				92
			


			

				93
			


			

				94
			


			

				95
			


			

				96
			


			

				97
			


			

				98
			


			

				99
			


			

				100
			


			

				101
			


			

				102
			


			

				103
			


			

				104
			


			

				105
			


			

				106
			


			

				107
			


			

				108
			


			

				109
			


			

				110
			


			

				111
			


			

				112
			


			

				113
			


			

				114
			


			

				115
			


			

				116
			


			

				117
			


			

				118
			


			

				119
			


			

				120
			


			

				121
			


			

				122
			


			

				123
			


			

				124
			


			

				125
			


			

				126
			


			

				127
			


			

				128
			


			

				129
			


			

				130
			


			

				131
			


			

				132
			


			

				133
			


			

				134
			


			

				135
			


			

				136
			


			

				137
			


			

				138
			


			

				139
			


			

				140
			


			

				141
			


			

				142
			


			

				143
			


			

				144
			


			

				145
			


			

				146
			


			

				147
			


			

				148
			


			

				149
			


			

				150
			


			

				151
			


			

				152
			


			

				153
			


			

				154
			


			

				155
			


			

				156
			


			

				157
			


			

				158
			


			

				159
			


			

				160
			


			

				161
			


			

				162
			


			

				163
			


			

				164
			


			

				165
			


			

				166
			


			

				167
			


			

				168
			


			

				169
			


			

				170
			


			

				171
			


			

				172
			


			

				173
			


			

				174
			


			

				175
			


			

				176
			


			

				177
			


			

				178
			


			

				179
			


			

				180
			


			

				181
			


			

				182
			


			

				183
			


			

				184
			


			

				185
			


			

				186
			


			

				187
			


			

				188
			


			

				189
			


			

				190
			


			

				191
			


			

				192
			


			

				193
			


			

				194
			


			

				195
			


			

				196
			


			

				197
			


			

				198
			


			

				199
			


			

				200
			


			

				201
			


			

				202
			


			

				203
			


			

				204
			


			

				205
			


			

				206
			


			

				207
			


			

				208
			


			

				209
			


			

				210
			


			

				211
			


			

				212
			


			

				213
			


			

				214
			


			

				215
			


			

				216
			


			

				217
			


			

				218
			


			

				219
			


			

				220
			


			

				221
			


			

				222
			


			

				223
			


			

				224
			


			

				225
			


			

				226
			


			

				227
			


			

				228
			


			

				229
			


			

				230
			


			

				231
			


			

				232
			


			

				233
			


			

				234
			


			

				235
			


			

				236
			


			

				237
			


			

				238
			


			

				239
			


			

				240
			


			

				241
			


			

				242
			


			

				243
			


			

				244
			


			

				245
			


			

				246
			


			

				247
			


			

				248
			


			

				249
			


			

				250
			


			

				251
			


			

				252
			


			

				253
			


			

				254
			


			

				255
			


			

				256
			


			

				257
			


			

				258
			


			

				259
			


			

				260
			


			

				261
			


			

				262
			


			

				263
			


			

				264
			


			

				265
			


			

				266
			


			

				267
			


			

				268
			


			

				269
			


			

				270
			


			

				271
			


			

				272
			


			

				273
			


			

				274
			


			

				275
			


			

				276
			


			

				277
			


			

				278
			


			

				279
			


			

				280
			


			

				281
			


			

				282
			


			

				283
			


			

				284
			


			

				285
			


			

				286
			


			

				287
			


			

				288
			


			

				289
			


			

				290
			


			

				291
			


			

				292
			


			

				293
			


			

				294
			


			

				295
			


			

				296
			


			

				297
			


			

				298
			


			

				299
			


			

				300
			


			

				301
			


			

				302
			


			

				303
			


			

				304
			


			

				305
			


			

				306
			


			

				307
			


			

				308
			


			

				309
			


			

				310
			


			

				311
			


			

				312
			


			

				313
			


			

				314
			


			

				315
			


			

				316
			


			

				317
			


			

				318
			


			

				319
			


			

				320
			


			

				321
			


			

				322
			


			

				323
			


			

				324
			


			

				325
			


			

				326
			


			

				327
			


			

				328
			


			

				329
			


			

				330
			


			

				331
			


			

				332
			


			

				333
			


			

				334
			


			

				335
			


			

				336
			


			

				337
			


			

				338
			


			

				339
			


			

				340
			


			

				341
			


			

				342
			


			

				343
			


			

				344
			


			

				345
			


			

				346
			


			

				347
			


			

				348
			


			

				349
			


			

				350
			


			

				351
			


			

				352
			


			

				353
			


			

				354
			


			

				355
			


			

				356
			


			

				357
			


			

				358
			


			

				359
			


			

				360
			


			

				361
			


			

				362
			


			

				363
			


			

				364
			


			

				365
			


			

				366
			


			

				367
			


			

				368
			


			

				369
			


			

				370
			


			

				371
			


			

				372
			


			

				373
			


			

				374
			


			

				375
			


			

				376
			


			

				377
			


			

				378
			


			

				379
			


			

				380
			


			

				381
			


			

				382
			


			

				383
			


			

				384
			


			

				385
			


			

				386
			


			

				387
			


			

				388
			


			

				389
			


			

				390
			


			

				391
			


			

				392
			


			

				393
			


			

				394
			


			

				395
			


			

				396
			


			

				397
			


			

				398
			


			

				399
			


			

				400
			


			

				401
			


			

				402
			


			

				403
			


			

				404
			


			

				405
			


			

				406
			


			

				407
			


			

				408
			


			

				409
			


			

				410
			


			

				411
			


			

				412
			


			

				413
			


			

				414
			


			

				415
			


			

				416
			


			

				417
			


			

				418
			


			

				419
			


			

				420
			


			

				421
			


			

				422
			


			

				423
			


			

				424
			


			

				425
			


			

				426
			


			

				427
			


			

				428
			


			

				429
			


			

				430
			


			

				431
			


			

				432
			


			

				433
			


			

				434
			


			

				435
			


			

				436
			


			

				437
			


			

				438
			


			

				439
			


			

				440
			


			

				441
			


			

				442
			


			

				443
			


			

				444
			


			

				445
			


			

				446
			


			

				447
			


			

				448
			


			

				449
			


			

				450
			


			

				451
			


			

				452
			


			

				453
			


			

				454
			


			

				455
			


			

				456
			


			

				457
			


			

				458
			


			

				459
			


			

				460
			


			

				461
			


			

				462
			


			

				463
			


			

				464
			


			

				465
			


			

				466
			


			

				467
			


			

				468
			


			

				469
			


			

				470
			


			

				471
			


			

				472
			


			

				473
			


			

				474
			


			

				475
			


			

				476
			


			

				477
			


			

				478
			


			

				479
			


			

				480
			


			

				481
			


			

				482
			


			

				483
			


			

				484
			


			

				485
			


			

				486
			


			

				487
			


			

				488
			


			

				489
			


			

				490
			


			

				491
			


			

				492
			


			

				493
			


			

				494
			


			

				495
			


			

				496
			


			

				497
			


			

				498
			


			

				499
			


			

				500
			


			

				501
			


			

				502
			


			

				503
			


			

				504
			


			

				505
			


			

				506
			


			

				507
			


			

				508
			


			

				509
			


			

				510
			


			

				511
			


			

				512
			


			

				513
			


			

				514
			


			

				515
			


			

				516
			


			

				517
			


			

				518
			


		
