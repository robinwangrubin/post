---
layout: post
title: Nginx-proxy_pass理解
categories: Nginx
---


* content
{:toc}


## 需求

用户访问：www.wangrubin.com/weixin/front/123.ttf 这个文件;

www.wangrubin.com被解析到了nginx上;

nginx需要到www.123.com/uploadServer/fonts/123.ttf 取到132.ttf这个文件返回给用户。

## 问题

www.123.com 做了防盗链配置，http-header中的host字段必须是自己的域名才会接受访问；

## nginx配置

{% highlight shell %}
location /weixin/front/ {
                proxy_pass http:///www.123.com/uploadServer/fonts/;
        }
{% endhighlight %}

## 我以为

1. 当用户浏览器访问www.wangrubin.com/weixin/front/123.ttf
2. 流量到nginx上，命中location /weixin/front/
3. nginx代替用户去访问http:///www.123.com/uploadServer/fonts/123.ttf
4. www.123.com这台主机看到的请求host是www.123.com；因为nginx完全自主的发起了新的连接


## 事实是

1. 当用户浏览器访问www.wangrubin.com/weixin/front/123.ttf
2. 流量到nginx上，命中location /weixin/front/
3. nginx代替用户去访问http:///www.123.com/uploadServer/fonts/123.ttf
4. www.123.com这台主机看到的请求host是www.wangrubin.com，不匹配本机server字段，不处理

## 解决方案

{% highlight shell %}
location /weixin/front/ {
		proxy_set_header Host www.wangrubin.com;
                proxy_pass http:///www.123.com/uploadServer/fonts/;
        }
{% endhighlight %}

> proxy_pass代理到upstream 需要添加proxy_set_header Host字段很容易理解，因为upstream中配置的IP地址，所以需要传递Host值，但是proxy_pass 代理到一个域名，我本以为是nginx自身为客户端去访问http://www.123.com/uploadServer/fonts/123.ttf，这是按照道理说Host字段应该是www.123.com，然而却不是这样的。

> 猜测：nginx在reload配置时会将域名解析成IP地址，保存到内存中，这样在高并发访问场景中，无需进行耗时的DNS解析。

## 拿数据说话

`没有手动传Host参数`

![1]({{ site.url }}/pic/nginx-proxx_pass/1490778110.png)


`手动传Host参数`

![1]({{ site.url }}/pic/nginx-proxx_pass/20170329170247.png)

---------------------

end
