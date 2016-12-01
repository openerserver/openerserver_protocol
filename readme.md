# OPener_Server：

### *什么是 OPener_Server?* 
Opener_Server 是一个轻Http容器标准。 具体来说：以 Http Server 作为底层架构，以异步非阻塞模式为主要思想，通过Http POST模式构建一个可注入代码的容器，新注入的代码码依靠脚本语言内置的EVAL方法来执行。 

### *主要特点：* 
* 编程思想：异步非阻塞模式贯穿程序。Http server是异步非阻塞模式，为了保证不与这个冲突，所有的注入代码均为异步非阻塞模式的实现。本容器作为以后的最小运行单元，保证异步非阻塞模式，可以方便大规模部署。
* 注入代码：现阶段通过脚本语言的内置函数来实现。例如perl的eval{};函数 , 同样具有这个特性的语言还有 python、javascript、php....
* 原则上，每个容器应用为了不与其他容器应用程序冲突，都应该启动一个新的进程。这个新的进程就是一个空容器，然后通过注入代码来实现其他应用。
* 初始容器默认使用https协议的10008端口作为管理端口。通常情况下第一个容器进程使用该默认管理端口，作为所有其他应用进程的管理进程。通过这个管理进程，可以实现启动其他应用的进程。
* 每一个启动的容器进程初始情况下是完全相同，不同的地方只有管理端口号是不同。
* 每个新的进程都有一个新的管理端口。原则上是11008往后的端口号，具体的端口号自己设定。我们未来会出一份列表，详细列出10008-11008之间的端口号的官方定义应用。
* 在管理端口上，会包含一些基本的http api，这些http api构成了Opener_Server标准的大部分。
* 标准实现的程序内部全部都是可替换指针函数。正常情况下，在任何时候，都可以热更换每一个函数。
* 任何时候你也可以通过http实时查看程序内部的运行情况，包含内部变量的情况、错误输出等等

### *次要特点：* 
* 注入代码一般分为两种模式：一种是直接注入全局性代码，将容器打造成一个独立的应用，例如注入DNS代码，生成一个独立的DNS服务器； 
* 一种是注册http 的url，并注入代码关联这个url的处理，从而生成新的http api。
* 注入代码后，可以保留在容器的配置文件中，下次启动该管理端口的容器时，自动去配置文件中，取回代码运行。
* 注入的代码也可以不在容器配置文件中，而在远程http服务器中，根据配置文件容器可以直接去远端的http服务器取回代码。
* 每个进程的退出依靠自己的管理端口，post上相应的http请求，就可以退出。
* 启动时的代码依靠每个进程自己去读取配置文件中代码。

### *标准的具体描述：* 
```perl
1. 首先你必须有一个http server 的实现。这个http server 的实现可以提供http api以管理自己。
2. 管理自己的模式为ajax模式post json字符串到url地址的/op下。
json字符串为：{action:'',reg_startup:""}
### reg_startup为真的话，当前动作插入到启动菜单中。如果进程的autorun为真，则进程启动的时候，自动运行这些reg_startup为真的动作。
### reg_startup的动作先执行，最后容器运行的时候先执行。

{action:'code',code:''} ### 在当前进程容器中，插入代码。code内代码以utf8的编码格式，插入运行。

### host："ip地址:端口号"。如果需要匹配全部则用*代替。
### url："/aa/11/22"。如果需要匹配全部则用*代替。
{action:'reg_url',url:"",host:'*:*',type:'file',go:""}       ### 指定host上的url为单个文件的浏览，文件地址在go内
{action:'reg_url',url:"",host:'*:*',type:'file_index',go:""} ### 指定host上的url为文件目录的浏览，目录地址在go内
{action:'reg_url',url:"",host:'*:*',type:'file_down',go:""}  ### 指定host上的url为单个文件的下载，文件地址在go内
{action:'reg_url',url:"*",host:'*:*',type:'file_root',go:""} ### 指定host上的http server 根目录的设定，目录地址在go内
{action:'reg_url',url:"",host:'*:*',type:'http_get',go:""}   ### 指定host上的url为http get方式的请求，这个请求的处理的代码位于go内。常用于get一个虚拟地址，使用go处理好数据并返回。
{action:'reg_url',url:"",host:'*:*',type:'form_post',go:""}  ### 指定host上的url为form的post方式的请求，这个请求的处理的代码位于go内。
{action:'reg_url',url:"",host:'*:*',type:'ajax_post',go:""}  ### 指定host上的url为ajax的post方式的请求（也可以说是Http 的post模式），这个请求的处理的代码位于go内。
{action:'reg_url',url:"",host:'*:*',type:'html5_file_post',go:""} ### 指定host上的url为html5的文件 post上方式的请求。使用ajax post模式上传大的文件。上传成功后调用go
{action:'remote_reg_url',remote_url:"",url:"",host:'*:*',type:''} ### 从远程url地址中取回需要reg的go内容，然后执行reg_url操作

{action:'new_http_server',port:'',host:''} ###在端口 port ，ip地址 host上按照http server的模式监听。
{action:'new_https_server',port:'',host:'',cert_file:''} ###在端口 port ，ip地址 host上按照https server的模式监听，并配置一个证书：cert_file，证书文件和当前进程在同一个地址。
{action:'list_url',host:""} ###列出当前进程的该host下所有注册url地址，
{action:'del_url',url:"",host:""} ### 删掉一个host下的注册url
{action:'list_server'} ### 列出当前进程内的所有 服务列表。
{action:'stop_server',host:"",port:""} ### 停止一个 ip地址是host,端口是port的 服务。
{action:'clear_startup'} ### 清除当前进程的启动代码
{action:'remote_code',remote_url:""} ### 在当前进程容器中插入一个远程代码，代码位于：remote_url。

{action:'script',script:""} ### 启动一个新的进程，执行script内容。
{action:'remote_script',remote_url:""} ### 从remote_url中取回script内容，然后启动一个新的进程
{action:'clear_all'} ### 清除该进程内所有后添加部分，恢复到一个干净的http server 容器。
{action:'start_worker',port:"",autorun:""} ### 开启一个新的进程容器，指定这个容器的管理端口是port, autorun来决定这个新的进程容器是否随最初的管理进程容器一同启动。
{action:'stop'} ## 退出当前进程，主要用于退出当前应用程序的进程

3. 默认的管理端口上的http server均为https模式。
4. 管理的时候，需要在http header中添加一个 opener_flag 字段，字段内容用来鉴定该请求是否为认证的请求。
5. 发送管理请求后的返回结果：
{url:'/op',result:'error',action:"",reason:""} ### 操作错误返回
{url:'/op',result:'ok',action:""}    ### 操作正确返回
6. {action:'new_http_server',port:'',host:'',reg_startup:'1'}客户端发送管理请求并带reg_startup>0时，需要容器检测一下本次请求是否与之前的reg_startup请求有重复。重复则放弃本次reg_startup注册。如果reg_startup为-1，则从服务器删掉这条注册。

7. 客户端发送请求到容器时，需要满足两种形式：
. 请求正常情况必须是并发请求。
. 当需要的时候，可以阻塞，等待前一个执行的结果返回后，再继续执行。
8. 可以任意启动一个http 或者https服务器
9. 当注入的启动代码重复的时候，返回错误，并不予以注入。
```

### *关于OPener_Server标准：* 
这个标准描述了该Http容器的最终实现。这个标准可以指导以不同的语言来实现这个Opener_Server容器。
现阶段Opener_Server标准主要靠Opener_Server.pl该程序来阐述。

Opener_Server.pl该程序的位置：https://github.com/openerserver/openerserver_perl


### *项目的初衷：* 
非常简单，由于开发dns服务器和其他各种服务器类应用均需要提供http api，所以每次开发应用服务器的同时均需要编写一个http server到每个应用服务器中。http server经常会更改，导致每个应用的代码全部需要更改。因此，最终决定编写一个强大的http server，然后将其他的应用程序塞入到这个http server中。


#### *历史：* 
***最初版本的Opener_Server 实现是 Larry Wang 以Perl语言编写的。项目开始于2011年12月。经过多次的迭代开发，于2015年4月基本成型。***

* 2016-08-20 添加容器标准的描述。
* 2016-01-24 正式发布到github.
* 2014-01-01 容器基本成形。
* 2011-12-24 开始编写.



#### *版权：* 
Larry wang 以The Apache License Version 2.0发表。
邮箱：openercn@gmail.com
