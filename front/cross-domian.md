简介
----------

一个利用html5的跨域api postMessage解决一个系统中，多个iframe跨域通信交互的js库。

github地址 ：[cross-domain][1]

## 背景 ##

   最初公司只有一个系统来做销售，随着公司业务越来越多，搭建很多类似的系统（这些系统本来是没有任何关系的，每个系统目前都非常复杂）。

   由于目前公司战略有调整，原来的销售是针对某种产品，现在销售工作要针对客户进行多产品的销售促成，这样一个销售人员就需要打开各种系统进行业务操作，非常不方便，而且销售数据间不能有效传递，所就需要把各个不相关的系统整合在一起，实现跨业务线销售和数据共享。若将这想要将这些复杂系统整合在一起，无论是从人力物力上都是不太可能接受的。

   所以选择了使用iframe将各系统嵌入一个框架系统，各系统从物理上还是分开不变，而从逻辑上（从用户角度看就是一个系统）看起来是一个系统。
    
   然而各系统采用了不同的域名，与主框架系统和其它业务系统有跨域问题（若将所有域名改为同一域名下可能会产生一些系统间页面元素和样式的冲突）
    
   故采用了HTML5标准下的postMessage来解决该问题。

## 介绍 ##

 - 示意图

 1. http://a.com 是最外层主系统的页面，为master
 2. http://b.com 和 http://c.com 为被嵌入的子系统slave，当然也可以嵌入N个子系统

master和slave都是有各自的域名，由于浏览器的安全限定，两个iframe正常是不能进行数据交换和api调用的。当然有一些特殊方法如jsonp,iframe name等。如果想了解，可以看看我的另一篇文章[jsonp实现原理][2] 。

在HTML5中新增了postMessage方法，postMessage可以实现跨文档消息传输（Cross Document Messaging），Internet Explorer 8, Firefox 3, Opera 9, Chrome 3和 Safari 4都支持postMessage。postMessage api详细介绍，请查看 [postMessage][3]

示意图如下：
https://raw.githubusercontent.com/FreeLanderEden/cross-domain/master/example/principle.jpg
![clipboard.png](https://raw.githubusercontent.com/FreeLanderEden/cross-domain/master/example/principle.jpg)

## 提供的主要API ##

    js库提供了简洁的调用和提供接口的方法，介绍如下

- 接口调用（向其它iframe发送数据）
```
        /**
		 * 发送消息方法
		 * @param {String} componentName组件名称
	     * @param {String} method接口名称（对方通过API extends提供的接口名）
         * @param {Object} data数据
	     * @param {Function} callback回调
         */
		 send : function(componentName,method,data,callback,type);
```
- 提供接口（提供前端接口，可供其它iframe调用）

```
       /**
		* 扩展接口方法,供调用方send方法调用
		* @param {String} name接口名称
		* @param {Function} fun 接口方法
		*/
		extends : function(name,fun);
```
##  例子 ##

 - Master代码如下
启动http服务，http://localhost/cross-domain/example/master.html
```
<!DOCTYPE html>
<html>
<head>
	<title>Test Page</title>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <!--引入js库-->
	<script type="text/javascript" src="../src/cross-domain.js"></script>
</head>
<body>
	Test Page MASTER
	<button onclick="sendMesg1()">send data to Slave1</button>
	<button onclick="sendMesg2()">send data to Slave2</button>
	<br/>
    <!--slave1-->
	<iframe src="http://127.0.0.1/cross-domain/example/slave1.html" name="SLAVE1" id="SLAVE1"></iframe>
    <!--slave2-->
	<iframe src="http://127.0.0.1/cross-domain/example/slave2.html" name="SLAVE2" id="SLAVE2"></iframe>
	<div id="content"></div>
</body>

<script type="text/javascript">
	var me = CD.component.name;
	function genInfo(name){
		return {info : "Hello [" + name + "] , I am [" + me + "] Now at " + new Date()};
	}
    //调用SLAVE2的changeSlave1前端接口，接口参数为genInfo("SLAVE1")
	function sendMesg1 (argument) {
		CD.send("SLAVE1" , "changeSlave1" ,genInfo("SLAVE1") ,function(data){
			console.log("callback fire");
			writeHtml(data);
		});
	}
    //调用SLAVE2的changeSlave2前端接口，接口参数为genInfo("SLAVE2")
	function sendMesg2 (argument) {
		CD.send("SLAVE2" , "changeSlave2" , genInfo("SLAVE2"));
	}
    //MASTER提供接口，可供SLAVE1和SLAVE2调用
	CD.extends("changeMaster" , function(data){
		writeHtml(data.info);
	});
    //当SLAVE1和SLAVE2调用changeMaster接口时，会打印在MASTER的页面中
	function writeHtml(text){
		var content = document.getElementById("content");
		content.innerHTML += "<br/>" + text;
	}
	console.log(CD);
</script>
</html>
```

- Slave1

启动http服务，http://127.0.0.1/cross-domain/example/slave1.html

```
<!DOCTYPE html>
<html>
<head>
	<title>slave1</title>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<script type="text/javascript" src="../src/cross-domain.js"></script>
</head>
<body>
	<div id="main">
		I am salve1 frame
		<button onclick="sendMesg()">send data</button>
	</div>
	<div id="content"></div>
</body>
<script type="text/javascript">
	var me = CD.component.name;
	function genInfo(name){
		return {info : "Hello [" + name + "] , I am [" + me + "] Now at " + new Date()};
	}
    //调用MASTER的changeMaster接口，数据为genInfo("MESTER")
	function sendMesg (argument) {
		CD.send("MESTER" , "changeMaster" ,genInfo("MESTER"));
	}

    //提供前端接口changeSlave1，可供MASTER和其它SLAVE调用
	CD.extends("changeSlave1" , function(data){
		writeHtml(data.info);
		return "Slave1 changeSlave1 is called";
	});
	function writeHtml(text){
		var content = document.getElementById("content");
		content.innerHTML += "<br/>" + text;
	}
	console.log(CD);
</script>

</html>
```

- Slave2

启动http服务，http://127.0.0.1/cross-domain/example/slave2.html

```
<!DOCTYPE html>
<html>
<head>
	<title>slave2</title>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<script type="text/javascript" src="../src/cross-domain.js"></script>
</head>
<body>
	<div id="main">
		I am slave2 frame
		<button onclick="sendMesg()">send data</button>
	</div>
	<div id="content"></div>

</body>
<script type="text/javascript">
	var me = CD.component.name;
	function genInfo(name){
		return {info : "Hello [" + name + "] , I am [" + me + "] Now at " + new Date()};
	}
    //调用MASTER的changeMaster接口，数据为genInfo("MESTER")
	function sendMesg (argument) {
		CD.send("MESTER" , "changeMaster" ,genInfo("MESTER"));
	}

    //提供前端接口changeSlave2，可供MASTER和其它SLAVE调用
	CD.extends("changeSlave2" , function(data){
		writeHtml(data.info);
	});
	function writeHtml(text){
		var content = document.getElementById("content");
		content.innerHTML += "<br/>" + text;
	}
	console.log(CD);
</script>
</html>
```
- 交互效果

![clipboard.png](https://segmentfault.com/img/bVbcVDB)

## 应用案例 ##

某企业作业系统
![图片描述](https://segmentfault.com/img/bVbcVD4)


  [1]: https://github.com/FreeLanderEden/cross-domain
  [2]: https://my.oschina.net/toopoo/blog/204279
  [3]: https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage
