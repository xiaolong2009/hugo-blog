---
date: 2016-10-15T11:46:53+08:00
title: tornado source code
tags: ["tornado"]
---

tornado 即是一个高性能的web server，又是一个web framwork，而且web server 采用的是asynchronous IO的网络模型，是一种很高效的模型，那tornado是怎么完成这些工作的，带着疑问我看了一下tornado的2.0版本的源码

在看tornado的源码之前还有些准备工作要做  
1. IO多路复用-epoll   
https://fukun.org/archives/10051470.html  
https://www.zhihu.com/question/32163005

2. Reactor模型  
http://blog.csdn.net/u013074465/article/details/46276967

tornado核心的几个部分如下
- HTTPServer
处理套接字的 listen/bind/accept。

- IOStream
处理套接字的 read/write。

- HTTPConnection
处理与 HTTP client 建立的连接，解析 HTTP Request 的 header 和 body。

- IOLoop
I/O loop，循环取出可用的 fd，并调用对应的事件处理函数。

- RequestHandler
处理请求，支持 GET/POST 等操作。
# ioloop.py
ioloop.py是典型的Reactor模型的实现，主要要看一下它的start方法里的核心调度过程。IOLoop 实例对象调用start后开始 epoll事件循环机制，该方法会一直运行直到 IOLoop 对象调用 stop 函数、当前所有事件循环完成。  

start方法中主要分三个部分：
- 运行注册了的回调函数
- 检查超时事件；
- epoll 返回I/O事件的处理

```python
def start(self):
        """Starts the I/O loop.

        The loop will run until one of the I/O handlers calls stop(), which
        will make the loop stop after the current event iteration completes.
        """
        if self._stopped:
            self._stopped = False
            return
        self._running = True
        while True:
            # Never use an infinite timeout here - it can stall epoll
            poll_timeout = 0.2

            # Prevent IO event starvation by delaying new callbacks
            # to the next iteration of the event loop.
            callbacks = self._callbacks
            self._callbacks = []
            # 运行注册了的回调函数
            for callback in callbacks:
                self._run_callback(callback)

            if self._callbacks:
                poll_timeout = 0.0

            # 超时处理
            if self._timeouts:
                now = time.time()
                while self._timeouts:
                    if self._timeouts[0].callback is None:
                        # the timeout was cancelled
                        heapq.heappop(self._timeouts)
                    elif self._timeouts[0].deadline <= now:
                        timeout = heapq.heappop(self._timeouts)
                        self._run_callback(timeout.callback)
                    else:
                        milliseconds = self._timeouts[0].deadline - now
                        poll_timeout = min(milliseconds, poll_timeout)
                        break

            if not self._running:
                break

            if self._blocking_signal_threshold is not None:
                # clear alarm so it doesn't fire while poll is waiting for
                # events.
                signal.setitimer(signal.ITIMER_REAL, 0, 0)

            # epoll阻塞，当有事件通知或超时返回event_pairs
            try:
                event_pairs = self._impl.poll(poll_timeout)
            except Exception, e:
                # Depending on python version and IOLoop implementation,
                # different exception types may be thrown and there are
                # two ways EINTR might be signaled:
                # * e.errno == errno.EINTR
                # * e.args is like (errno.EINTR, 'Interrupted system call')
                if (getattr(e, 'errno', None) == errno.EINTR or
                    (isinstance(getattr(e, 'args', None), tuple) and
                     len(e.args) == 2 and e.args[0] == errno.EINTR)):
                    continue
                else:
                    raise

            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL,
                                 self._blocking_signal_threshold, 0)

            # Pop one fd at a time from the set of pending fds and run
            # its handler. Since that handler may perform actions on
            # other file descriptors, there may be reentrant calls to
            # this IOLoop that update self._events
            # 对epoll返回event_pairs事件的处理
            # _events中存储的是epoll_wait 执行后的待处理事件字典
            self._events.update(event_pairs)
            while self._events:
                fd, events = self._events.popitem()
                try:
                    self._handlers[fd](fd, events)
                except (KeyboardInterrupt, SystemExit):
                    raise
                except (OSError, IOError), e:
                    if e.args[0] == errno.EPIPE:
                        # Happens when the client closes the connection
                        pass
                    else:
                        logging.error("Exception in I/O handler for fd %d",
                                      fd, exc_info=True)
                except:
                    logging.error("Exception in I/O handler for fd %d",
                                  fd, exc_info=True)
        # reset the stopped flag so another start/stop pair can be issued
        self._stopped = False
        if self._blocking_signal_threshold is not None:
            signal.setitimer(signal.ITIMER_REAL, 0, 0)
```
# iostream.py
IOStream主要提供的功能就是异步的读写操作,IOStream中定义了_read_buffer和_write_buffer，这样我们就不用直接读写socket，进而实现异步读写，这些操作都封装在IOStream类中。概括来说，IOStream对socket的读写做了一层封装，通过使用两个缓冲区，实现对socket的异步读写。  
初始化的时候建立了两个buffer，然后把自己的socket放到了io_loop。这样，当这个socket有读写的时候，就会回调到注册的事件self._handle_events。_handle_events会根据事件的类型来判断是执行读还是写

# httpserver.py
在HTTPServer实例化过程中保存了处理请求的Application对象。调用listen方法，listen方法调用了2个方法
- bind 
bind方法是 一个标准的服务器启动过程，先建立socket，然后bind listen，最后将socket加入到io_loop，注册的事件是ioloop.IOLoop.READ，也就是读事件。回调函数为_handle_events, 一旦listen socket可读, 说明客户端请求到来, 然后调用_handle_events接受客户端的请求
- start

```
if not self.io_loop:
    self.io_loop = ioloop.IOLoop.instance() # 创建 ioloop 对象
for fd in self._sockets.keys(): # 把socket字典中所有的socket加入到ioloop handler列表中，并监听可读事件，回调函数为_handle_events
    self.io_loop.add_handler(fd, self._handle_events, ioloop.IOLoop.READ)
```

然后来看一下_handle_events是怎么实现的

```python
def _handle_events(self, fd, events):
        # 当接受到服务端socket的可读事件时，建立连接
        while True:
            try:
                # 客户端的socket, 以及客户端的地址
                connection, address = self._sockets[fd].accept()
            except socket.error, e:
                if e.args[0] in (errno.EWOULDBLOCK, errno.EAGAIN):
                    return
                raise
            ...
            try:
                if self.ssl_options is not None:
                    stream = iostream.SSLIOStream(connection, io_loop=self.io_loop)
                else:
                    stream = iostream.IOStream(connection, io_loop=self.io_loop)
                # 创建 HTTPonnection连接对象，处理http请求
                HTTPConnection(stream, address, self.request_callback,
                               self.no_keep_alive, self.xheaders)
            except:
                logging.error("Error in connection callback", exc_info=True)
```
_handle_events方法实际上就是Reactor模型概率中的concrete event handler
# HTTPonnection
这个类用来处理http的请求，包括读取http请求头，读取post过来的数据，调用用户自定义的处理方法，以及把响应数据写给客户端socket。

HTTPconnection中处理的过程是
iostream通过socket.recv()把数据读入_read_buffer, 然后调用_read_from_buffer(), 当发现_read_buffer中有'/r/n/r/n'就会调用HTTPConnection._on_headers来处理http header，然后解析_read_buffer提取出http的method、version等信息生成_request，再根据Content-Length获取request body。调用application， 并把request作为参数传递，_request中已包含 Http Request中的所有信息，最后调用Application的_call__方法。

```python
class HTTPConnection(object):
    """Handles a connection to an HTTP client, executing HTTP requests.

    We parse HTTP headers and bodies, and execute the request callback
    until the HTTP conection is closed.
    """
    def __init__(self, stream, address, request_callback, no_keep_alive=False,
                 xheaders=False):
        self.stream = stream
        self.address = address
        self.request_callback = request_callback
        self.no_keep_alive = no_keep_alive
        self.xheaders = xheaders
        self._request = None
        self._request_finished = False
        # Save stack context here, outside of any request.  This keeps
        # contexts from one request from leaking into the next.
        # 发现_read_buffer中有'/r/n/r/n'就会调用HTTPConnection._on_headers来处理http header
        self._header_callback = stack_context.wrap(self._on_headers)
        self.stream.read_until(b("\r\n\r\n"), self._header_callback)

    def write(self, chunk):
        """Writes a chunk of output to the stream."""
        assert self._request, "Request closed"
        if not self.stream.closed():
            self.stream.write(chunk, self._on_write_complete)

    def finish(self):
        """Finishes the request."""
        assert self._request, "Request closed"
        self._request_finished = True
        if not self.stream.writing():
            self._finish_request()

    def _on_write_complete(self):
        if self._request_finished:
            self._finish_request()

    def _finish_request(self):
        if self.no_keep_alive:
            disconnect = True
        else:
            connection_header = self._request.headers.get("Connection")
            if self._request.supports_http_1_1():
                disconnect = connection_header == "close"
            elif ("Content-Length" in self._request.headers
                    or self._request.method in ("HEAD", "GET")):
                disconnect = connection_header != "Keep-Alive"
            else:
                disconnect = True
        self._request = None
        self._request_finished = False
        if disconnect:
            self.stream.close()
            return
        self.stream.read_until(b("\r\n\r\n"), self._header_callback)

    def _on_headers(self, data):
        try:
            data = native_str(data.decode('latin1'))
            eol = data.find("\r\n")
            start_line = data[:eol]
            try:
                method, uri, version = start_line.split(" ")
            except ValueError:
                raise _BadRequestException("Malformed HTTP request line")
            if not version.startswith("HTTP/"):
                raise _BadRequestException("Malformed HTTP version in HTTP Request-Line")
            headers = httputil.HTTPHeaders.parse(data[eol:])
            # 解析_read_buffer,生成_reqeust
            self._request = HTTPRequest(
                connection=self, method=method, uri=uri, version=version,
                headers=headers, remote_ip=self.address[0])

            content_length = headers.get("Content-Length")
            if content_length:
                content_length = int(content_length)
                if content_length > self.stream.max_buffer_size:
                    raise _BadRequestException("Content-Length too long")
                if headers.get("Expect") == "100-continue":
                    self.stream.write("HTTP/1.1 100 (Continue)\r\n\r\n")
                # 根据Content-Length获取request body
                self.stream.read_bytes(content_length, self._on_request_body)
                return

            self.request_callback(self._request)
        except _BadRequestException, e:
            logging.info("Malformed HTTP request from %s: %s",
                         self.address[0], e)
            self.stream.close()
            return

    def _on_request_body(self, data):
        self._request.body = data
        content_type = self._request.headers.get("Content-Type", "")
        if self._request.method in ("POST", "PUT"):
            if content_type.startswith("application/x-www-form-urlencoded"):
                arguments = parse_qs_bytes(native_str(self._request.body))
                for name, values in arguments.iteritems():
                    values = [v for v in values if v]
                    if values:
                        self._request.arguments.setdefault(name, []).extend(
                            values)
            elif content_type.startswith("multipart/form-data"):
                fields = content_type.split(";")
                for field in fields:
                    k, sep, v = field.strip().partition("=")
                    if k == "boundary" and v:
                        httputil.parse_multipart_form_data(
                            utf8(v), data,
                            self._request.arguments,
                            self._request.files)
                        break
                else:
                    logging.warning("Invalid multipart/form-data")
        # 调用application， 并把request作为参数传递，_request中已包含 Http Request中的所有信息
        # 调用Application的__call__方法
        self.request_callback(self._request)
```
# web.py
HTTPConnection在_on_request_body方法中调用到了Application的__call__方法，__call__方法中会匹配url，并调用RequestHandler._exexute()方法，在_execute中会执行具体的处理请求的业务方法。最后调用RequestHandler.flush() 通过iostream发送数据到客户端，一次请求就完成了。

```python
def __call__(self, request):
        """Called by HTTPServer to execute the request."""
        transforms = [t(request) for t in self.transforms]
        handler = None
        args = []
        kwargs = {}
        handlers = self._get_host_handlers(request)
        ...
        # 处理函数
        handler._execute(transforms, *args, **kwargs)
        return handler
        
def _execute(self, transforms, *args, **kwargs):
        """Executes this request with the given output transforms."""
        self._transforms = transforms
        with stack_context.ExceptionStackContext(
            self._stack_context_handle_exception):
            if self.request.method not in self.SUPPORTED_METHODS:
                raise HTTPError(405)
            # If XSRF cookies are turned on, reject form submissions without
            # the proper cookie
            if self.request.method not in ("GET", "HEAD", "OPTIONS") and \
               self.application.settings.get("xsrf_cookies"):
                self.check_xsrf_cookie()
            # 预处理的地方，可以重写来做一些预处理的工作，                
            self.prepare()
            if not self._finished:
                args = [self.decode_argument(arg) for arg in args]
                kwargs = dict((k, self.decode_argument(v, name=k))
                              for (k,v) in kwargs.iteritems())
                # 具体执行的地方
                getattr(self, self.request.method.lower())(*args, **kwargs)
                if self._auto_finish and not self._finished:
                    # 调用flush, 通过iostream发送数据到客户端
                    self.finish()
```
tornado的源码大概就是这样，其中还有一些地方我也没有完全看懂，等以后再仔细研究。