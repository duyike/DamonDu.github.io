---
layout:     post
title:      "Java Web 开发(一)：简易税务计算器"
subtitle:   "—— 基于 Java + JS 实现"
date:       2017-09-29 13:35:00
author:     "Damon To"
header-style: text
catalog:    true
tags:
    - Java
    - Java Web
    - Javascript
---

> 2018.03.02更新：本项目实现了一个简单的税务计算器，而核心上则是实现了一个极其简单甚至简陋的 Http Server。在学习了 JVM，Java concurrent 这些基础知识并且了解了一些框架技术后，回过头来看几个月前做的这个项目，就会觉得当时的想法有些过于简单和直接，而且也发现项目中充斥着许多 Bad Smelling 的代码。但是同时，这个项目十分真实地展现初学者在面对更加复杂的开发时会遇到的难处与困惑。也正是因为有这些困惑的存在，各种框架技术和开发脚手架才有其存在的价值。理解这些新手会有的困惑，可以从另一个角度去理解框架技术的作用与意义。

## 项目简介

Tax Calculator：一个基于 Java + Javascript 实现的个人税务计算器。基本功能是：输入收入、计算并输出应缴税金、修改税务计算规则等。

* 开发环境： JRE 1.8.0_131 (Win 10)
* 技术栈：
  * 前端： Jquery + Bootstrap + Ajax
  * 后端： Java

## 开发过程

### (一) 架构设计

项目架构如下 UML 图所示：

![](/img/in-post/2017-09-29-TaxCalculator/rear-end-UML.jpg)

UML图中省略了 Web UI 的具体实现。其中需要注意的：

1. 对于 Tax Table，一开始我将 Tax Table 的数据与方法写在同一个类中，后来开发过程中发现应该是将其分开：**将 tax table 的数据作为一个类并设置 get/set 修改器，而具体的对 tax table 的修改，加载，保存等方法写在另外一个命名为 RuleManager 的类中。**这样的好处是低耦合，工程拓展性更好。试想如果工程需要实现多种税务规则共存，或者对税务规则文件的存在多种存储格式，在这些场景下这样的设计能减少修改代码的工作量。另外从语义的角度，这样的设计也更加清晰，代码可读性更高。
2. **单独设计 Control 类而不是直接将 main 函数所在的类作为控制类。**同样是处于拓展性考虑，像在此次实验中我设计了两套 UI：命令行 UI 与 Web UI，这样这两个 main 函数就只需要调用 Control 类的方法，两套 UI 互相独立互不影响。
3. **严谨地设置单实例类型。**

在本次实验过程中，由于对 Java 及相关工具与 API 的不熟悉，在实现过程中走了不少的弯路，其中不乏有一些不止在 Java ， 而且在整个计算机领域都适用的开发经验和体会，所以想在此记录一下。

### (二) 数据存储： Java 处理 XML 文件

本次实验我使用 XML 文件来储存数据，主要是考虑到 XML 平台支持性好，方便后续 Web 的开发，而且数据量不多而且语义性强，用 XML 来处理比较合适。

```java
public void load() {
		Document document = parse("resource/config.xml");
		try {
			int numOfLevel = Integer.parseInt(document.getElementsByTagName("numOfLevel").item(0).getTextContent());
			double firstThreshold = Integer.parseInt(document.getElementsByTagName("firstThreshold").item(0).getTextContent());
			double threshold[] = new double[numOfLevel];
			double taxRate[] = new double[numOfLevel];
			Node thresholdNode = document.getElementsByTagName("threshold").item(0);
			Node taxRateNode = document.getElementsByTagName("taxRate").item(0);
			if (thresholdNode.getNodeType() == Node.ELEMENT_NODE && taxRateNode.getNodeType() == Node.ELEMENT_NODE) {
				Element thresholdElement = (Element) thresholdNode;
				Element taxRateElement = (Element) taxRateNode;
				for (int i = 0; i < numOfLevel; i++) {
					threshold[i] = Double.parseDouble(thresholdElement.getElementsByTagName("item").item(i).getTextContent());
					taxRate[i] = Double.parseDouble(taxRateElement.getElementsByTagName("item").item(i).getTextContent());
				}
			}
			rule = new TaxRule(numOfLevel, firstThreshold, threshold, taxRate);
		}
		catch (Exception e) {
			System.out.println("Wrong format of config.xml!");
		}
	}
```

如上代码，实现了从XML文件中加载数据。这里我是用的是最基础的`javax.xml`的最原始的 DOM 操作。可以看到代码量还是比较大的。如果是在数据量比较大的其他项目，我想还是使用较为主流的 JDOM 或者 DOM4J。

### (三) 使用 Java 实现简单的服务器工作

在本次实验中我使用`java.net.Socket`和`java.net.ServerSocket`实现了一个简单的服务器，并根据前端的需求提供相应的接口。本来以为会很简单，但是实际开发中遇到了没有预料到的困难，出了很多莫名其妙的 bug。

#### Request Header 的解析

Request Header 的基本格式如下：

```
GET /index.html HTTP/1.1
Host: localhost:8000
Connection: keep-alive
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36
Upgrade-Insecure-Requests: 1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.8
```

获取到 Header 字符串后，就可以进行字符串解析，一个简单的例子如下：

```java
//解析 Request Method
private String parseMethod(String requestString) {
	int index;
	index = requestString.indexOf(' ');
	if (index != -1) { 
		return requestString.substring(0, index);
	}
	return null;
}
```

解析其他的信息也于此大同小异，在此不列举，如有需要可以在上文提到的代码地址中查看源代码。由一开始对需要从 Request Header 中获取什么信息没有一个大概的想法，导致最终 Request 类很臃肿，各种 Parser 中存在不少的重复代码。而且解析应该尽量使用正则表达式等效率较高地工具，大量用字符串的相应工具会导致实现臃肿。

#### Response Header 的构造

```java
public void sendStaticResource() throws IOException {
  byte[] bytes = new byte[BUFFER_SIZE];
  FileInputStream fis = null;

  try {
      System.out.println("method is:" + request.getMethod());
      File file = new File(HttpServer.WEB_ROOT, request.getUri());
      System.out.println(file.exists());
      if (file.exists()) {
        fis = new FileInputStream(file);
        String type = request.parseType();
        String header = "HTTP/1.1 200\r\n" + "Content-Type: text/" + type + "\r\n" + "Content-Length: " + file.length() + "\r\n" + "\r\n";
        output.write(header.getBytes());
        int ch = fis.read(bytes, 0, BUFFER_SIZE);
        while (ch != -1) {
          output.write(bytes, 0, ch);
          ch = fis.read(bytes, 0, BUFFER_SIZE);
      }
  } 
      //... 
}
```

Response类中的 `=sendStaticResource()` 方法实现了根据 Request 的分析结果，生成对应的 response。要构造一个合法的 Response Header 以保证成功的 response，这次实验我使用简单的字符串拼接的方法，这种方法比较朴素和繁杂，可以使用对应功能的库方法来保证实现的简洁性和准确性。

#### 将文件流或字符串流转二进制流

或许是由于 Response Header 不完全规范的缘故，我在传输 response body 时遇到了各种各样的 Bug。最终迫于无奈选择将传输数据以明文字符串的形式直接写在 response body 中，待改进。

### (四) Web UI 实现

界面基于 Bootstrap 前端框架进行搭建。为了保证前端的数据能正常发送到服务器，以及能对服务器的响应做正常的处理，需要在 Javascript 代码中完善前端逻辑。

#### 加载 XML 数据到页面

```javascript
function loadXMLDoc(url){
	var xmlhttp;
	var numOfLevel, firstThreshold, threshold, taxRate, item, xml;
	if (window.XMLHttpRequest){// code for IE7+, Firefox, Chrome, Opera, Safari
  		xmlhttp=new XMLHttpRequest();
  	}
	else{// code for IE6, IE5
  		xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
  	}
  	xmlhttp.overrideMimeType("text/xml");
	xmlhttp.onreadystatechange = function(){
  		if (xmlhttp.readyState == 4 && xmlhttp.status == 200){
   			txt = "<thead><tr><th>Level</th><th>Threshold</th><th>Tax Rate</th></tr></thead><tbody>";
    		numOfLevel = parseInt(xmlhttp.responseXML.documentElement.getElementsByTagName("numOfLevel")[0].innerHTML);
    		firstThreshold = xmlhttp.responseXML.documentElement.getElementsByTagName("firstThreshold")[0].innerHTML;
			threshold = xmlhttp.responseXML.documentElement.getElementsByTagName("threshold")[0];
			taxRate = xmlhttp.responseXML.documentElement.getElementsByTagName("taxRate")[0];
			for (var i = 0; i < numOfLevel; i++) {
				thresholdItem = threshold.getElementsByTagName("item")[i].innerHTML;
				taxRateItem = taxRate.getElementsByTagName("item")[i].innerHTML;
				txt = txt + "<tr><th scope='row'>" + (i + 1) + "</th><td><input type='text' class='form-control' readonly='readonly' value='"+ thresholdItem + "'></td><td><input type='text' class='form-control' readonly='readonly' value='"+ taxRateItem + "'></td></tr>";
			}
		    txt=txt + "</tbody>";
		    $(".table").append(txt);
		    $("#num-of-level").attr("value", numOfLevel);
		    $("#first-threshold").attr("value", firstThreshold);
	    }
  	}
  	xmlhttp.open("GET",url,true);
	xmlhttp.send();
}
```

在 Ajax 中，`xmlhttp.responseXML`提供了针对 XML 文件的 DOM 操作，利用 DOM 操作加载 XML 文件数据。

#### 构建 XML 并 POST

当用户保存税务规则修改，需要将修改完的新税务表以 XML 形式 POST 给服务器，代码如下：

```javascript
$("#save_btn").click(function(){
	var numOfLevel = parseInt($("#num-of-level")[0].value);
	var firstThreshold = $("#first-threshold")[0].value;
	var threshold = new Array(numOfLevel);
	var taxRate = new Array(numOfLevel);
	for (var i = 0; i < numOfLevel; i++){
		var threshold_seletor = ".table > tbody > tr:nth-child(" + (i + 1) + ") > td:nth-child(2) > input";
		var taxRate_seletor = ".table > tbody > tr:nth-child(" + (i + 1) + ") > td:nth-child(3) > input";
		threshold[i] = $(threshold_seletor)[0].value;
		taxRate[i] = $(taxRate_seletor)[0].value;
	}
	var xml = `xml:<taxRule><firstThreshold>${firstThreshold}</firstThreshold><numOfLevel>${numOfLevel}</numOfLevel><threshold>`;
	for (var i = 0; i < numOfLevel; i++){
		xml = xml + `<item>${threshold[i]}</item>`;
	}
	xml = xml + "</threshold><taxRate>";
	for (var i = 0; i < numOfLevel; i++){
		xml = xml + `<item>${taxRate[i]}</item>`;
	}
	xml = xml + "</taxRate></taxRule>";
	$.post("/", xml, function(){
		console.log("post");
	});
});
```

## 项目总结

第一次尝试开发一个简单的 Hybrid App。全栈开发是几乎每个开发者心中的梦想，在开发过程中，从一开始觉得很简单，到后来遇到了许多困难，由于不熟悉 Java API 走进了许多坑，最后在爬坑掉坑的过程中收获了很多切切实实。开发一个 Hybrid App 要求开发者在开发前做更多更细节的架构设计工作，你需要考虑你的数据如何存储，取出与处理，在这个流通过程中什么架构才能低耦合地进行完善地处理。另外，细节方面，细化到每个需求的技术实现选择哪一个库，选择哪一个 API，选择什么样的数据格式，都是值得斟酌考虑的。