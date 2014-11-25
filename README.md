# PEP 333-zh-CN 
============
> 翻译自 `Python Web Server Gateway Interface v1.0`  [PEP 333 - Python Web Server Gateway Interface v1.0](https://www.python.org/dev/peps/pep-0333/)

## 译者的话
```
Pass
```
## 内容
* [序言](#序言)  
* [摘要](#摘要)
* [基本原理及目标](#基本原理及目标)  
* [概述](#概述)  
	* [应用程序/框架](#应用程序/框架)
	* [服务器/网关](#服务器/网关)
	* [中间件:可扮演两边角色的组件](#中间件:可扮演两边角色的组件)
* [规格的详细说明](#规格的详细说明)  
	* [`environ`变量](#environ变量)
		* [输入和错误流](#输入和错误流)
	* [start_response() Callable](#start_response() Callable)
		* [处理Content-Length头信息](#处理Content-Length头信息)
	* [缓冲和流](#缓冲和流)
		* [中间件处理块边界](#中间件处理块边界)
		* [可调用的write()函数](#可调用的write()函数)
	* [Unicode问题](#Unicode问题)
	* [错误处理](#错误处理)
	* [HTTP 1.1 Expect/Continue请求头](#HTTP 1.1 ExpectContinue请求头)
	* [HTTP的其他特性](#HTTP的其他特性)
	* [线程支持](#线程支持)
* [实现/应用手册](#实现/应用手册)
	* Server Extension APIs
	* Application Configuration
	* URL Reconstruction
	* Supporting Older (<2.2) Versions of Python
	* Optional Platform-Specific File Handling
* [QA问答](#QA问答)
* Proposed/Under Discussion
* [鸣谢](#鸣谢)
* [参考文献](#参考文献)
* [版权声明](#版权声明)

###序言(Done)
注意: 关于支持Python 3.x的更新版本及包含一些社区勘误，补充，更正的相关说明，请参照PEP 3333.

###摘要(Done)
本文档描述一份在web服务器与web应用/web框架之间的标准接口，此接口的目的是使得web应用在不同web服务器之间具有可移植性。

###基本原理及目标
Python目前拥有大量的web框架，比如 Zope, Quixote, Webware, SkunkWeb, PSO, 和Twisted Web--这里我仅列举出几个[参考文献1]。这么多的选择让新手无所适从，因为总得来说，框架的选择都会限制web服务器的选择。  

对比之下，虽然java也拥有许多web框架，但是java的"servlet" API使得使用任何框架编写出来的应用程序都可以在任何支持" servlet" API的web服务器上运行。

服务器中这种针对python的API（不管服务器是用python写的(如: Medusa)，还是内嵌python(如: mod_python)，亦或是通过一种网关协议来调用Python(如:CGI, FastCGI等))的使用和普及，将人们从web框架的选择和web服务器的选择中分离开来，使得用户可以自由选择适合他们的组合，而web服务器和web框架的开发者也能够把精力集中到各自的领域。  

基于此，这份PEP建议在web服务器和web应用/web框架之间建立一种简单的通用的接口规范，即Python Web服务器网关接口(WSGI)。

但是光有这么一份规范对于改变web服务器和web应用/框架的现状是不够的，只有当那些web服务器和web框架的作者/维护者们真正地实现了WSGI，这份WSGI规范才将起到它该有的效果。  

然而，由于目前还没有任何框架或服务器实现了WSGI，那些支持WSGI的框架作者也没有什么直接的奖励，因此，我们的WSGI必须拟定地足够容易实现，这样才能降低框架作者们在实现接口这件事上的初始投资成本。
 
由此可见，服务器和框架两边接口的实现的简单性，对于WSGI的实用性来说，绝对是非常重要的，同时，这一点也是任何设计决策的首要依据。
```
pass
```
###概述(Done)
WSGI接口可以分为两端：服务器/网关和应用程序/Web框架。服务器端调用一个由应用程序端提供的可调用的对象，至于该对象是如何被调用的，这要取决于服务器/网关这一端。我们假定有一些服务器/网关会要求应用程序的部署人员编写一个简短的脚本来启动一个服务器/网关的实例，并提供给服务器/网关一个应用程序对象，而还有的一些服务器/网关则不需要这样，它们会需要一个配置文件又或者是其他机制来指定应该从哪里导入或者获得应用程序对象。
 
除了纯粹的服务器/网关和应用程序/框架，还可以创建叫做中间件的组件，中间件实现了这份规约当中的两端(服务器端和应用程序端)，我们可以这样解释中间件，对于包含它们的服务器，中间件是应用程序，而对于包含在服务器当中的应用程序来说，中间又扮演着服务器的角色。不仅如此，中间件还可以用来提供可扩展的API，内容转换，导航和其他有用的功能。
 
在整个规格说明书中，我们将使用的术语"a callable(可调用的)"意思是"一个函数，方法，类，或者拥有 __call__ 方法的一个对象实例",这取决于服务器，网关，或者应用程序根据需要而选择的合适实现技术。相反，服务器，网关，或者请求一个可调用对象(callable)的应用程序必须不依赖callable(可调用对象)具体的提供方式。记住，可调用对象(callable)只是被表用，不会自省(译者注：introspect，自省，Python的强项之一，指的是代码可以在内存中象处理对象一样查找其它的模块和函数）

###应用程序/框架(Done) 
一个应用程序对象就是一个简单的接受2个参数的可调用对象(callable object)，这里的对象并不能理解为它真的需要一个对象实例：一个函数、方法、类、或者带有 `__call__` 方法的对象实例都可以用来做应用程序对象。应用程序对象必须可以被多次调用，实质上所有的服务器/网关(除了CGI)都会产生这样的重复请求。
 
(注意：虽然我们把他叫做"应用程序"对象，但这并不意味着程序员要把WSGI当做API来调用！我们假定应用程序开发者将会仍然使用更高层的框架服务来开发它们的应用程序，WSGI只是一个提供给框架和服务器开发者使用的工具，它并没有打算直接向应用程序开发者提供支持)
 
这里我们来看两个应用程序对象的示例；其中，一个是函数，另一个是类:
```python
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!\n']


class AppClass:
    """Produce the same output, but using a class

    (Note: 'AppClass' is the "application" here, so calling it
    returns an instance of 'AppClass', which is then the iterable
    return value of the "application callable" as required by
    the spec.

    If we wanted to use *instances* of 'AppClass' as application
    objects instead, we would have to implement a '__call__'
    method, which would be invoked to execute the application,
    and we would need to create an instance for use by the
    server or gateway.
    """

    def __init__(self, environ, start_response):
        self.environ = environ
        self.start = start_response

    def __iter__(self):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        self.start(status, response_headers)
        yield "Hello world!\n"
 ```
####服务器/网关(Done)
每一次，当HTTP客户端(冲着应用程序来的)发来一个请求，服务器/网关都会调用应用程序可调用对象(callable)。为了说明方便，这里有一个CGI网关，简单的说它就是一个以一个应用程序对象为参数的函数实现，请注意，这个例子中包含有限的错误处理，因为默认情况下没有被捕获到的异常都会被输出到`sys.stderr`并被服务器记录下来。

```python
import os, sys

def run_with_cgi(application):

    environ = dict(os.environ.items())
    environ['wsgi.input']        = sys.stdin
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True

    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'

    headers_set = []
    headers_sent = []

    def write(data):
        if not headers_set:
             raise AssertionError("write() before start_response()")

        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             sys.stdout.write('Status: %s\r\n' % status)
             for header in response_headers:
                 sys.stdout.write('%s: %s\r\n' % header)
             sys.stdout.write('\r\n')

        sys.stdout.write(data)
        sys.stdout.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[0], exc_info[1], exc_info[2]
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]
        return write

    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```
### 中间件:可扮演两边角色的组件（Done)
注意到单个对象可以作为请求应用程序的服务器存在，也可以作为被服务器调用的应用程序存在。这样的“中间件”可以执行以下这些功能:
 - 在相应地重写environ变量之后，根据目标URL将请求路由到不同的应用程序对象
 - 允许多个应用程序或框架在同一个进程中并行运行
 - 通过在网络中转发请求和应答，实现负载均衡和远程处理
 - 对上下文(content)进行后加工，比如应用xsl样式表
 
中间件的存在对于"服务器/网关"和"应用程序/框架"来说是透明的，并不需要特殊的支持。希望在应用程序中加入中间件的用户只需简单得把中间件当作应用提供给服务器，并配置中间件组件以服务器的身份来调用应用程序。当然，中间件组件包裹的“应用程序”也可能是另外一个包裹应用程序的中间件组件，这样循环下去就构成了我们所说的"中间件栈"了。

最重要的别忘了，中间件必须遵循WSGI的服务器和应用程序两边提出的一些限制和要求，甚至有些时候,对中间件的要求比纯粹的服务器或应用程序还要严格，这些我们都会在这份规范文档中指出来。
 
这里有一个(好玩的)中间件组件的例子，它使用`Joe Strout`写的`piglatin.py`程序将text/plain的响应转换成pig latin(译者注:将英语词尾改成拉丁语式)(注意：一个“真实”的中间件组件很可能会使用更加鲁棒的方式来检查上下文(content)的类型和下文(content)的编码。同样，这个简单的例子还忽略了一个单词还可能跨区块分割的可能性)。

```python
from piglatin import piglatin

class LatinIter:

    """Transform iterated output to piglatin, if it's okay to do so

    Note that the "okayness" can change until the application yields
    its first non-empty string, so 'transform_ok' has to be a mutable
    truth value.
    """

    def __init__(self, result, transform_ok):
        if hasattr(result, 'close'):
            self.close = result.close
        self._next = iter(result).next
        self.transform_ok = transform_ok

    def __iter__(self):
        return self

    def next(self):
        if self.transform_ok:
            return piglatin(self._next())
        else:
            return self._next()

class Latinator:

    # by default, don't transform output
    transform = False

    def __init__(self, application):
        self.application = application

    def __call__(self, environ, start_response):

        transform_ok = []

        def start_latin(status, response_headers, exc_info=None):

            # Reset ok flag, in case this is a repeat call
            del transform_ok[:]

            for name, value in response_headers:
                if name.lower() == 'content-type' and value == 'text/plain':
                    transform_ok.append(True)
                    # Strip content-length if present, else it'll be wrong
                    response_headers = [(name, value)
                        for name, value in response_headers
                            if name.lower() != 'content-length'
                    ]
                    break

            write = start_response(status, response_headers, exc_info)

            if transform_ok:
                def write_latin(data):
                    write(piglatin(data))
                return write_latin
            else:
                return write

        return LatinIter(self.application(environ, start_latin), transform_ok)
# Run foo_app under a Latinator's control, using the example CGI gateway
from foo_app import foo_app
run_with_cgi(Latinator(foo_app))
```
### 规格的详细说明 (Done)
应用程序对象必须接受两个位置参数，为了方便说明，我们不妨将它们分别命名为`environ`和`start_response`，但是这并不意味着它们必须取这两个名字。服务器或网关必须用这两个位置参数(注意不是关键字参数)来调用应用程序对象(比如，像上面展示的,调用`result = application(environ,start_response)`)

`environ`参数是一个字典对象，也是一个有着CGI风格的环境变量。这个对象必须是一个python内建的字典对象(不能是子类、用户字典(UserDict)或其他对字典对象的模仿)，应用程序必须允许以任何它需要的方式来修改这个字典， `environ`还必须包含一些特定的WSGI需要的变量(在后面章节里会提到)，有可以包含一些服务器特定的扩展变量，通过下面提到的命名规范来命名。

`start_response`参数是一个可调用者(callable)，它接受两个必须的位置参数和一个可选参数。为方便说明，我们分别将它们命名为`status`, `response_headers`,和 `exc_info` 。在强调一下，这并不不是说它们必须取这些名字。应用程序必须用这些位置参数来请求可调用者(callable) start_response(比如像这样：start_response(status,response_headers))。

`status`参数是一个形式如"999 Message here"这样的状态字符串。而`response_headers`参数是包含有(header_name,header_value)的元组,用来描述HTTP响应头。可选的`exc_info`参数会在接下来的`The start_response() Callable`和 `错误处理` 两节中描述，它只有在应用程序捕获到了错误并试图在浏览器上显示错误的时候才有用。

`start_response` 可调用者(callable)必须返回一个 write(body_data) 可调用者(callable)，write(body_data)接受一个位置参数：一个将要被做为HTTP响应体的一部分输出的字符串(注意：提供可调用者 write() 只是为了支持某些现有框架的命令式输出APIs；新的应用程序或框架应当尽量避免使用，详细情况请看 Buffering and Streaming 一节。)

当应用程序被服务器调用的时候，它必须返回能生成0个或多个字符串的iterable。这可以通过几种方式来实现，比如通过返回一个包含一系列字符串的列表，又或者是通过让应用程序本身是一个能生成多个字符串的生成器(generator)，又或者是通过让应用程序本身是一个类并且它的实例是一个可迭代的(iterable)。总之，不论通过什么途径完成，应用程序对象必须总是返回一个能生成0个或多个字符串的可迭代的(iterable)。

服务器或者网关必须将产生的字符串以一种无缓冲的方式传输到客户端，总是在传完一个字符串之后再去请求下一个字符串。(换句话说，应用程序必须自己负责实现缓冲。更多关于应用程序输出应该如何处理的细节请阅读下面的 `Buffering and Streaming`章节。)

服务器或网关应该将产生的字符串当做二进制字节序列来对待：特别地，它必须确保行的结尾没有被修改。应用程序必须负责确保将要传至HTTP客户端的字符串是以与客户端匹配的编码输出(服务器/网关可能会附加HTTP传输编码，或者为了实现一些类似字节级传输(byte-range transmission)这样的HTTP的特性而进行一些转换，更多HTTP特性的细节请看下面的 Other HTTP Features )

服务器如果成功调用`len(iterable)`，它将认为此结果是正确的并且信赖这个结果。也就是说，如果应用程序返回的可迭代的(iterable)字符串提供了一个能用的`__len__()` 方法，那么服务器就肯定应用程序确实是返回了正确的结果(关于这个方法正常情况下如何被使用的请阅读 Handling the Content-Length Header )

如果应用程序返回的可迭代者(iterable)有一个叫做close()的方法，则不论当前的请求是正常结束还是由于错误而终止，服务器/网关都**必须**在结束该请求之前调用这个方法。（这么做是为了支持应用程序对资源的释放，这份规约将试图补充PEP 325中生成器的支持，以及其他有cloase()方法的通用的可迭代者(iterable))

(注意：应用程序必须在可迭代者(iterable)产生第一个主体(body)字符串之前请求`start_response()`可调用者(callable)，这样服务器才能在发送任何主体(body)内容之前发送响应头。然而这一步的调用也可能在可迭代者(iterable)第一次迭代的时候执行,所以服务器不能假定在它们开始迭代之前 start_response() 已经被调用过了)

最后要说的是，服务器和网关不能使用应用程序返回的可迭代者(iterable)的其他任何属性，除非是针对服务器或网关的特定类型的实例，比如wsgi.file_wrapper返回的“file wrapper”（阅读 Optional Platform-Specific File Handling )。通常情况下，只有在这里指定的属性，或者通过PEP 234 iteration APIs访问的属性才是可以接受的。

#### environ变量 (Done)

`environ`字典被用来包含这些CGI环境变量，这些变量定义可以在Common Gateway Interface specification [2]中找到。下面所列出的这些变量必须给定，除非它们的值是空字符串,在这种情况下如果下面没有特别指出的话他们会被忽略。

------
__REQUEST_METHOD__  
HTTP的请求方式, 比如 "GET" 或者 "POST"。这个参数永远不可能是空字符串，故必须给出。

------
__SCRIPT_NAME__  
URL请求中'路径'('path')的开始部分，对应了应用程序对象，这样应用程序就知道它的虚拟位置。如果该应用程序对应服务器的 根目录的话， 那么`SCRIPT_NAME`的值可能为空字符串。

------
__PATH_INFO__  
URL请求中'路径'('path')的其余部分，指定请求的目标在应用程序内部的虚拟位置。如果请求的目标是应用程序根目录并且末尾没有'/'符号结尾的话，那么`PATH_INFO`可能为空字符串 。

------
__QUERY_STRING__  
URL请求中紧跟在"?"后面的那部分，它可以为空或者不存在。

------
__CONTENT_TYPE__  
HTTP请求中`Content-Type`字段包含的所有内容，它可以为空或者不存在。

------
__CONTENT_LENGTH__  
HTTP请求中`Content-Length`字段包含的所有内容，它可以为空或者不存在。

------
__SERVER_NAME, SERVER_PORT__  
这两个变量可以和 SCRIPT_NAME、PATH_INFO 一起组成一个完整的URL。然而要注意的是，如果有出现HTTP_HOST，那么在重建URL请求的时候应当优先使用 HTTP_HOST而非 SERVER_NAME 。详细内容请阅读下面的 `URL Reconstruction`这一章节 。SERVER_NAME 和 SERVER_PORT这两个变量永远不可能是空字符串，并且总是必须给定的。

------
__SERVER_PROTOCOL__  
客户端发送请求的时候所使用的协议版本。通常是类似"HTTP/1.0" 或 "HTTP/1.1"这样的字符串，可以被应用程序用来判断如何处理请求HTTP请求报头。（事实上我认为这个变量更应该被叫做 REQUEST_PROTOCOL，因为这个变量代表的是在请求中使用的协议，而且看样子和服务器响应时使用的协议毫无关系。 。然而，为了保持和CGI的兼容性，这里我们还是沿用已有的名字SERVER_PROTOCOL)

------
__HTTP_ 变量组__  
这组变量对应着客户端提供的HTTP请求报头(即那些名字以 "HTTP_" 开头的变量)。这组变量的存在与否应该和HTTP请求中相对应的HTTP报头保持一致。

------
一个服务器或网关应该尽可能多地提供其他可用的CGI变量。另外，如果启用了SSL，服务器或网关应该也尽可能地提供可用的Apache SSL环境变量 [5] ，比如 HTTPS=on 和SSL_PROTOCOL。不过要注意，任何使用了上面没有列出的变量的应用程序对不支持相关扩展的服务器来说就必然有点不可移植的缺点了。(比如，不发布文件的web服务器就没法提供一个有意义的 ``DOCUMENT_ROOT 或 PATH_TRANSLATED变量。)
 
一个支持WSGI的服务器或网关应该在文档中描述它们自己的定义的同时，适当地说明下它们可以提供些什么变量。而应用程序这边应该对所有它们要求的每一个变量的存在性进行检查，并且在检查到某些变量不存在时有备用的措施。
 
注意: 缺少变量 (比如在不需要验证的情况下的 REMOTE_USER ) 应该被排除在`environ`字典之外。同样需要注意，CGI定义的变量如果存在的话必须是字符串。任何除字符串类型以外的CGI变量的存在都是违反本规范的。

除了CGI定义的变量，`environ` 字典也可以包含任意操作系统的环境变量，并且必须包含下面这些WSGI定义的变量:

| 变量        | 值    |
| --------   | -----| 
| wsgi.version	| 元组tuple (1, 0)，代表WSGI版本 1.0 |   
|wsgi.url_scheme| 应用程序被调用过程中的一个字符串，表示URL中的"scheme"部分。正常情况下，它的值是"http"或者"https"，视场合而定。| 
|wsgi.input	|一个能被HTTP请求主体(body)读取的输入流(类文件对象)(服务器或者网关在读取的时候可能是按需读取，因为应用程序不定时发来请求。或者它们会预读取客户端的请求体然后缓存在内存或者磁盘中，又或者根据它自己的参数，利用其他任意的技术来提供这样一种输入流)|  
|wsgi.errors|输出流(类文件对象)，用来写错误信息的，目的是记录程序或者其他在标准化及可能的中心化错误。它应该是一个“文本模式”的流；举一个例子,应用程序应该用"\n"作为行结束符，并且默认服务器/网关能将它转换成正确的行结束符。对许多服务器来说，`wsgi.errors`是服务器的主要错误日志。当然也有其它选择，比如`sys.stderr`，或者干脆是某种日志文件。服务器的文档应当包含这类解释：比如该如何配置这些日志，又或者该从哪里去查找这些记录下来的输出。如果需要，一个服务器或网关还可以向不同的应用程序提供不同的错误流。 |
|wsgi.multithread	|如果应用程序对象同时被在同一个进程中的不同线程调用，则这个参数值应该为"true"，否则就为“false"|
|wsgi.multiprocess	|如果相同的应用程序对象同时被另外一个线程调用，则此参数值应该为"true"；否则就为"false"。|
|wsgi.run_once	|如果服务器/网关期待(但不保证)应用程序在它所在的进程生命期间只会被调用一次，则这个值应该为"true"。正常情况下，对于那些基于CGI(或类似的)的网关，这个值只会是"true"。|  

最后想说的是，这个`environ`字典有可能会包含服务器定义的变量。这些变量应该用小写，数字，点号及下划线来命名，并且必须定义一个该服务器/网关特有的前缀开头。举个例子，`mod_python`在定义变量的时候，就会使用类似`mod_python.some_variable`这样的名字。

#####输入和错误流 (Done)
服务器提供的输入输出流必须提供以下的方法

|方法(Method)|流(Stream)|注释(Notes)|  
| --------| -----| ------|
|read(size)|input|1|
|readline()|input|1, 2|
|readlines(hint)|input|1, 3|
|\__iter__()|input| |
|flush()|errors|4|
|write(str)|errors|	
|writelines(seq)|errors|
	
以上所有方法的语义在Python Library Reference里已经写得很具体了，除了在注释栏特别标注的注意点之外。

1. 服务器读取的长度不一定非要超过客户端指定的`Content-length`, 并且如果应用程序尝试去读取超过那个点，则服务器可以模拟一个流结束（end-of-file）的条件。而应用程序这边则不应该去尝试读取比指定的CONTENT_LENGTH长度更多的数据。
2. 可选参数size是不支持用于readline()方法中的，因为它有可能给开发服务器的作者们增加复杂度，所以在实际中它不常用。
3. 请注意readlines()方法中的隐藏参数对于调用者和实现者都是可选。应用程序方可以自由选择不提供它，而服务器或网关这端也可以自由选择是否忽略它。
4. 由于错误流不能回转(rewound)，服务器和网关可以立即选择自由地继续向前写操作(forward write)，而不需要缓存。在这种情况下，flush()方法可能就是个空操作(no-op)。不过，可移植的应用程序不能假定这个输出是无缓冲的或者flush是空操作。可移植的应用程序如果需要确保输出确实已经被写入，则必须调用flush()方法。(例如：从多进程中写入同一个日志文件的时候，可以做到最小化的数据交织）
 
每一个遵循此规范的服务器都必须支持上表所列出的每一个方法。每一个遵循此规范的应用程序都不能使用除上表之外的其他方法或属性。特别需要指出的是，应用程序千万不要试图去关闭这些流，就算它们自己有对close()方法做处理。

####The start_response() Callable
```
pass
```

####处理Content-Length头信息 (Done)
如果应用程序没有提供Content-Length头，服务器/网关可以有好几种方式来处理它，其中最简单的就是在response完成的时候关闭客户端连接。

然而在某些情况下，服务器或网关可能会要么自己生成Content-Length头，要么至少避免了关闭客户端连接这件事。如果应用程序没有调用write()callable，并返回一个len()是1的iterable，那么服务器便可以自动地识别Content-Length长度，这是通过iterable产生出来的第一个的字符串的长度来判断的。

还有，如果服务器和客户端都支持HTTP/1.1 "分块传输编码(chunked encoding)"[3],那么服务器可以在每一次调用write()方法发送数据块(Chunk)或iterable迭代生成的字符串的时候，因此会为每个chunk数据块生成Content-Length头。这样可以让服务器保持客户端长连接，如果需要的话。注意如果真要这么做的话，服务器必须完全遵循RFC2616规范，要不然就回退到另外的策略来处理Content-Length的缺失。
 
(注意：应用程序和中间件的输出一定不能使用任何类型的Transfer-Encoding，比如chunking or gzipping等；因为在逐跳路由("hop-by-hop")操作中，这些encoding都是实际的服务器/网关的职权。详细信息参见下面的Other HTTP Features章节）

#####中间件处理块边界 (Done)
为了更好地支持异步应用程序和服务器，中间件组件一定不能阻塞迭代，该迭代等待从应用程序的iterable中返回多个值。如果中间件需要从应用程序中累积更多的数据来生成一个输出流，那么它必须生成(yield)一个空字符串。

让我们换一种方式来表述这个需求，每一次底层应用程序生成(yields)一个值，中间件组件都必须至少生成(yields)一个值。如果中间件不能生成(yields)任何值，那么它也必须生成(yields)一个空字符串。

这个要求确保了异步的服务器和应用程序能共同协作，在需要同时提供多个应用程序实例的时候减少线程的数量。

注意,这样的要求同时也意味着一旦处于底层应用的程序返回了一个iterable，中间件就必须尽快的返回一个iterable。另外，中间件调用write() callable来传输由底层应用程序yielded的数据是不允许的。中间件仅可以使用它父服务器的write() callable来传输由底层应用程序利用中间
件提供的write() callable发送来的数据。

#####可调用的write()函数 
```
pass
```

####Unicode Issues
```
pass
```

####错误处理  
```
pass
``` 

####Http1.1的长连接
```
pass
```

####其他的HTTP特性
```
pass
```
 
####线程支持(done)
```
pass
```

###实现/应用 事项
```
pass
```
####服务扩展API
```
pass
```

####应用程序配置 (done)
```
pass
```

#### URL的构建
```
pass
```
###QA问答
1.为什么evniron必须是字典？用子类(subclass)不行吗？ (Done)
用字典的原理是为了最大化地满足在服务器间的移植性。另一种选择就是定义一些字典方法的子集，并以字典的方法作为标准的便捷的接口。然而事实上，大多服务器可能只需要找到一个合适的字典就足够它们用了，并且框架的作者往往期待完整的字典特性可用，因为多半情况是这样的。但是如果有一些服务器选择不用字典，那么尽管这类服务器也“符合”规范，但会有互用性的问题出现。因此使用强制的字典简化了规范和并确保了互用性。。
 
注意，以上这些并不妨碍服务器或框架的开发者向`evnrion`字典里加入自定义的变量来提供特别的服务。我们推荐使用这种方式提供任何一种增值服务。
 
2.为什么你既能调用write()又能yield字符串/返回一个迭代器(iterable)？我们难道不应该只选择一种做法吗？ (Done)
如果我们仅仅使用迭代的做法，那么现存的框架将遭受"push"可用性的折磨。但是，如果我们只支持通过write()推送，那么服务器在传输大文件的时候性能将恶化（如果一个工作线程(worker)没有将所有的output都发送完成，那么将无法进行下一个新的request请求）。因此，我们做这样的妥协，好处是允许应用程序支持两个方法，视情况而定，并且比起需要push-only的方式来说，只给那些服务器的实现者们多了一点负担而已。
 
3.close()方法是拿来做什么的？(Done)
在应用程序执行期间，当writes完成后，应用程序可以通过一个try/finally代码块来确保资源都被释放了。但是，如果应用程序返回一个迭代(iterable)，那么在迭代器被垃圾收集器收集之前任何资源都不会被释放。这里的close()惯用法允许应用程序在一个request请求完成阶段释放重要资源，并且它向前兼容PEP 325中提到的迭代器中的try/finally.
 
4.为什么这个接口要设计地这么初级？我希望添加更多酷炫的W功能!（例如 cookies, 会话(sessions), 持久性(persistence),...） (Done)
记住，这并不是另一个Python的web框架，这仅仅是一个框架向web服务器通信的方法，反之亦然。如果你想拥有上面你说的这些特性，你需要选择一个提供这些你想要的特性的框架。并且如果这个框架让你创建一个WSGI应用程序，你将可以让它在泡在大部份支持WSGI的服务器上面。同样，一些WSGI服务器或许会通过在他们的`environ`字典里的提供的对象来提供一些额外的服务；可以参阅这些服务器具体的文档了解详情。（当然，使用这样扩展的应用程序将无法移植到其他的WSGI-based服务器上）

 
5.为什么使用CGI的变量而不是旧的HTTP头呢？并且为什么将它们和WSGI定义的变量混在一起呢？（Done)
许多现有的框架很大程序上是建立在CGI规范的基础上的，并且现有的web服务器知道如何生成CGI变量。相比之下，另一种表示到达的HTTP信息的方式不仅分散破碎更缺乏市场支持。因此使用CGI“标准”看起来是个不错的办法来最大化发挥现有的实现。至于将它们同WSGI变量混合在一起，那是因为分离他们的话会导致需要传入两个字典参数，显然这样做没什么好处。
 
6.那关于状态字符串，我们可不可以仅仅使用数字来代替，比如说传入200而不是"200 OK"？(Done)
这样做会使服务器/网关被复杂化，因为那样的话服务器/网关就需要一个数值状态和相应信息的对照表。相比之下，让应用程序或框架的作者们在他们处理专门的响应代码的时候顺便输入一些额外的信息则显得要简单地多，并且经常是现有的框架已经有一个这样的表包含这些需要的信息了。总之，权衡之后，我们认为这个让应用程序/框架来负责要比服务器或网关来负责要合适。
 
7.为什么wsgi.run_once不能保证app只运行一次？
```
pass
```
 
8.Feature x(dictionaries, callables, etc.)对于应用程序代码来说是很丑陋的，我们可以使用对象来代替吗？
 
```
pass
```

###还在讨论的提议
```
pass
```

###鸣谢 (Done)

感谢那些Web-SIG邮件组里面的人，没有他们周全的反馈，将不可能有我这篇修正草案。特别地，我要感谢：  
- mod_python的作者Gregory "Grisha" Trubetskoy，是他毫不留情地指出了我的第一版草案并没有提供任何比“普通旧版的CGI”有优势的地方，他的批评促进了我去寻找更好的方法。  
- Ian Bicking，是他总是唠叨着要我适当地提供多线程(multithreading)及多进程(multiprocess)相关选项，对了，他还不断纠缠我让我提供一种机制可以让服务器向应用程序提供自定义的扩展数据。  
- Tony Lownds，是他提出了`start_response`函数的概念，提供给它status和headers两个参数然后返回一个write函数。他的这个想法为我后来设计异常处理功能提供了灵感，尤其是在考虑到中间件复写(overrides)应用程序的错误信息这方面。  
- Alan Kennedy, 一个有勇气去尝试实现WSGI-on-Jython(在我的这份规约定稿之前)的人，他帮助我形成了“supporting older versions of Python”这一章节，以及可选的`wsgi.file_wrapper`套件。  
- Mark Nottingham，是他为这份规约的HTTP RFC 发行规范做了大量的后期检查工作，特别针对HTTP/1.1特性，没有他的指出，我甚至不知道有这东西存在。  

###参考文献 (Done)
[1]	The Python Wiki "Web Programming" topic ( http://www.python.org/cgi-bin/moinmoin/WebProgramming )  
[2]	The Common Gateway Interface Specification, v 1.1, 3rd Draft ( http://ken.coar.org/cgi/draft-coar-cgi-v11-03.txt )  
[3]	"Chunked Transfer Coding" -- HTTP/1.1, section 3.6.1 ( http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6.1 )  
[4]	"End-to-end and Hop-by-hop Headers" -- HTTP/1.1, Section 13.5.1 ( http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13.5.1 )  
[5]	mod_ssl Reference, "Environment Variables" ( http://www.modssl.org/docs/2.8/ssl_reference.html#ToC25 )  
###版权声明(Done)  
这篇文档被托管在Mercurial上面.  
原文链接: https://hg.python.org/peps/file/tip/pep-0333.txt  
