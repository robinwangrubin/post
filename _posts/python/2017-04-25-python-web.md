---
layout: post
title: Python抓取特定XML JSON HTML文件内容
categories: Python
---

* content
{:toc}


## XML文件处理

{% highlight shell %}
#!/usr/bin/python
# -*- coding: UTF-8 -*-

from xml.parsers.expat import ParserCreate
from urllib import request

# 定义回调机制
class MovieHandler(object):
   def __init__(self):
      self.CurrentData = ""
      self.type = ""
      self.format = ""
      self.year = ""
      self.rating = ""
      self.stars = ""
      self.description = ""

   # 元素开始事件处理
   def startElement(self, tag, attributes):
      self.CurrentData = tag
      if self.CurrentData == "movie":
         print("*****Movie*****")
         title = attributes["title"]
         print("Title:", title)

   # 元素结束事件处理
   def endElement(self, tag):
      if self.CurrentData == "type":
         print("Type:", self.type)
      elif self.CurrentData == "format":
         print("Format:", self.format)
      elif self.CurrentData == "year":
         print("Year:", self.year)
      elif self.CurrentData == "rating":
         print("Rating:", self.rating)
      elif self.CurrentData == "stars":
         print("Stars:", self.stars)
      elif self.CurrentData == "description":
         print("Description:", self.description)
      self.CurrentData = ""

   # 内容事件处理
   def characters(self, content):
      if self.CurrentData == "type":
         self.type = content
      elif self.CurrentData == "format":
         self.format = content
      elif self.CurrentData == "year":
         self.year = content
      elif self.CurrentData == "rating":
         self.rating = content
      elif self.CurrentData == "stars":
         self.stars = content
      elif self.CurrentData == "description":
         self.description = content

# 获取XML文件
with request.urlopen("http://wangrubin.com/testfile/movies.xml") as f:
        xml = f.read()

handler = MovieHandler()
parser = ParserCreate()

# 重写原有的StartElementHandler EndElementHandler CharacterDataHandler
parser.StartElementHandler = handler.startElement
parser.EndElementHandler = handler.endElement
parser.CharacterDataHandler = handler.characters

parser.Parse(xml)
{% endhighlight %}
