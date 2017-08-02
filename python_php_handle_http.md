## python和php请求和处理HTTP请求

> 引言：在开发系统时，可能会需要调用其它系统的接口，另一方面，本系统也可能需要提供接口给其它系统调用，比较常用的接口调用方式是通过HTTP协议，因此，了解如何向其它系统发起请求以及如何处理其它系统发送过来的请求是十分重要的。

### 1 python(GET/POST)

python中发起HTTP请求时需要用到两个库urllib和urllib2。这两个模块都可以处理URL请求，但是提供了不同的功能，而且相互之间不可以替代：urllib也可以发起请求，但是不能添加headers；urllib提供urlencode生成查询字符串，但是urllib2没有。因此，两个模块的功能有所不同，经常一起使用。

GET：

```python
param = { "name": "luofeng" }
encoded_param = urllib.urlencode(param)
req = urllib2.Request(url = '%s%s%s' % (url, '?', encoded_param))
res = urllib2.urlopen(req).read()
```

POST：

```python
param = { "name": "luofeng" }
encoded_param = urllib.urlencode(param)
req = urllib2.Request(url = url, data = encoded_param)
req = urllib2.urlopen(req).read()
```

GET和POST的区别主要是参数的传递方式，GET方法将参数放在URL中，GET方法将参数放在请求体中。

### 2 python(JSON)

上面的GET和POST只能传递简单的键值对数据，对于复杂的数据结构，可能需要将部分字段先进行处理，然后再传递到后台。因此，接口中，最常用的调用方式就是HTTP+JSON。也就是直接把所有的参数放在JSON的结构中，传递给后台。

```python
param = { "name": "luofeng", "attr": { "age": "17" } }
headers = {}
headers["Content-Type"] = "application/json"
json_param = json.dumps(param)
req = urllib2.Request(url = url, data = json_param, headers = headers)
req = urllib2.urlopen(req).read()
```

传送JSON数据与普通的GET和POST的区别就在于参数是JSON数据，当然，请求类型也是POST。至于上面的请求头中的类型可以设置也可以不设置，应该是Request中自己处理了。

### 3 PHP处理请求

当处理普通的GET和POST请求时，可以直接用超全局变量$_GET、$_POST、$_REQUEST，其中，$_GET变量是将URL中的参数转换为键值对，$_POST是将POST的参数转换为键值对，$_REQUEST则综合了$_GET和$_POST。

而当处理上面的JSON数据时，获取用户传递的参数则不能用超全局变量了，需要用特殊的方式获取。

网上有说，可以用$GLOBALS["HTTP_RAW_POST_DATA"]和$HTTP_RAW_POST_DATA获取数据，经过测试，发现获取的数据为空，我的PHP版本是5.6.22。

看还有人说用php://input，而且PHP手册上也推荐使用php://input。

```php
$req_data = file_get_content("php://input");
$json_data = json_decode($req_data, true);
```
