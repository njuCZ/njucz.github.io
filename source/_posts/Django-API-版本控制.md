title: Django API 版本控制
date: 2015-03-09 11:15:08
tags: [API versioning, django]
---
在做后台系统时，我们一般希望API变动越少，这样对调用者比较方便有利；另一方面，由于软件变化的不可预测性，系统在发展的过程中，不可避免的需要添加新的资源、或者修改现有资源，改动升级是难以避免的。因此API的设计需要考虑版本控制策略。
通常而言，API的版本控制策略通常有3种模式：第一种是平台的API永远只有一个版本，所有的用户都必须使用最新的API，任何API的修改都会影响到平台所有的用户。第二种是平台的API版本自带版本号，用户根据自己的需求选择使用对应的API，需要使用新的API特性，用户必须自己升级。第三种是兼容性版本控制，平台只有一个版本，但是最新版本需要兼容以前版本的API行为。
要做到良好的API版本控制，需要兼容第二种和第三种模式，即提供一个统一的兼容版本入口，同时对于不能兼容历史版本的API保留历史版本，支持用户能够调用到历史版本的API。另外，对历史版本的API支持一定要有时间和用户限制，即老版API支持到一定时间就删除，新用户必须使用新版API，否则一个API有很多个版本会让平台的维护非常痛苦。

# 版本控制的方式
## 方法一：利用URL
```
HTTP GET:
https://haveibeenpwned.com/api/v2/breachedaccount/foo
```

## 方法二：利用用户自定义的request header
```
HTTP GET:  
https://haveibeenpwned.com/api/v2/breachedaccount/foo
api-version: 2
```

## 方法三：利用content type
```
HTTP GET:  
https://haveibeenpwned.com/api/v2/breachedaccount/foo
Accept: application/vnd.haveibeenpwned.v2+json  
```

## 方法四：利用content type
```
HTTP GET:  
https://haveibeenpwned.com/api/v2/breachedaccount/foo
Accept: application/vnd.haveibeenpwned+json; version=2.0 
```

## 方法五：利用URL里的parameter
```
HTTP GET:  
https://haveibeenpwned.com/api/v2/breachedaccount/foo
Accept: application/vnd.haveibeenpwned+json; version=2.0 
```

# Django API Versioning

以下会介绍使用django完成API版本化的方法，以达到以下的访问方式：
```
http://xxx.com/api/(resource)/
http://xxx.com/api/v1/(resource)/
```
以及允许让客户端通过如下方式进行访问：
```
http://xxx.com/api/(resource)/
X-Version： 1
```

首先django 项目会有一个API 的 app，它的基本结构如下所示：
![](/img/2_app_structure.png)

## project urls.py:
```
url(r'^api/', include('project.api.urls', namespace='api')),
```

## api app level urls.py:
```
url(r'', include('api.v1.urls', namespace='default')),
url(r'^v2/', include('api.v2.urls', namespace='v2')),
url(r'^v1/', include('api.v1.urls', namespace='v1')),
```

## version level urls.py
```
url(r'^hello/', 'api.v1.views.HelloWorld', name="HelloWorld"),
```

创建个中间件，读取request中的header是否含有X_VERSION，若含有的话就改变请求的URL.
```
from django.core.urlresolvers import resolve
from django.core.urlresolvers import reverse
from project.settings import API_VERSION
from django.http import Http404

class VersionSwitch(object):
    def process_request(self, request):
        r = resolve(request.path_info)
        version = request.META.get('HTTP_X_VERSION', False)
        if r.namespace.startswith('api:') and version:
            if int(version[1:]) > API_VERSION:
                raise Http404
            old_version = r.namespace.split(':')[-1]
            request.path_info = reverse('{}:{}'.format(r.namespace.replace(old_version, version), r.url_name), args=r.args, kwargs=r.kwargs)
```

测试 URL
```
curl -H "X-Version: v1" http://xxx.com/api/hello/
```

# 参考文章
[版本控制策略](http://ningandjiao.iteye.com/blog/1990004)
[Web API 版本控制的几种方式](http://blog.csdn.net/hengyunabc/article/details/20506345)
[django-rest-framework-api-versioning](http://james.lin.net.nz/2014/02/18/django-rest-framework-api-versioning/)
[API Version策略索引和公司使用情况](http://www.lexicalscope.com/blog/2012/03/12/how-are-rest-apis-versioned/)