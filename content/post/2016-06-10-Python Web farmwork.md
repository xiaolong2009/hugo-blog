---
date: 2016-06-10T11:46:53+08:00
title: Python Web framework
tags: ["python"]
---

上一篇博客说了wsgi的的内容，并简单编写了一个乞丐版的web framework，这次详细说一下怎么设计一个功能相对完善的web framework  
在开始看本篇博客前，需要先掌握cookie、session以及wsgi的内容，不然理解起来会比较吃力

---
一个web框架需要包括如下几个功能
1. 定义全局的http错误、常用header字典、response状态码字典
2. 处理request、生成response header 即框架需要一个全局的request和response对象对resquest和response进行处理
3. url路由 即通过web server传递过来的环境变量信息，确定最终调用哪个函数来进行处理
4. 拦截器 即做请求的预处理
5. 模板引擎 即函数处理完成要在浏览器中展示，需要模板引擎进行渲染
6. 暴露一个WSGI入口处理函数给web server调用

下面我们以廖雪峰实战篇里web框架为例，展开来看一下具体这些功能都是怎么实现的  
web.py 一共有1000行左右的代码，其实做的就是咱们说的这6个事情，下面一件一件来看

==1 定义全局的http错误、常用header字典、response状态码字典==

```python
_RESPONSE_STATUSES = {
    # Informational
    100: 'Continue',
    101: 'Switching Protocols',
    102: 'Processing',

    # Successful
    200: 'OK',
    201: 'Created',
    202: 'Accepted',
    ...
}

# 匹配response 状态码
_RE_RESPONSE_STATUS = re.compile(r'^\d{3}(\ \w+)?')

_RESPONSE_HEADERS = (
    'Accept-Ranges',
    'Age',
    'Allow',
    'Cache-Control',
    ...
)

_RESPONSE_HEADER_DICT = dict(zip(map(lambda x: x.upper(), _RESPONSE_HEADERS), _RESPONSE_HEADERS))

# 自定义header 用来显示服务器信息 可要可不要
_HEADER_X_POWERED_BY = ('X-Powered-By', 'transwarp/1.0')

# http 错误类
class HttpError(Exception):
    '''
    >>> e = HttpError(404)
    >>> e.status
    '404 Not Found'
    '''
    def __init__(self, code):
        super(HttpError, self).__init__()
        self.status = '%d %s' % (code, _RESPONSE_STATUSES[code])

    def headr(self, name, value):
        if not hasattr(self, '_header'):
            self._headr = [_HEADER_X_POWERED_BY]
        self._header.append((name, value))

    @property
    def headers(self):
        if hasattr(self, '_header'):
            return self._header
        return []

    def __str__(self):
        return self.status

    __repr__ = __str__

```

==2 处理request、生成response header 即框架需要一个全局的request和response对象对resquest和response进行处理==

```python
# 全局ThreadLocal对象
ctx = threading.local()

# request 对象
class Request(object):
    def __init__(self, environ):
        # 从环境变量里获取请求地址、请求方法等信息
        self._environ = environ

    def _get_raw_input(self):
        """
        用来生成Request的k-v字典
        """
        if not hasattr(self, '_raw_input'):
            def _convet(item):
                if isinstance(item, list):
                    return [_to_unicode(i.value) for i in item]
                if item.filename:
                    # 针对文件上传的情况
                    return MultiFile(item)
                return _to_unicode(item.value)
            # wsgi.input 类文件对象，保存了post请求中的body
            # cgi.FieldStorage() 生成字典，处理包括environ中的QueryString和wsgi.input中的参数，格式化
            fs = cgi.FieldStorage(fp=self._environ['wsgi.input'], environ=self._environ, keep_blank_values=True)
            d = {}
            for k in fs.keys():
                d[k] = _convet(fs[k])
            self._raw_input = d
        return self._raw_input

    # 在request body中取值，根据key返回value
    def get(self, key, default=None):
        """
        The same as request[key], but return default value if key is not found.

        >>> from StringIO import StringIO
        >>> r = Request({'REQUEST_METHOD':'POST', 'wsgi.input':StringIO('a=1&b=M%20M&c=ABC&c=XYZ&e=')})
        >>> r.get('a')
        u'1'
        >>> r.get('empty')
        >>> r.get('empty', 'DEFAULT')
        'DEFAULT'
        """
        return self._get_raw_input().get(key, default)

    # 返回key-value的dict
    def input(self, **kw):
        """

        >>> from StringIO import StringIO
        >>> r = Request({'REQUEST_METHOD':'POST', 'wsgi.input':StringIO('a=1&b=M%20M&c=ABC&c=XYZ&e=')})
        >>> r.get('a')
        u'1'
        >>> i = r.input(x=2008)
        >>> i.get('x')
        2008
        """
        copy = dict(**kw)
        for k, v in self._get_raw_input().iteritems():
            copy[k] = v[0] if isinstance(v, list) else v
        return copy

    # 返回URL的path，后面构建url路由的地方会用到
    @property
    def path_info(self):
        return urllib.unquote(self._environ.get('PATH_INFO', ''))

    # 返回request method
    @property
    def method(self):
        """
        >>> from StringIO import StringIO
        >>> r = Request({'PATH_INFO':'/', 'REQUEST_METHOD':'POST', 'wsgi.input':StringIO('a=1&b=M%20M&c=ABC&c=XYZ&e=')})
        >>> r.path_info
        '/'
        >>> r.method
        'POST'
        """
        return self._environ['REQUEST_METHOD']

    def _get_headers(self):
        if not hasattr(self, '_headers'):
            d = {}
            for k,v in self._environ.iteritems():
                if k.startswith('HTTP_'):
                    d[k[5:].replace('_', '-')] = _to_unicode(v)
            self._headers = d
        return self._headers

    # 返回HTTP header
    @property
    def headers(self):
        '''

        >>> r = Request({'HTTP_USER_AGENT': 'Mozilla/5.0', 'HTTP_ACCEPT': 'text/html'})
        >>> H = r.headers
        >>> H['ACCEPT']
        u'text/html'
        >>> H['USER-AGENT']
        u'Mozilla/5.0'
        >>> L = H.items()
        >>> L.sort()
        >>> L
        [('ACCEPT', u'text/html'), ('USER-AGENT', u'Mozilla/5.0')]
        '''
        return dict(**self._get_headers())
    
    # 从environ中获取cookie 并切分转换为字典格式 方便获取
    def _get_cookie(self):
        if not hasattr(self, '_cookie'):
            d = {}
            c_str = self._environ.get('HTTP_COOKIE')
            if c_str:
                for c in c_str.split(';'):
                    pos = c.find('=')
                    d[c[:pos].strip()] = urllib.unquote(_to_unicode(c[pos+1:]))
            self._cookie = d
        return self._cookie

    # 根据key返回Cookie value
    def cookie(self, name, default=None):
        return self._get_cookie().get(name, default)

    @property
    def cookies(self):
        '''
        >>> r = Request({'HTTP_COOKIE':'A=123; url=http%3A%2F%2Fwww.example.com%2F'})
        >>> r.cookies['A']
        u'123'
        >>> r.cookie('url')
        u'http://www.example.com/'
        '''
        return dict(**self._get_cookie())

# response对象
class Response(object):
    # 默认的返回的status 和 header
    def __init__(self):
        self._status = '200 OK'
        self._headers = {'CONTENT-TYPE': 'text/html; charset=utf-8'}

    @property
    def headers(self):
        """
        生成header，设置Cookie，Set-Cookie
        _RESPONSE_HEADER_DICT是一个保存了常用header的字典
        """
        L = [(_RESPONSE_HEADER_DICT.get(k, k), v) for k, v in self._headers.iteritems()]
        if hasattr(self, '_cookie'):
            for v in self._cookie.itervalues():
                # 最终在这里设置cookie
                L.append(('Set-Cookie', _to_str(v)))
        L.append(_HEADER_X_POWERED_BY)
        return L

    # 设置header
    def set_header(self, name, value):
        key = name.upper()
        if not key in _RESPONSE_HEADER_DICT:
            key = name
        self._headers[key] = _to_str(value)

    def unset_header(self, name):
        '''
        Unset header by name and value.

        >>> r = Response()
        >>> r.content_type
        'text/html; charset=utf-8'
        >>> r.unset_header('CONTENT-type')
        >>> r.content_type
        '''
        key = name.upper()
        if not key in _RESPONSE_HEADER_DICT:
            key = name
        if key in self._headers:
            del self._headers[key]

    @property
    def content_type(self):
        '''
        Get content type from response. This is a shortcut for header('Content-Type').

        >>> r = Response()
        >>> r.content_type
        'text/html; charset=utf-8'
        >>> r.content_type = 'application/json'
        >>> r.content_type
        'application/json'
        '''
        return self._headers.get('CONTENT-TYPE', None)

    # 设置content_type
    @content_type.setter
    def content_type(self, value):
        '''
        Set content type for response. This is a shortcut for set_header('Content-Type', value).
        '''
        if value:
            self.set_header('CONTENT-TYPE', value)
        else:
            self.unset_header('CONTENT-TYPE')

    # 设置Cookie，最后在headers方法里通过Set-Cookie设置cookie，并在start_response中返回给server
    def set_cookie(self, name, value, max_age=None, expires=None, path='/'):
        '''

        >>> r = Response()
        >>> r.set_cookie('company', 'Abc, Inc.', max_age=3600)
        >>> r._cookie
        {'company': 'company=Abc%2C%20Inc.; Max_Age=3600; Path=/'}
        '''
        if not hasattr(self, '_cookie'):
            self._cookie = {}
        l = ['%s=%s' % (urllib.quote(name), urllib.quote(value))]
        if not expires:
            if isinstance(expires, (float, int, long)):
                l.append('Expires=%s' % datetime.datetime.fromtimestamp(expires, UTC_0).strftime('%a, %d-%b-%Y %H:%M:%S GMT'))
            if isinstance(expires, (datetime.datetime, datetime.time)):
                l.append('Expires=%s' % expires.astimezone(UTC_0).strftime('%a, %d-%b-%Y %H:%M:%S GMT'))
        if isinstance(max_age, (int, long)):
            l.append('Max_Age=%s' % max_age)
        l.append('Path=%s' % path)
        self._cookie[name] = '; '.join(l)

    def delete_cookie(self, name):
        """
        删除Cookie实际就是把过期时间设置到当前时间之前
        """
        self.set_cookie(name, '__delete__', expires=0)

    # 设置status
    @property
    def status(self):
        return self._status

    @status.setter
    def status(self, value):

        if isinstance(value, (int, long)):
            if value>=100 and value<=900:
                st = _RESPONSE_STATUSES.get(value, '')
                if st:
                    self._status = '%d %s' % (value, st)
                else:
                    self._status = str(value)
            else:
                raise ValueError('bad response status code %d' % value)
        elif isinstance(value, basestring):
            if isinstance(value, unicode):
                self._status = value.encode('utf-8')
            if _RE_RESPONSE_STATUS.match(value):
                self._status = value
            else:
                raise ValueError('bad response status code %s' % value)
        else:
            raise TypeError('Bad type of response code.')

```

==3 4 5 6这几个步骤可以合在一起来说，贯穿了整个框架==  
入口是get_wsgi_application，也就是第6步，是整个框架的入口，fn_exec是实现拦截器和url路由的函数入口， 也就是第三第四步，_build_interceptor_chain将url路由和拦截器合在一起进行，因为拦截器其实也可以看做路由的一部分，这里截器接受一个next函数，这样，一个拦截器可以决定调用next()继续处理请求还是直接返回，理解这一点也很重要，经过url路由后就到了具体函数的处理部分，r就是函数最终的返回值，如果r是Template类型的就会用模板引擎来进行渲染，也就是第5步，具体的渲染逻辑是在jinja2中实现（其实就是字符串的替换，这里不做讨论，后面会详细说一下模板引擎是怎么写的），最后通过start_response将状态码和header返回给web server，然后将渲染后的内容返回 作为resonse body，整个框架的任务就结束了。

```python
# 定义GET
def get(path):
    '''
    >>> @get('/hello')
    ... def test():
    ...     return None
    >>> test.__route__
    '/hello'
    >>> test.__method__
    'GET'
    '''
    def _decorate(func):
        func.__route__ = path
        func.__method__ = 'GET'
        return func
    return _decorate

# 定义post
def post(path):
    '''
    >>> @post('/admin')
    ... def test():
    ...     return None
    >>> test.__method__
    'POST'
    >>> test.__route__
    '/admin'
    '''
    def _decorate(func):
        func.__route__ = path
        func.__method__ = 'POST'
        return func
    return _decorate

# 匹配只要是有:的就认为是有参数
_re_route = re.compile(r'(\:[a-zA-Z_]\w*)')


# 构建参数分离的正则表达式:abc/ 即名为abc的参数
def _build_regex(path):
    """

    >>> _build_regex('/:path') == '^\\/(?P<path>[^\\/]+)$'
    True
    >>> _build_regex('/:id/:user/commout/') == '^\\/(?P<id>[^\\/]+)\\/(?P<user>[^\\/]+)\\/commout\\/$'
    True
    >>> _build_regex('/path/to/:file') == '^\\/path\\/to\\/(?P<file>[^\\/]+)$'
    True
    """
    re_list = ['^']
    is_var = False
    for v in _re_route.split(path):
        if is_var:
            varname = v[1:]
            re_list.append('(?P<%s>[^\\/]+)' % varname)
        else:
            s = ''
            for c in v:
                if c>='0' and c<='9':
                    s = s + c
                elif c>='a' and c<='z':
                    s = s + c
                elif c>='A' and c<='Z':
                    s = s + c
                else:
                    s = s + '\\' + c # 加\\是转移为\在r'\-'与'\\-'等同，在正则表达式下r'\-'等同'-'
            re_list.append(s)
        is_var = not is_var
    re_list.append('$')
    return ''.join(re_list)


# 定制url路由
class Route(object):
    """
    >>> @get('/:id/active')
    ... def test(arg):
    ...     print 'test', arg
    >>> r = Route(test)
    >>> r.is_static
    False
    >>> r.match('/tim/active')
    ('tim',)
    >>> r('tim')
    test tim
    """
    def __init__(self, func):
        self.path = func.__route__
        self.method = func.__method__
        self.is_static = _re_route.search(self.path) is None # 判断path是否有参数
        if not self.is_static:
            self.route = re.compile(_build_regex(self.path)) # 构造参数匹配正则表达式
        self.func = func

    def match(self, path):
        m = self.route.match(path)
        if m:
            return m.groups()
        return None

    def __call__(self, *args):
        return self.func(*args)


def _static_file_generator(fpath):
    BLOCK_SIZE = 8192
    with open(fpath, 'rb') as f:
        block = f.read(BLOCK_SIZE)
        while block:
            yield block
            block = f.read(BLOCK_SIZE)


# 静态文件路由
class StaticFileRoute(object):

    def __init__(self):
        self.method = 'GET'
        self.is_static = False
        self.route = re.compile('^/static/(.+)$')

    def match(self, url):
        if url.startswith('/static/'):
            return (url[1:], )
        return None

    def __call__(self, *args):
        fpath = os.path.join(ctx.application['document_root'], args[0])
        if not os.path.isfile(fpath):
            raise notfound()
        fext = os.path.splitext(fpath)[1]
        ctx.response.content_type = mimetypes.types_map.get(fext.lower(), 'application/octet-stream')
        return _static_file_generator(fpath)

def favicon_handler():
    return static_file_handler('/favicon.ico')

# 定义模版
def view(path):
    '''
    >>> @view('index.html')
    ... def test():
    ...     return dict(a=123,b=234)
    >>> t = test()
    >>> t.template_name
    'index.html'
    >>> t.model
    {'a': 123, 'b': 234}
    '''
    def _decorater(func):
        @functools.wraps(func)
        def _wrapper(*args, **kw):
            r = func(*args, **kw)
            if isinstance(r, dict):
                return Template(path, **r)
            raise ValueError('func result must be dict')
        return _wrapper
    return _decorater

_RE_INTERCEPTROR_STARTS_WITH = re.compile(r'^([^\*\?]+)\*?$')
_RE_INTERCEPTROR_ENDS_WITH = re.compile(r'^\*([^\*\?]+)$')


# 设置拦截器的匹配正则表达式
def _build_pattern_fn(pattern):
    m = _RE_INTERCEPTROR_STARTS_WITH.match(pattern)
    if m:
        return lambda p: p.startswith(m.group(1))
    m = _RE_INTERCEPTROR_ENDS_WITH.match(pattern)
    if m:
        return lambda p: p.endswith(m.group(1))
    raise ValueError('pattern define interceptor')


# 定义拦截器
def interceptor(pattern='/'):
    '''
    >>> @interceptor('*/api')
    ... def test():
    ...     return None
    >>> test.__interceptor__('hello/api')
    True
    >>> @interceptor('/api/*')
    ... def func():
    ...     return None
    >>> func.__interceptor__('/api/get/')
    True
    '''
    def _decorator(func):
        func.__interceptor__ = _build_pattern_fn(pattern)
        return func
    return _decorator


# 工具函数，匹配path时拦截器起作用
def _build_interceptor_fn(func, next):
    def _wrapper():
        # 拦截器函数匹配path，先调用拦截器函数，在拦截器函数中调用next则代表继续处理请求
        if func.__interceptor__(ctx.request.path_info):
            return func(next)
        # 继续处理请求
        else:
            return next()
    return _wrapper


# 工具函数，遍历所有的拦截器，如果匹配到path就用拦截器包住
def _build_interceptor_chain(last_fn, *interceptor):
    '''
    >>> def target():
    ...     print 'test'
    ...     return 123
    >>> @interceptor('/')
    ... def f1(next):
    ...     print 'before f1'
    ...     try:
    ...         return next()
    ...     finally:
    ...         print 'after f1'
    >>> @interceptor('/api')
    ... def f2(next):
    ...     print 'before f2'
    ...     return next()
    >>> @interceptor('/')
    ... def f3(next):
    ...     print 'before f3'
    ...     try:
    ...         return next()
    ...     finally:
    ...         print 'after f3'
    >>> class Test(object):
    ...     path_info = '/api'
    >>> ctx.request = Test()
    >>> chain = _build_interceptor_chain(target, f1, f2, f3)
    >>> chain()
    before f1
    before f2
    before f3
    test
    after f3
    after f1
    123
    >>> class Test2(object):
    ...     path_info = '/test'
    >>> ctx.request = Test2()
    >>> chain = _build_interceptor_chain(target, f1, f2, f3)
    >>> chain()
    before f1
    before f3
    test
    after f3
    after f1
    123
    '''
    L = list(interceptor)
    L.reverse()
    fn = last_fn
    for f in L:
        fn = _build_interceptor_fn(f, fn)
    return fn

# 模版与model映射类
class Template(object):
    def __init__(self, template_name, **kw):
        self.template_name = template_name
        self.model = dict(**kw)

# 定义模版引擎
class TemplateEngine(object):
    def __call__(self, path, model):
        pass

# 缺省使用
class Jinja2TemplateEngine(TemplateEngine):
    def __init__(self, templ_dir, **kw):
        from jinja2 import Environment, FileSystemLoader
        self._env = Environment(loader=FileSystemLoader(templ_dir), **kw)

    def add_filter(self, name, fn_filter):
        self._env.filters[name] = fn_filter

    def __call__(self, path, model):
        return self._env.get_template(path).render(**model).encode('utf-8')


class WSGIApplication(object):
    def __init__(self, document_root=None, **kw):

        self._running = False

        self._document_root = document_root

        self._get_static = {}
        self._post_static = {}

        self._get_dynamic = []
        self._post_dynamic = []

        self._interceptors = []

    def _check_not_running(self):
        if self._running:
            raise RuntimeError("can't modify WSGIApplication when it is running")

    # 添加一个URL定义
    def add_url(self, func):
        self._check_not_running()
        route = Route(func)
        if route.is_static:
            if route.method == 'GET':
                self._get_static[route.path] = route
            elif route.method == 'POST':
                self._post_static[route.path] = route
        else:
            if route.method == 'GET':
                self._get_dynamic.append(route)
            elif route.method == 'POST':
                self._post_dynamic.append(route)
        logging.info('add route %s ' % route.method+':'+route.path)

    # 从模块添加url
    def add_model(self, mod):
        self._check_not_running()
        m = mod
        logging.info('add model %s' % mod.__name__)
        for name in dir(m):
            fn = getattr(m, name)
            if hasattr(fn, '__route__') and hasattr(fn, '__method__'):
                self.add_url(fn)

    # 添加一个Interceptor定义
    def add_interceptor(self, func):
        self._check_not_running()
        self._interceptors.append(func)
        logging.info('add interceptor %s ' % func.func_name)

    # 设置TemplateEngine
    @property
    def template_engine(self):
        self._template_engine

    @template_engine.setter
    def template_engine(self, engine):
        self._temlate_engine = engine

    # 返回WSGI处理函数
    def get_wsgi_application(self, debug=False):
        self._check_not_running()
        if debug:
            self._get_dynamic.append(StaticFileRoute())
        self._running = True

        _application = dict(document_root=self._document_root) # 这里只在取静态文件时有用，传入一个目录

        # 根据method与path路由获取处理函数，分为一般的和带参数的
        def fn_route():
            path = ctx.request.path_info
            method = ctx.request.method
            if method == 'GET':
                fn = self._get_static.get(path, None)
                if fn:
                    return fn()
                for fn in self._get_dynamic:
                    args = fn.match(path)
                    if args:
                        return fn(*args)
                raise notfound()
            if method == 'POST':
                fn = self._post_static.get(path, None)
                if fn:
                    return fn()
                for fn in self._post_dynamic:
                    args = fn.match(path)
                    if args:
                        return fn(*args)
                raise notfound()
            return badrequest()

        # 对于处理函数包裹上拦截器规则
        fn_exec = _build_interceptor_chain(fn_route, *self._interceptors)

        # WSGI入口处理函数
        def wsgi(env, start_response):
            ctx.application = _application
            # env是web server传递过来的，用它来获取request的信息，并绑定在ctx上
            ctx.request = Request(env)
            response = ctx.response = Response()
            try:
                # r其实就是最终处理函数的返回值
                r = fn_exec()
                if isinstance(r, Template):
                    r = self._temlate_engine(r.template_name, r.model)
                elif isinstance(r, unicode):
                    r = r.encode('utf-8')
                if r is None:
                    r = []
                # 通过start_response函数，将状态码和header返给web server 进而到client
                start_response(response.status, response.headers)
                # 按照wsgi的约定，将可迭代对象返回，web server会遍历这个对象，作为response body
                return r
            except RedirectError, e:
                response.set_header('Location', e.location)
                start_response(e.status, response.headers)
                return []
            except HttpError, e:
                start_response(e.status, response.headers)
                return ['<html><body><h1>', e.status, '</h1></body></html>']
            except Exception, e:
                logging.exception(e)
                if not debug:
                    start_response('500 Internal Server Error', [])
                    return ['<html><body><h1>500 Internal Server Error</h1></body></html>']
                exc_type, exc_value, exc_traceback = sys.exc_info()
                fp = StringIO()
                traceback.print_exception(exc_type, exc_value, exc_traceback, file=fp)
                stacks = fp.getvalue()
                fp.close()
                start_response('500 Internal Server Error', [])
                return [
                   r'''<html><body><h1>500 Internal Server Error</h1><div style="font-family:Monaco, Menlo, Consolas, 'Courier New', monospace;"><pre>''',
                    stacks.replace('<', '&lt;').replace('>', '&gt;'),
                    '</pre></div></body></html>']
            finally:
                del ctx.request
                del ctx.response
        return wsgi
```
