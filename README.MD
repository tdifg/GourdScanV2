#Gourdscan v2.0

2016/09/25 update

与`第一版`Gourdscan比较

- 不同:
 - 用`redis`作为数据库而不是`mysql`。
 - `webui`改为基于`tornado`，不再需要php环境。
 - `proxy`改为`http+https`混合代理，大部分网站均可测试。
 - 使用了`python scapy`模块指定网卡抓http包，可以有效通过混杂模式获取通过本机的流量并测试，而无需设置代理。
 - 增加规则，同时可以扫描`xpath，ldap，lfi，xss`等漏洞。
- 相同: 
 - 用`sqlmapapi`进行sql注入检测。

## 启动
### 消费者（gdscan_worker）
`redis-server ./redis.conf` 启动redis  
`python sqlmapapi.py -s  -H 127.0.0.1 -p 8776`启动sqlmapapi  
`python selfscan.py` 启动工作端，从数据库获取请求并加上payload进行测试  
`之后在配置文件里填好redis与sqlmapapi的配置信息`
`在'gdscan_scapy/lib/raw2http.py'中和mix_proxy.py中存在domain域名黑名单和ext后缀黑名单，如果需要可以自行更改`
`在'gdscan_scapy/white_domain.conf'中存在域名白名单，如果不需要过滤域名，请保持其为空，如果需要过滤，用','隔开，用正则表达式过滤。比如需要过滤"*.ly.com"，应填写"(.+)\.ly\.com"`

### 生产者 (gdscan_scapy)
`python proxy_io.py` 运行代理，端口默认为10806，浏览器使用这个代理(比较稳定)  
或者使用较新的`python mix_proxy.py`(端口默认为10086，相对不太稳定，在访问https站点中经常飘红，因为不信任证书的问题，下文有QA解答，至于代理证书的安全性，可以不使用已经提供的证书，新证书的生成方法：`openssl req -new -x509 -days 365 -nodes -out key.cet -keyout key.pem`)  
如果是需要做流量检测，可以使用`python scapyscan.py`来对某个网卡或者全局进行检测。
#####注意：不要在嗅探流量的同时，在本机开启sqlmap等程序，否则这些流量也会被加入到redis中形成死循环！！！

### web界面
可以在worker所在机器上启动`python app.py --address=0.0.0.0 --port=99`#监听于99端口  


## 1、 运行逻辑
```
   gdscan_scapy 得到数据包，可能是通过代理，也可能是通过scapy
   gdscan_worker 发送payload，检测相应漏洞
   ScanerUI 通过web界面显示数据
```

## 2、 所需环境
```
    必要环境：
    pip install requests
    pip install tornado
    apt-get install redis-server
    pip install redis
    sqlmapapi
```
```
    #scapy模块独立于其他模块，如果不需要无需以下测试。
    #mac系统需要brew install python 安装另外的Python，建议使用最新的python2.7版本
    scapy所需环境：
        pip install scapy或：wget https://github.com/secdev/scapy/archive/v2.3.2.zip
            python setup.py install 
        可能需要：wget https://github.com/dugsong/libdnet/archive/master.zip
            cd python
            python setup.py install 
        可能需要：http://dfn.dl.sourceforge.net/sourceforge/pylibpcap/pylibpcap-0.6.4.tar.gz
            python setup.py install  
    #win下需要的环境都已经打包在了bin目录，win7 32位scapy测试成功。
        可能需要：http://www.microsoft.com/en-us/download/confirmation.aspx?id=44266
        

    测试是否能使用scapy：
    ^^>>> python
    Python 2.7.12 (default, Jun 29 2016, 14:05:02)
    [GCC 4.2.1 Compatible Apple LLVM 7.3.0 (clang-703.0.31)] on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    >>> from scapy.all import *
    WARNING: No route found for IPv6 destination :: (no default route?)
    >>> def capture(x):
    ...     if 'HTTP/' in x.lastlayer().original and x.lastlayer().original[0:4] != 'HTTP':
    ...             print x.lastlayer().original
    ...
    >>> sniff(iface='eth0',filter="tcp",prn=lambda x:capture(x))
    #此处的iface改成自己网卡的名字，或者不填写iface参数(默认监听所有网卡)，如果能看到下面的http包说明环境安装成功了
    GET /index.php?do=login HTTP/1.1
    Host: xssnow.com
    Connection: keep-alive
    Cache-Control: max-age=0
    Upgrade-Insecure-Requests: 1
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.103 Safari/537.36
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Accept-Encoding: gzip, deflate, sdch
    Accept-Language: zh-CN,zh;q=0.8,en;q=0.6
    
    
    GET /themes/default/css/bootstrap.min.css HTTP/1.1
    Host: xssnow.com
    Connection: keep-alive
    Cache-Control: max-age=0
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.103 Safari/537.36
    Accept: text/css,*/*;q=0.1
    Referer: http://k2y.in/index.php?do=login
    Accept-Encoding: gzip, deflate, sdch
    Accept-Language: zh-CN,zh;q=0.8,en;q=0.6
    If-None-Match: "22e7d-184dc-53474b3b53907"
    If-Modified-Since: Sat, 04 Jun 2016 14:31:59 GMT
```

##3.QA  
(1)Q：`测试https的时候，即使信任了站点，只能出来首页的文字`  
   A：`即使信任了某个站点，也不意味着信任了其子域名或其他域名，如果该站点从其他站点加载https数据，则需要将其一样信任，chrome下具体方式是打开浏览器开发者选项(f12或command+i)，点击"network"，重新刷新页面，双击红色的url，并信任该域，再次刷新重复上述步骤，直到可以完全加载出来。`  
(2)Q：`引入scapy module的时候显示缺少libdnet`  
   A：`在所需环境中有libdnet及其依赖的安装方式`  
(3)Q: win安装模块的时候提示“python version 2.7 is required, but not found”  
   A： python 64位版本在注册表中的信息与32位版本的不一样，所以在注册表里面找不到python的位置，谷歌搜索下”python version 2.7 is required“就出来了。  
(4)Q：`linux下启动redis提示“Bad directive or wrong number of arguments”`  
   A：redis版本太低，建议官网下载3.2以上版本再使用本conf文件，或者不使用提供这里的conf文件。
  
##4.To Do  
(1)优化测试url参数，更有效地减少重复。  
(2)WEB界面持续升级。  
(3)HTTP+HTTPS代理稳定性优化。  
(4)scapy资源占用改进。  
(5)伪静态的支持。  
(6)不再使用sqlmapapi，替换成减少资源占用的弱规则扫描和强化扫描精度的强规则扫描及中等规则扫描。  

##5.Thanks
感谢项目中使用的sqlmap等开源项目、为本项目开发做出贡献的【|→上善若水】童鞋、mottoin.com网站和其他对本项目有帮助的朋友。   


##6.本程序已有docker镜像  
链接: https://pan.baidu.com/s/1miLKhW8 密码: thrf  
ssh密码：    
gourdscan_admin123~  
redis:  
redis-server ~/GourdScan_v2/redis.conf  
