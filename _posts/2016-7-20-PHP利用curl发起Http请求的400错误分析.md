---
layout: post
title: PHP利用curl发起Http请求的400错误分析
category: php
tags: PHP,CURL,HTTP,400
date: 2014-05-01
---


# PHP利用curl发起Http请求的400错误分析

### 背景

#### 现象一-php封装的http请求接口调用url   

![php的curl_exec接口调用结果.png](http://o9evx54l5.bkt.clouddn.com/php_curl_exec_results.png)


**1.response的rawheader值显示400错误**   

```
HTTP/1.1 400 Unknown Version
Server: Tengine/2.0.3
Date: Tue, 19 Jul 2016 07:38:15 GMT
Content-Length: 0
Connection: keep-alive

```       

**2.自己封装的发送http请求接口**  

```        
/**
* 发送httpful get请求
* @param  [array] $params [description]
* @return [object]         [description]
*/
public  function sendGetRequest($serverUrl,$params){
    try{
        $str_params = '';
        foreach ($params as $key => $value) {
            $str_params .= "&$key=".$value;
        }
        
        $response =  \Httpful\Request::get($serverUrl.$str_params)->send();
        return  $response->body;
    }catch(Execption $e){
        return array("statuscode"=>'-1',"message"=>'服务器出错了');
    }
    
}      
```    

**3.Request.php封装的Request对象的send方法**   

> nategood/httpful中的Request::send方法使用php的*curl_exec*接口发起http请求。

```                
/**
* Actually send off the request, and parse the response
* @return string|associative array of parsed results
* @throws ConnectionErrorException when unable to parse or communicate w server
*/
public function send()
{
    if (!$this->hasBeenInitialized())
        $this->_curlPrep();

    $result = curl_exec($this->_ch);

    if ($result === false) {
        if ($curlErrorNumber = curl_errno($this->_ch)) {
            $curlErrorString = curl_error($this->_ch);
            $this->_error($curlErrorString);
            throw new ConnectionErrorException('Unable to connect: ' . $curlErrorNumber . ' ' . $curlErrorString);
        }

        $this->_error('Unable to connect.');
        throw new ConnectionErrorException('Unable to connect.');
    }

    $info = curl_getinfo($this->_ch);

    // Remove the "HTTP/1.x 200 Connection established" string and any other headers added by proxy
    $proxy_regex = "/HTTP\/1\.[01] 200 Connection established.*?\r\n\r\n/s";
    if ($this->hasProxy() && preg_match($proxy_regex, $result)) {
        $result = preg_replace($proxy_regex, '', $result);
    }

    $response = explode("\r\n\r\n", $result, 2 + $info['redirect_count']);

    $body = array_pop($response);
    $headers = array_pop($response);

    curl_close($this->_ch);

    return new Response($body, $headers, $this, $info);
}
```             

**4.curl_exec调用前url值**      
  
```     
http://192.168.59.146/api?version=1.0&format=json&appkey=KtSNKxk3&access_token=changyanyun&method=pan.file.export&uid=3062000039000412278&fileId=3aaaa5c8-3eaa-4511-91e7-46831d418f10&fileIndex={"lifecycle":{"auditstatus":"0"},"general":{"source":"UGC","creator":"\u6559\u5e080523","uploader":"gsres_iflytek_f968bca78360d38abcbaf23a5a318b12","extension":"ppt","title":"Unit12 What did you do last weekdend\u8bfe\u65f64\uff081\uff09"},"properties":{"subject":["01"],"edition":["01"],"stage":["010001"],"book":["01010101-001"],"unit":["01"],"course":[""],"unit1":["01"],"unit2":[""],"unit3":[""],"phase":["03"],"type":["0100"],"rrtlevel1":["08"]}}
```      

#### 现象二-浏览器中直接访问url

![浏览器直接访问url接口调用结果.png](http://o9evx54l5.bkt.clouddn.com/browser_results.png)


```
http://192.168.59.146/api?version=1.0&format=json&appkey=KtSNKxk3&access_token=changyanyun&method=pan.file.export&uid=3062000039000412278&fileId=3aaaa5c8-3eaa-4511-91e7-46831d418f10&fileIndex={ %22lifecycle%22:{ %22auditstatus%22:%220%22},%22general%22:{ %22source%22:%22UGC%22,%22creator%22:%22\u6559\u5e080523%22,%22uploader%22:%22gsres_iflytek_f968bca78360d38abcbaf23a5a318b12%22,%22extension%22:%22ppt%22,%22title%22:%22Unit12%20What%20did%20you%20do%20last%20weekdend\u8bfe\u65f64\uff081\uff09%22},%22properties%22:{ %22subject%22:[%2201%22],%22edition%22:[%2201%22],%22stage%22:[%22010001%22],%22book%22:[%2201010101-001%22],%22unit%22:[%2202%22],%22course%22:[%22%22],%22unit1%22:[%2202%22],%22unit2%22:[%22%22],%22unit3%22:[%22%22],%22phase%22:[%2203%22],%22type%22:[%220100%22],%22rrtlevel1%22:[%2208%22]}}
```

### 原因分析

对比现象一和现象二中的url值，可以发现：

1. fileIndex参数值为json字符串  
2. fileIndex参数值中的title属性值中的空格字符在现象二中被编码为%20,且双引号"被编码为%22，此操作为浏览器自动执行
3. 将url中的空格进行编码后，curl_exec接口返回200

#### 测试代码

```
<?php
// 创建一个新cURL资源
$ch = curl_init();

$url = 'http://192.168.59.146/api?version=1.0&format=json&appkey=KtSNKxk3&access_token=changyanyun&method=pan.file.export&uid=3062000039000412278&fileId=3aaaa5c8-3eaa-4511-91e7-46831d418f10&fileIndex={"lifecycle":{"auditstatus":"0"},"general":{"source":"UGC","creator":"\u6559\u5e080523","uploader":"gsres_iflytek_f968bca78360d38abcbaf23a5a318b12","extension":"ppt","title":"Unit12 What did you do last weekdend\u8bfe\u65f64\uff081\uff09"},"properties":{"subject":["01"],"edition":["01"],"stage":["010001"],"book":["01010101-001"],"unit":["01"],"course":[""],"unit1":["01"],"unit2":[""],"unit3":[""],"phase":["03"],"type":["0100"],"rrtlevel1":["08"]}}';

//对url中的空格进行转义
if(isset($_GET["urlencode"])){
	$url = str_replace(" ", '%20', $url);
}

var_dump("url");
var_dump($url);
// 设置URL和相应的选项
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_HEADER, 0);

// 抓取URL并把它传递给浏览器
$result = curl_exec($ch);
var_dump("result");
var_dump($result);
if ($result === false) {
	if ($curlErrorNumber = curl_errno($this->_ch)) {
		$curlErrorString = curl_error($this->_ch);
		$this->_error($curlErrorString);
		throw new ConnectionErrorException('Unable to connect: ' . $curlErrorNumber . ' ' . $curlErrorString);
	}

	$this->_error('Unable to connect.');
	throw new ConnectionErrorException('Unable to connect.');
}
var_dump("ch");
var_dump($ch);

$info = curl_getinfo($ch);
var_dump("info");
var_dump($info);
// Remove the "HTTP/1.x 200 Connection established" string and any other headers added by proxy
$proxy_regex = "/HTTP\/1\.[01] 200 Connection established.*?\r\n\r\n/s";
if (preg_match($proxy_regex, $result)) {
	$result = preg_replace($proxy_regex, '', $result);
}

$response = explode("\r\n\r\n", $result, 2 + $info['redirect_count']);
var_dump("response");
var_dump($response);

$body = array_pop($response);
var_dump("body");
var_dump($body);
$headers = array_pop($response);
var_dump("headers");
var_dump($headers);


// 关闭cURL资源，并且释放系统资源
curl_close($ch);
?>
```

#### 百分号编码(即url编码)   

一般来说，URL只能使用英文字母、阿拉伯数字和某些标点符号，不能使用其他文字和符号。阮一峰在他的[博文][6]中提到，网络标准[RFC 1738][5]做了硬性规定:

> "...Only alphanumerics [0-9a-zA-Z], the special characters "$-_.+!*'()," [not including the quotes - ed], and reserved characters used for their reserved purposes may be used unencoded within a URL."
<br>"只有字母和数字[0-9a-zA-Z]、一些特殊符号"$-_.+!*'(),"[不包括双引号]、以及某些保留字，才可以不经过编码直接用于URL。"

[url编码][4]是特定上下文的统一资源定位符 (URL)的编码机制,服务器解析http请求时如果遇到非标准字符，会对该非标准字符进行编码


### 总结

当http请求出现400错误时，一般是**http请求无效**(**bad request**)，url参数编码格式不正确导致。而url是有一定格式要求的，一般只能使用英文字母、阿拉伯数字和一些特殊字符，其他字符如空格和双引号需要经过编码后才能用户url。      

* 在浏览器中直接访问时浏览器会自动进行编码处理
* 但是在程序中直接调用时则需要事先对url进行编码，如php使用curl_exec调用时**需要对空格等非法字符进行编码**。
* 特别是针对url参数中包含[json数据][3]值需要多加注意。

[1]: http://www.checkupdown.com/status/E400_zh.html   
[2]: http://stackoverflow.com/questions/9349703/server-returned-http-response-code-400    
[3]: http://stackoverflow.com/questions/26996679/http-error-400-bad-request-when-posting-json-data-over-httpurlconnection     
[4]: https://zh.wikipedia.org/wiki/%E7%99%BE%E5%88%86%E5%8F%B7%E7%BC%96%E7%A0%81     
[5]: http://www.ietf.org/rfc/rfc1738.txt     
[6]: http://www.ruanyifeng.com/blog/2010/02/url_encoding.html
