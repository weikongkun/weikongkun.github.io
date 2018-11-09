---
title: 手写自己的Tomcat
date: 2018-11-08 22:00:12
tags: Tomcat
categories: Tomcat

---

# 程序结构：

```java
+---config
|   +---ServletMapping.java
|   +---ServletMappingConfig.java
+---server
|   +---mytomcat
|      	+---MyTomcat.java（main）
|   +---MyRequest.java
|   +---MyReponse.java
+---servlets
|   +---DemoServlet_01.java
|   +---DemoServlet_HelloWorld.java
+---MyServlet.java
```



# 写自定义的http请求和响应封装类

请求封装类MyRequest.java

```java
package com.pywkk.server;

import java.io.IOException;
import java.io.InputStream;

public class MyRequest {
	private String url;
	private String method;
	//根据输入流得到http请求信息并解析出url和请求方法
	public MyRequest(InputStream inputStream) throws IOException {
		String httpRequest = "";
		byte[] httpRequestBytes = new byte[1024];
		int length = 0;
		if ((length = inputStream.read(httpRequestBytes)) > 0) {
			httpRequest = new String(httpRequestBytes, 0, length);
		}
		
		String httpLine = httpRequest.split("\n")[0];//得到请求消息行
		url = httpLine.split("\\s")[1];//得到url
		method = httpLine.split("\\s")[0];//得到方法类型
	}
	
	public String getUrl() {
		return url;
	}

	public String getMethod() {
		return method;
	}
}
```

请求封装类MyReponse.java

```java
package com.pywkk.server;

import java.io.IOException;
import java.io.OutputStream;

public class MyResponse {
	private OutputStream outputStream;
	public MyResponse(OutputStream outputStream) {
		this.outputStream = outputStream;
	}
	//将响应正文加入响应报文中
	public void write(String content) throws IOException {
		StringBuffer httpResponse = new StringBuffer();
		httpResponse.append("HTTP/1.1 200 ok\n")
			.append("Content-Type: text/html\n")
			.append("\r\n")
			.append("<html><body>")
			.append(content)
			.append("</body></html>");
		outputStream.write(httpResponse.toString().getBytes());
		outputStream.close();
	}
}
```

# 自定义Servlet模板类和继承类

自定义servlet模板类MyServlet

```java
package com.pywkk;

import com.pywkk.server.MyRequest;
import com.pywkk.server.MyResponse;

public abstract class MyServlet {
	public abstract void doGet(MyRequest myRequest, MyResponse myResponse);
	
	public abstract void doPost(MyRequest myRequest, MyResponse myResponse);
	
	public void service(MyRequest myRequest, MyResponse myResponse) {
		if ("GET".equalsIgnoreCase(myRequest.getMethod()))
			doGet(myRequest, myResponse);
		else if ("POST".equalsIgnoreCase(myRequest.getMethod()))
			doPost(myRequest, myResponse);
	}
}
```

MyServlet测试的继承类

DemoServlet_01.java

```java
package com.pywkk.servlets;

import java.io.IOException;

import com.pywkk.MyServlet;
import com.pywkk.server.MyRequest;
import com.pywkk.server.MyResponse;

public class DemoServlet_01 extends MyServlet {

	@Override
	public void doGet(MyRequest myRequest, MyResponse myResponse) {
		try {
			myResponse.write("DemoServlet_01 get method......");
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
	}

	@Override
	public void doPost(MyRequest myRequest, MyResponse myResponse) {
		try {
			myResponse.write("DemoServlet_01 post method......");
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
	}
	
}
```

DemoServlet_HelloWorld.java

```java
package com.pywkk.servlets;

import java.io.IOException;

import com.pywkk.MyServlet;
import com.pywkk.server.MyRequest;
import com.pywkk.server.MyResponse;

public class DemoServlet_HelloWorld extends MyServlet {

	@Override
	public void doGet(MyRequest myRequest, MyResponse myResponse) {
		try {
			myResponse.write("DemoServlet_HelloWorld get method......");
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

	@Override
	public void doPost(MyRequest myRequest, MyResponse myResponse) {
		try {
			myResponse.write("DemoServlet_HelloWorld post method......");
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

}
```

# 自定义映射类和配置类

用映射类存储servlet的映射关系，代替Tomcat中的web.xml中某个servlet的映射关系

```java
package com.pywkk.config;

public class ServletMapping {
	private String servletName;//servlet命名
	private String url;//映射的url
	private String clazz;//对应的类名
	
	public ServletMapping(String servletName, String url, String clazz) {
		this.servletName = servletName;
		this.url = url;
		this.clazz = clazz;
	}
	public String getServletName() {
		return servletName;
	}
	public void setServletName(String servletName) {
		this.servletName = servletName;
	}
	public String getUrl() {
		return url;
	}
	public void setUrl(String url) {
		this.url = url;
	}
	public String getClazz() {
		return clazz;
	}
	public void setClazz(String clazz) {
		this.clazz = clazz;
	}	
}
```

用配置类存储定义的所有MyServlet映射信息

```java
package com.pywkk.config;

import java.util.ArrayList;
import java.util.List;

public class ServletMappingConfig {
	public static List<ServletMapping> servletMappingList = new ArrayList<>();
	//预加载映射信息
	static {
		servletMappingList.add(new ServletMapping("DemoServlet_01", "/demo01", "com.pywkk.servlets.DemoServlet_01"));
		servletMappingList.add(new ServletMapping("DemoServlet_HelloWorld", "/HelloWorld", "com.pywkk.servlets.DemoServlet_HelloWorld"));
	}
}
```

# 服务器类

```java
package com.pywkk.server.mytomcat;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.HashMap;
import java.util.Map;
import com.pywkk.MyServlet;
import com.pywkk.config.ServletMapping;
import com.pywkk.config.ServletMappingConfig;
import com.pywkk.server.MyRequest;
import com.pywkk.server.MyResponse;

public class MyTomcat {
	private int port = 8080;//默认端口8080
	private Map<String, String> urlServlet = new HashMap<>();
	
	public MyTomcat(int port) {
		this.port = port;
	}
	public MyTomcat() {
	}
	public void start() {
		initServletMapping();//初始化容器，解析servlet映射信息
		ServerSocket serverSocket = null;
		try {
			serverSocket = new ServerSocket(port);
			System.out.println("MyTomcat is start......");
			while(true) {
				Socket socket = serverSocket.accept();
				InputStream inputStream = socket.getInputStream();
				OutputStream outputStream = socket.getOutputStream();
				MyRequest myRequest = new MyRequest(inputStream);
				MyResponse myResponse = new MyResponse(outputStream);
				dispatch(myRequest, myResponse);
				socket.close();
			}
		} catch (IOException e) {
			e.printStackTrace();
		} finally {
			if (serverSocket != null) {
				try {
					serverSocket.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}	
	}
	//利用反射，创建对应的MyServlet类，并调用service方法
	private void dispatch(MyRequest myRequest, MyResponse myResponse) {
		String clazz = urlServlet.get(myRequest.getUrl());
		try {
			Class<MyServlet> myServletClass = (Class<MyServlet>) Class.forName(clazz);
			MyServlet myServlet = myServletClass.newInstance();
 			myServlet.service(myRequest, myResponse);
		} catch (ClassNotFoundException | InstantiationException | IllegalAccessException e) {
			e.printStackTrace();
		}
	}
	//初始化并装载MyServlet映射信息
	private void initServletMapping() {
		for (ServletMapping servletMapping : ServletMappingConfig.servletMappingList)
			urlServlet.put(servletMapping.getUrl(), servletMapping.getClazz());
	}
	public static void main(String[] args) {
		new MyTomcat(8080).start();
	}
}
```

### 待改进......

# 参考目录：

1、[写一款 Tomcat 也没有那么难](https://mp.weixin.qq.com/s?__biz=MzIzMzgxOTQ5NA==&mid=2247485667&idx=1&sn=98a48800561d8e06c71ba3c20cf52b1c&chksm=e8fe94eadf891dfc1ca53e3487f8425002e43251777f71d43efed3bb7a7f45154897abb6ecf6&mpshare=1&scene=23&srcid=1108RGTEgqXo8eZ9csKs4qxW#rd)



