---
title: ubuntu+java+imgbb图床脚本
categories: 学习笔记
tags: [图床, WeiboPicBed, Java]
abbrlink: 58bb5a76
date: 2019-04-14 03:20:11
---



# Markdown图床

## 图床需求

接触markedown三年左右，不经常写。刚开始用过一段“小书匠”写[博客园](http://www.cnblogs.com/oucbl/)，主要是语法兼容和图床的问题，还不能跨平台。

之后几乎没用过，最近觉得要建自己的博客，而且也应过经常使用markedown写博客，而（稳定的）图床是不得不面临的。



**对本人来说，上传图片的方式最频繁的、最快捷的需求就是直接复制粘贴图片**



## 图床选择

基于国内网络环境，本文推荐两个稳定快速的图床：新浪微博、Imgbb。



一些教程是说到**Typora+ipic+七牛云**的方式；

不过iPic插件仅支持mac系统，而且我的七牛也没审核通过，所以也没有去倒腾。

 

# 新浪微博图床插件

这个得益于github大神的开源项目**WeiboPicBed（新浪微博图床 Chrome扩展）**

**使用方法：**Chrome浏览器 + 插件

**注意事项：**需要提前在chrome里面登录新浪微博

**插件特点：**

- 支持**点选/拖拽/本地粘贴**3种方式上传图片至新浪微博图床
- 支持批量上传
- 可选择返回的图片尺寸（三种尺寸大小）
- 可生成图片链接,HTML,UBB和Markdown四种格式
- 上传历史浏览和删除.
- 支持返回https安全协议的图片地址

插件的截图如下：

![](https://ws1.sinaimg.cn/large/9d89d613ly1g26zo81c6cj20lx0d9mya.jpg)



# Imgbb(ibb)图床使用

## imgbb网站特点

- 支持中文语言，国内网络可直接访问（美国服务器）
- 支持任意**拖放/粘贴**图片到网页里面即可上传图片（最大 16 MB，jpg,png,bmp,gif）
- 支持匿名（无登录）上传
- 支持API上传

## Java脚本上传

### 功能

从剪切板中直接复制上传缓存图片到imgbb服务器，返回文本（可粘贴）

### 原理

简单叙述一下流程：

1. 从剪切板中读取图片缓存
2. 将图片缓存转base64编码
3. API上传图片并返回状态
4. 将链接文本复制到剪切板


### 源码

个人说明：

1. 对java有些机制不熟悉，很简单的几个功能硬是各种错误，各种尝试，花了一个晚上才完成。如果有什么问题或者更好的方法欢迎直接批评建议。
2. 本程序是Ubuntu系统，复制文字的方法直接（Ctrl+V）粘贴并不起作用，找了好久的教程发现需要保持个系统进程，其他操作系统并未测试。
3. Java的http访问也是很不习惯（不熟悉），尤其是post参数，比如字符串类型，json类型，表单类型等的请求很混乱。Header还容易出问题。


```java

/**
 * @author oucbl
 * @date 2019.04.13
 */

import java.awt.Toolkit;
import java.awt.datatransfer.Clipboard;
import java.awt.datatransfer.DataFlavor;
import java.awt.datatransfer.StringSelection;
import java.awt.datatransfer.Transferable;
import java.awt.image.BufferedImage;
import java.awt.image.RenderedImage;
import java.io.BufferedReader;
import java.io.ByteArrayOutputStream;
import java.io.InputStreamReader;
import java.io.IOException;
import java.io.UncheckedIOException;
import java.net.URL;
import java.net.URLEncoder;
import javax.net.ssl.HttpsURLConnection;

import java.util.Base64;
import javax.imageio.ImageIO;

import org.json.simple.JSONObject;
import org.json.simple.parser.JSONParser;

public class ClipboardToBase64
{
	/**
	 * 将字符串复制到剪切板（复制）
	 * 参考：http://landcareweb.com/questions/18364/fu-zhi-dao-quan-ju-jian-tie-ban-bu-gua-yong-yu-ubuntuzhong-de-java
	 * @throws IOException
	 */
	public static void setSysClipboardText(String text) throws IOException
	{
		Clipboard clip = Toolkit.getDefaultToolkit().getSystemClipboard();

		Transferable tText = new StringSelection(text);
		clip.setContents(tText, null);
		// 上两行可有替换成
		// StringSelection tText = new StringSelection(text);
		// clip.setContents(tText, tText);

		System.out.println("字符串复制成功。");
		System.in.read(); // 只是让进程保持活动状态

	}
	/**
	 * 从剪切板获得图片缓存
	 */
	public static BufferedImage getImageFromClipboard() throws Exception
	{
		Clipboard sysc = Toolkit.getDefaultToolkit().getSystemClipboard();
		Transferable cc = sysc.getContents(null);
		if (cc == null)
			return null;
		else if (cc.isDataFlavorSupported(DataFlavor.imageFlavor))
		{
			return (BufferedImage) cc.getTransferData(DataFlavor.imageFlavor);
		}
		return null;
	}
	/**
	 * 将图片缓存转为base64编码
	 */
	public static String imgToBase64String(final RenderedImage img,
			final String formatName)
	{
		final ByteArrayOutputStream os = new ByteArrayOutputStream();
		try
		{
			ImageIO.write(img, formatName, os);
			return Base64.getEncoder().encodeToString(os.toByteArray());
		} catch (final IOException ioe)
		{
			throw new UncheckedIOException(ioe);
		}
	}

	/**
	 * 参考：https://www.baeldung.com/httpurlconnection-post 通过api上传图片
	 * 
	 * @param imgbbKey
	 *            imgbb上传API的key
	 * @param imgb64str
	 *            图片的base64编码
	 * @param imgname
	 *            待上传图片的命名
	 * @return 并返回json响应文本
	 * @throws Exception
	 */
	public static String uploadImgbb(String imgbbKey, String imgb64str,
			String imgname) throws Exception
	{
		String httpsURL = "https://api.imgbb.com/1/upload";
		URL myUrl = new URL(httpsURL);
		HttpsURLConnection con = (HttpsURLConnection) myUrl.openConnection();
		con.setRequestMethod("POST");
		con.setRequestProperty("User-Agent", "Mozilla/5.0");
		con.setDoOutput(true);

		StringBuffer params = new StringBuffer();
		// 表单参数与get形式一样
		params.append("key").append("=").append(imgbbKey).append("&")
				.append("image").append("=")
				.append(URLEncoder.encode(imgb64str)).append("&").append("name")
				.append("=").append(URLEncoder.encode(imgname));
		byte[] bypes = params.toString().getBytes();
		con.getOutputStream().write(bypes);// 输入参数

		BufferedReader in = new BufferedReader(
				new InputStreamReader(con.getInputStream()));
		String inputLine;
		StringBuffer response = new StringBuffer();

		while ((inputLine = in.readLine()) != null)
		{
			response.append(inputLine);
		}
		in.close();

		String responseString = response.toString();
		System.out.println(responseString);
		return responseString;
	}
	/**
	 * 格式化json文本，并返回所需文本
	 */
	public static String getMdtext(String imgbbres)
	{
		String mdtext = null;
		try
		{
			JSONParser parser = new JSONParser();
			Object resultObject = parser.parse(imgbbres);

			if (resultObject instanceof JSONObject)
			{
				JSONObject obj = (JSONObject) resultObject;
				if (obj.get("status").toString().contains("200"))
				{
					JSONObject data = (JSONObject) obj.get("data");
					mdtext = "![ ](" + data.get("url").toString() + ")";
				}
			}

		} catch (Exception e)
		{
			// TODO: handle exception
		}
		return mdtext;
	}
	public static void main(String[] args) throws Exception
	{
		String imgbbKey = "f8418f659962dec949f7271a6287a167"; // 可修改为自己的key

		BufferedImage image = getImageFromClipboard();
		if (image == null)
		{
			System.out.println("clipboard image is null");
		} else
		{
			String b64str = imgToBase64String(image, "png");
			System.out.println(b64str.length());
			String imgbbres = uploadImgbb(imgbbKey, b64str, "imgname");
			String mdtext = getMdtext(imgbbres);

			System.out.println(mdtext);
			setSysClipboardText(mdtext);
		}
	}
}
```


### 期待

1. 本文java程序只是一个核心上传的控制台程序，如果可以**提供GUI的接口并打成jar包**，甚至可以有预览、编辑等简单实用功能，并不见得不是一个高效的可替代方案。
2. Flameshot v0.6.0 是Ubuntu平台很强大的一个C++开源截图软件，自带**上传Imgur**功能，不过看不到具体的配置，而且网络也不通，如果能把这个功能替换或扩展到别的多个**可用平台**，就方便多了。




## 相关工具

新浪微博图床 Chrome扩展（github）

https://github.com/Suxiaogang/WeiboPicBed

Imgbb官网

https://imgbb.com/



------

### 参考

题 复制到全局剪贴板不适用于Ubuntu中的Java

http://landcareweb.com/questions/18364/fu-zhi-dao-quan-ju-jian-tie-ban-bu-gua-yong-yu-ubuntuzhong-de-java