---
title: 一个免费ss网站的数据爬取过程
categories: 科学上网
tags: [SS, Python]
abbrlink: 656a7ed5
date: 2019-04-14 16:47:41
---


# 引言

偶然发现一个免费ss分享网站，本以为简单的url请求即可获取数据。但是没想到在网站的反爬机制很严格，这反而激起了我的好奇心。

不过对于爬虫经验技术来说，是一个很好的学习检验的机会。



# 爬虫整体概况

数据获取的核心是围绕2个**http请求**：

|  方法    |   地址   |  参数   |   备注   |
| :----:|:----:|:----:|----|
|  get    |  url   |   无   |  响应源码中有可用信息   |
|   post  |  url1  | a,b,c  |    返回加密数据文本  |

![](http://ww1.sinaimg.cn/large/9d89d613ly1g24pnzo80gj20pa0oywhi.jpg)

# 主要功能方法

## 绕过DDOS保护（Cloudflare）

简单来讲cloudflare就是通过js验证访问是否来至真正的web浏览器。

![](http://ww1.sinaimg.cn/large/9d89d613ly1g24q32f71mj20hi05zaag.jpg)

如上图所示，访问url 时候一般会进入5秒左右倒计时；请求状态码是503。
通过抓包可以看到503请求数秒后跳转一个类似“https://xxxxx/cdn-cgi/l/chk_jschl?s=xxx”的302请求，成功后即可跳转到期望页面。

**解决方法：**使用任意一个第三方库：cloudflare-scrape 或者 **cloudflare-scrape-js2py**
这2个库方法通用，安装cloudflare-scrape-js2py方法如下：
```sh
$ git clone https://github.com/VeNoMouS/cloudflare-scrape-js2py.git
$ sudo python3 setup.py install
```

由于Cloudflare不断改变和强化其保护页面，最初使用的 cloudflare-scrape很快就不能用了，当然可以去提交问题或者查看相关问题的解决方案。
我也是各种尝试各种查，不过最后换了 cloudflare-scrape-js2py就解决了问题。

主要代码如下：
```python
session = requests.session()
scraper = cfscrape.create_scraper(sess=session)
scraper = cfscrape.create_scraper(delay=11)

req = scraper.get(url)
html = req.text
```

## post中参数a,b,c的解析
get->url 返回源码如下图（部分）：
![](http://ww1.sinaimg.cn/large/9d89d613ly1g24qcdqgfgj20v40ilq9l.jpg)

### post中参数a,b,c的解析

网页源码js中的参数a,b,c
解析a,b,c参数主要是依赖正常显示的url的源码中的js代码，如下图重点地方以划出。
上图第一个被划线的一句js如下：
```js
if (detect) {
    var a = '317d2cefb40586a9';
    var b = 'e1a5f5499211cbd4';
    var c = 'e59ab31720d6cf48';
} else {
    var b = 'e59ab31720d6cf48';
    var c = '317d2cefb40586a9';
    var a = 'e1a5f5499211cbd4';
}
```

根据多次观察，可知每次需要取else中的a,b,c的值，代码如下：
```python
# 获取页面源码中else{后面的abc的值
# 参数值只能为: 'a' ,'b', 'c'
def get_var_abc(key='a'):
    # 反向查找 并截取
    ivs = key + "='"
    ivl = html.rindex(ivs)
    key = html[ivl + len(ivs): ivl + len(ivs) + 16]
    return key
```
二维码（base64文本）解码
上图中*docodeImage*方法中第一个参数值如下：

`data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFcAAABXAQMAAABLBksvAAAABlBMVEX///8AAABVwtN+AAAACXBIWXMAAA7EAAAOxAGVKw4bAAAAq0lEQVQ4jc3Ruw0DIQwGYEcu3B0LRGINOmbKAgm3AKxExxpIXiDXUURxjkgnXXExKe+vPjfgB8DZYkQisEhVPAEGwgCqPUePqQ2cMo8tOPAf/fT+3W6WI6+5bPhlIxjd1bSqePIv0+TtNAPhUjCJ5vW1R5ZIukUaLw0U97LYuWhe9xDIBtLc9+nq3VXNnufMEQaODm7ffzUTL1k1YCr83G596H5fstJA8bnyAVNpDVwJjnB4AAAAAElFTkSuQmCC`

这个参数值相当于一个图片的base64编码，其实就是一个二维码。
这个二维码是一个长度为16的字符，其实也是post中的一个参数值。

代码如下：
```python
# 直接通过base64字符串解析二维码；获取二维码的值
def decode_qrcode(html):
    qrdatas = html.split('data:image/png;base64,')[1].split("',")[0]
    qrbytes = bytes(qrdatas, encoding="ascii")
    decode_result = decode(Image.open(BytesIO(codecs.decode(qrbytes,'base64'))))
    qrvalue =str(decode_result[0].data, encoding='utf-8')
    return qrvalue
```

### post参数a,b,c值的确定
上图中的一段省略版的js代码如下：
```python
decodeImage('data:image/png;base64,iVBORw0KGxx...xxwJjnB4AAAAAElFTkSuQmCC',
    function(c) {
        $.post("data.php",{a:a,b:b,c:encc(c)},
```

多次观察发现*function(key)*方法中的参数随机a,b,c任一，而这个随机参数(a/b/c)需要用**二维码的值**来替代。
**说明：post中的a,b,c的值是依赖js中的a,b,c的值和二维码的值**，所以a,b,c的值要注意以免混淆。

代码如下：
```python
# 根据规则分别求出3种情况下的a,b,c
def get_param_abc(html):
    a = get_var_abc(key='a')
    b = get_var_abc(key='b')
    c = get_var_abc(key='c')
    symbolKey = html.split('data:image/png;base64,')[1].split("function(")[1][:1]
    print('symbolKey',symbolKey)
    print(symbolKey=='a',symbolKey=='b',symbolKey=='c')
    if symbolKey == 'a':
        a = decode_qrcode(html)
    elif symbolKey == 'b':
        b = decode_qrcode(html)
    elif symbolKey == 'c':
        c = decode_qrcode(html)

    # js2py.translate_file('encc.min.js', 'encc.py')
    from encc import PyJsHoisted_encc_
    c = PyJsHoisted_encc_(c).value
    print('\n%s\n%s\n%s'% (a,b,c))
    return a,b,c
```

### post参数c的值的加密

```js
$.post("data.php", {
    a: a,
    b: b,
    c: encc(c)
},
```

这一行是js中post的请求，是固定的对参数c进行加密，网站中的js加密算法比较清晰。
这时的思路有3种：
1. 直接通过python执行js代码；
2. 第三方（库）自动代码转换（ js->python ）；
3. 根据js中的加密思想，手动进行python编码；

比较推荐前2种方法，毕竟第3种手动转码太难太耗时间。
这里推荐2个执行js的库：**Js2Py**，PyExeJs
本文使用的是Js2Py，这个强大的库可以直接执行js代码和转换js文件。

所以这行js对应的python代码为：
```js
a,b,c = get_param_abc(html)

#post提交a,b,c,得到密文
payload = {'a':a,'b':b,'c':c}
session.cookies = scraper.cookies  # 必须带上cookies
req = scraper.post(url2,data=payload)
msg = req.text
```

## AES加密数据解码
### 确定AES加密模式（弃用）
最初是想用判断语句指定加密模式参数，但有幸后来看到大牛的好方法。
不过有一个小点需要说明一下，就是可以从js源码中分析出来加密参数；
如上节图上被括起来的js文本如下：

```js
//js eval加密文本
eval(function(p,a,c,k,e,d){e=function(c){return(c<a?"":e(parseInt(c/a)))+((c=c%a)>35?String.fromCharCode(c+29):c.toString(36))};if(!''.replace(/^/,String)){while(c--)d[e(c)]=k[c]||e(c);k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1;};while(c--)if(k[c])p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c]);return p;}('1 2=\'4\'+\'h\'+\'8\'+\'9\'+\'a\';1 5=\'3\'+\'6\';1 7=0.b.g(d,i,{c:e,2:0.2.4,f:0.5.3});',19,19,'CryptoJS|var|mode|Pkcs7|CBC|pad|ZeroPadding|dec|CTR|ECB|OFB|AES|iv||y|padding|decrypt|CFB|x'.split('|'),0,{}))

//js 解密（解压缩）后的文本
var mode = 'CBC' + 'CFB' + 'CTR' + 'ECB' + 'OFB';
var pad = 'Pkcs7' + 'ZeroPadding';
var dec = CryptoJS.AES.decrypt(d, x, {
    iv: y,
    mode: CryptoJS.mode.CBC,
    padding: CryptoJS.pad.Pkcs7
});
```


### 免判断加密模式并解密（推荐）
中间和朋友电话闲聊中问过一句模式的选择，朋友只是说模式不重要；当时就觉得应该有一种遍历类型的解法。

根据js代码可知：下面python代码的*decrypt_data*方法的参数*key，iv*分别对应post参数的*a，b*；即：
```js
endata = base64.b64decode(msg)
key = bytes(a,encoding="utf-8")
iv  = bytes(b,encoding="utf-8")

ssdata = decrypt_data(key, iv, endata)
```
由此对应的python代码如下：

```python
"""解密数据得到ss信息，返回list对象"""
def decrypt_data(key, iv, endata):
    ctr = Counter.new(128, initial_value=int(binascii.hexlify(iv), 16))
    modes = [
        AES.new(key, AES.MODE_ECB),
        AES.new(key, AES.MODE_CBC, iv),
        AES.new(key, AES.MODE_OFB, iv),
        AES.new(key, AES.MODE_CTR, counter=ctr),
        AES.new(key, AES.MODE_CFB, iv, segment_size=128)
    ]
    rtdatas = None
    for mode in modes:
        try:
            dec = mode.decrypt(endata).decode('utf-8')
            print(dec[-200:])     # 解密后文本结尾可能存在'\0'
            dec = dec.rstrip('\0')  # 去除结尾空白符'\0'
            rtdatas = json.loads(dec)['data']
            break
        except:
            pass
    return rtdatas
```
网站的加密模式是5种的随机的一种，其中可能遇到CFB模式中会报错，后来发现pycryto的版本问题，安装pycrytodome即可。

### 解码数据并测延时
post密文解密后是json的文本，下图是格式化片段：

![](http://ww1.sinaimg.cn/large/9d89d613ly1g24qxyazfrj20sv08tgpg.jpg)

账号的处理及测试可参考github大神的方法：**[检测程序介绍](https://github.com/free-ss/free-ss.site/blob/master/%E6%A3%80%E6%B5%8B%E7%A8%8B%E5%BA%8F%E4%BB%8B%E7%BB%8D.md)**

# 最后

对于这个网站学习挺好，其他就不要多想了。

## 相关资源

### 本文相关库

python=3.6.5
brotli=1.0.7
cfscrape=2.0.3
requests=2.21.0
Js2Py=0.60
Pillow=5.1.0    # Pillow(PIL Fork)
cryptodemo=0.0.1

### 第三方开源库

https://github.com/Anorov/cloudflare-scrape
https://github.com/VeNoMouS/cloudflare-scrape-js2py

https://github.com/PiotrDabkowski/Js2Py

### 在线测试工具

在线二维码-Convert Your Base64 to Image
https://codebeautify.org/base64-to-image-converter
js在线解密/美化
https://tool.lu/js/
在线json解析
https://www.sojson.com/

---

### 参考

binascii.Error: Incorrect padding, even when string length is multiple of 4
https://stackoverflow.com/questions/45879045/binascii-error-incorrect-padding-even-when-string-length-is-multiple-of-4

python字符串str和字节数组相互转化
https://www.cnblogs.com/hhh5460/p/5243305.html

Python3操作二维码图片
http://blog.niuhemoon.xyz/pages/2018/04/27/Python3-QRcode/

关于xxx网站js加密算法的研究
https://github.com/gcdd1993/analyzeSS

Python爬取某站科学上网帐号(5.15更新)
https://mtaoist.xyz/2018/04/07/python-getss/
xiaoTaoist：Auto-Shadowsocks项目
https://github.com/xiaoTaoist/Auto-Shadowsocks