---
title: Docker 1.12 Swarm集群实战 第四章
categories: Docker
---

<!DOCTYPE html>
<html>

  <head>
	<meta charset="utf-8">
	<meta http-equiv="X-UA-Compatible" content="IE=edge">
	<meta name="viewport" content="width=device-width, initial-scale=1">
	<title>Docker 1.12 Swarm集群实战 第四章</title>
	<meta name="description" content="">
	<link rel="canonical" href="/docker/Docker-Swarm4.html">
	<link rel="alternate" type="application/rss+xml" title="Robin's Personal Website" href="/feed.xml" />
        <link rel="stylesheet" type="text/css" href="/static/css/bootstrap.min.css">
        <link rel="stylesheet" type="text/css" href="/static/css/index.css">
	<script type="text/javascript" src="/static/js/jquery-1.11.1.min.js"></script>
	<script type="text/javascript" src="/static/js/bootstrap.min.js"></script>
        <link rel="stylesheet" type="text/css" href="/static/css/monokai_sublime.min.css">
	<script type="text/javascript" src="/static/js/highlight.min.js"></script>
	<script type="text/javascript" src="/static/js/index.js"></script>
	<script>hljs.initHighlightingOnLoad();</script>
</head>


 <!--  <body data-spy="scroll" data-target="#myAffix"> -->
  <body>

    <header>

<!-- navbar -->
  <nav class="navbar navbar-inverse">
  <div class="container">
    <!-- Brand and toggle get grouped for better mobile display -->
    <div class="navbar-header">
      <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1">
        <span class="sr-only">Toggle navigation</span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </button>
      <a class="navbar-brand" href="/">Robin's Personal Website</a>
      <p class="navbar-text"></p>
    </div>
    <!-- Collect the nav links, forms, and other content for toggling -->
    <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
      <ul class="nav navbar-nav navbar-right">

        
          <li>
        
        <a href="/">Home</a></li>

        
          
            
              <li>
            
            <a href="/project/"><span class="glyphicon "></span> Project</a></li>
          
        
          
            
              <li>
            
            <a href="/about/"><span class="glyphicon "></span> About</a></li>
          
        
          
        
          
        
      </ul>
    </div><!-- /.navbar-collapse -->
  </div><!-- /.container-fluid -->
</nav>

</header>

    <div id="main" class="container main">
      <div class="row">
  <div id="myArticle" class="col-sm-9">
    <div class="post-area post">
      <header>
        <h1>Docker 1.12 Swarm集群实战 第四章</h1>
        <p>Nov 20 2016</p>
      </header>
      <hr>
      <article>
        <ul id="markdown-toc">
  <li><a href="#section" id="markdown-toc-section">前言</a>    <ul>
      <li><a href="#section-1" id="markdown-toc-section-1">应用源码分析</a></li>
    </ul>
  </li>
  <li><a href="#rolling-updates" id="markdown-toc-rolling-updates">Rolling Updates</a>    <ul>
      <li><a href="#section-2" id="markdown-toc-section-2">更新应用代码</a></li>
      <li><a href="#build" id="markdown-toc-build">重新build镜像</a></li>
      <li><a href="#pushregistry" id="markdown-toc-pushregistry">push镜像到本地registry</a></li>
      <li><a href="#worker" id="markdown-toc-worker">滚动更新worker服务</a></li>
    </ul>
  </li>
  <li><a href="#section-3" id="markdown-toc-section-3">容器服务回滚</a></li>
</ul>

<h2 id="section">前言</h2>

<p>通过上一章节的测试，我们怀疑是程序代码的问题导致瓶颈所在</p>

<p>这一章我们会修正代码，并学习如何利用docker swarm滚动更新我们的服务副本容器</p>

<h3 id="section-1">应用源码分析</h3>

<p>下面我们来一起看看rng和hasher应用的源代码</p>

<figure class="highlight"><pre><code class="language-shell" data-lang="shell"><span class="o">[</span>root@node0 ~]# <span class="nb">cd</span> /root/orchestration-workshop/dockercoins/
<span class="o">[</span>root@node0 dockercoins]# ls
docker-compose.yml  docker-compose.yml-images  docker-compose.yml-logging  hasher  rng  webui  worker
<span class="o">[</span>root@node0 dockercoins]# <span class="nb">cd </span>rng/
<span class="o">[</span>root@node0 rng]# cat rng.py 
from flask import Flask, Response
import os
import socket
import <span class="nb">time
</span>app <span class="o">=</span> Flask<span class="o">(</span>__name__<span class="o">)</span>
<span class="c"># Enable debugging if the DEBUG environment variable is set and starts with Y</span>
app.debug <span class="o">=</span> os.environ.get<span class="o">(</span><span class="s2">"DEBUG"</span>, <span class="s2">""</span><span class="o">)</span>.lower<span class="o">()</span>.startswith<span class="o">(</span><span class="s1">'y'</span><span class="o">)</span>
hostname <span class="o">=</span> socket.gethostname<span class="o">()</span>
urandom <span class="o">=</span> os.open<span class="o">(</span><span class="s2">"/dev/urandom"</span>, os.O_RDONLY<span class="o">)</span>
@app.route<span class="o">(</span><span class="s2">"/"</span><span class="o">)</span>
def index<span class="o">()</span>:
    <span class="k">return</span> <span class="s2">"RNG running on {}</span><span class="se">\n</span><span class="s2">"</span>.format<span class="o">(</span>hostname<span class="o">)</span>
@app.route<span class="o">(</span><span class="s2">"/&lt;int:how_many_bytes&gt;"</span><span class="o">)</span>
def rng<span class="o">(</span>how_many_bytes<span class="o">)</span>:
    <span class="c"># Simulate a little bit of delay</span>
    time.sleep<span class="o">(</span>0.1<span class="o">)</span>
    <span class="k">return </span>Response<span class="o">(</span>
        os.read<span class="o">(</span>urandom, how_many_bytes<span class="o">)</span>,
        <span class="nv">content_type</span><span class="o">=</span><span class="s2">"application/octet-stream"</span><span class="o">)</span>
<span class="k">if </span>__name__ <span class="o">==</span> <span class="s2">"__main__"</span>:
    app.run<span class="o">(</span><span class="nv">host</span><span class="o">=</span><span class="s2">"0.0.0.0"</span>, <span class="nv">port</span><span class="o">=</span>80<span class="o">)</span></code></pre></figure>

<p>time.sleep(0.1)每个请求会自动延迟100ms响应.</p>

<p>hasher和worker的也一样</p>

<pre><code>[root@node0 hasher]# cat hasher.rb 
require 'digest'
require 'sinatra'
require 'socket'
set :bind, '0.0.0.0'
set :port, 80
post '/' do
    # Simulate a bit of delay
    sleep 0.1
    content_type 'text/plain'
    "#{Digest::SHA2.new().update(request.body.read)}"
end
get '/' do
    "HASHER running on #{Socket.gethostname}\n"
end
</code></pre>

<p>现在你知道100ms延迟的原因了吧. 下面我们以worker服务为例, 更新下源代码缩短下响应时间.</p>

<h2 id="rolling-updates">Rolling Updates</h2>

<p>在swarm 集群的环境下, 我们每个服务都会有多个容器副本, 如何在不停止应用的情况下滚动更新每个容器副本就十分重要了</p>

<p>docker swarm集群为我们提供了方便的命令</p>

<p>那么如果我们要发布一个新版本的worker服务需要做什么呢:</p>

<ol>
  <li>更新worker服务的源代码, 缩短time.sleep(0.1)到0.01</li>
  <li>重新build worker镜像, 使用一个新的tag版本</li>
  <li>push新镜像到我们本地的镜像仓库</li>
  <li>滚动更新worker服务, 使用新的镜像</li>
</ol>

<p>在开始之前, 如果你一直跟着文章做的, 上一章为了性能测试我们已经把worker服务副本scale成0了, 我们先恢复成10个副本.</p>

<pre><code>[root@node0 ~]# docker service scale worker=10
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
anm91nvl3sdm  debug     global    alpine                                  sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.1
</code></pre>

<h3 id="section-2">更新应用代码</h3>

<p>下面我们来更新worker服务的代码~/orchestration-workshop/dockercoins/worker/worker.py 把sleep时间从0.1改成0.01</p>

<pre><code>[root@node0 dockercoins]# cat worker/worker.py |grep time.sleep
    time.sleep(0.01)
            time.sleep(10)
</code></pre>

<h3 id="build">重新build镜像</h3>

<pre><code>[root@node0 dockercoins]# docker build -t localhost:5000/dockercoins_worker:v0.01 worker
</code></pre>

<h3 id="pushregistry">push镜像到本地registry</h3>

<pre><code>[root@node0 dockercoins]# docker push localhost:5000/dockercoins_worker:v0.01
</code></pre>

<h3 id="worker">滚动更新worker服务</h3>

<p>下面我们有了新worker服务的镜像v0.01, 我们来滚动更新我们的10个worker:</p>

<pre><code>[root@node0 ~]# docker service update worker --update-parallelism 2 --update-delay 5s --image localhost:5000/dockercoins_worker:v0.01
</code></pre>

<p>–update-parallelism：每次update的容器数量</p>

<p>–update-delay：每次更新之后的等待时间</p>

<p>–image：镜像名称</p>

<p>查看更新情况</p>

<pre><code>[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                    COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1      
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1   
3l07xdfj4rjk  redis     1/1       redis                                    
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1    
anm91nvl3sdm  debug     global    alpine                                   sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                 
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.01
</code></pre>

<p><img src="/pic/docker-swarm-4/308450367.png" alt="01" /></p>

<p>现在已经达到我们的预期值了：10个worker每秒产生40个docker币</p>

<h2 id="section-3">容器服务回滚</h2>

<p>如果我们发现, 新版本worker有问题希望回滚怎么办呢?</p>

<p>很简单, 跟上面更新的命令一样啊, 直接只用v0.1版本的镜像就可以了.</p>

<p>上面我们演示了滚动更新, 如果你希望一次性都更新或者回滚呢, 更简单了, 不加参数就行了.</p>

<p>下面我们一次性回滚所有worker服务到v0.1版本</p>

<pre><code>[root@node0 dockercoins]# docker service update worker --image localhost:5000/dockercoins_worker:v0.1
worker
[root@node0 dockercoins]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
anm91nvl3sdm  debug     global    alpine                                  sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.1  
</code></pre>

<p>可以看到worker服务都回滚到v0.1版本了</p>

<hr />
<p>end</p>

      </article>
      <hr>
        <script>
            window._bd_share_config={"common":{"bdSnsKey":{},"bdText":"","bdMini":"2","bdMiniList":false,"bdPic":"","bdStyle":"1","bdSize":"24"},"share":{}};with(document)0[(getElementsByTagName('head')[0]||body).appendChild(createElement('script')).src='http://bdimg.share.baidu.com/static/api/js/share.js?v=89860593.js?cdnversion='+~(-new Date()/36e5)];
        </script>
    </div>
	    
  </div>
  
  <div id="content" class="col-sm-3">
    <!-- <div id="myAffix" class="shadow-bottom-center hidden-xs" data-spy="affix" data-offset-top="0" data-offset-bottom="-20"> -->
    <div id="myAffix" class="shadow-bottom-center hidden-xs" >
      <div class="categories-list-header">
        Content
      </div>
      <div class="content-text"></div>
    </div>
  </div>
  
</div>

    </div>

    
    <div id="top" data-toggle="tooltip" data-placement="left" title="back to top">
      <a href="javascript:;">
        <div class="arrow"></div>
        <div class="stick"></div>
      </a>
    </div>

    <footer class="">
  <div class="container">
    <div class="row">
      <div class="col-md-12">
        <a href="mailto:robin.wangrubin@gmail.com"><span class="glyphicon glyphicon-envelope"></span> robin.wangrubin@gmail.com</a>
        <span>|</span>
        <span>Copyright&copy; 2016 WangRuBin</span>
        <span>|</span>
        <span>Powered By Jekyll · Theme By Gaohaoyang</span>
      </div>
    </div>
  </div>
</footer>

  
  </body>
</html>
