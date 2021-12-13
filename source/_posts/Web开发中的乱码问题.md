---
title: Web开发中的乱码问题
toc: true
---

###### 主要内容
```
1. 字符编码理论简述
    1.1 ASCII
    1.2 ISO8859-1
    1.3 Unicode
    1.4 GBK
    
2. 可能发生的中文乱码
    2.1 中文变问号，如：???
    2.2 中文变奇怪字符，如：ä½ å¥½ 或者 ÄãºÃ
    2.3 中文变“复杂中文”，如：浣犲ソ
    2.4 中文变成一堆黑色菱形+问号，如：�����

3. Web开发中涉及到的中文编解码
    3.1 URL中出现的中文
    3.2 Form表单中出现的中文
    3.3 JSP中涉及的编码
    3.4 文件的上传和下载中涉及到的中文乱码
4. 总结
```
## 1. 字符编码理论简述
本文主要是围绕Web开发中涉及到的中文编码这一常见问题展开，包括了对字符编码基础理论的简述以及常见几种编码标准的介绍。其中包括：ASCII、ISO8859-1、Unicode、GBK。下面先对这些字符编码集进行简单的介绍。

#### 1.1 ASCII
ASCII也就是美国信息交换标准码，采用单字节编码方案，但是编码只用了后七位字节，表示范围0-127共128个字符。ASCII码相对于其它编码也是最早出现的。从上世纪60年代提出开始，到1986年最终定型。

> 为什么选择7位编码?ASCII在最初设计的时候需要至少能表示64个码元：包括26个字母+10个数字+图形标示+控制字符，如果用6bit编码，可扩展部分没有了，所以至少需要7bit。那么8bit呢？最终也被标准委员会否定，原因很简单：满足编码需求的前提下，最小化传输开销。

####  1.2 ISO8859-1
ISO-8859-1也被称为Latin1，使用单字节8bit编码，可以表示256个西欧字符。其隶属于ISO8859标准的一部分，还有ISO8859-2、ISO8859-3等等。每一种编码都对应一个地区的字符集。比如：ISO8859-1表示西欧字符，ISO-8859-16表示中欧字符集，等等。

####  1.3 Unicode
不管是ASCII还是ISO8859-1，其编码范围都是有局限的。而Unicode标准的目标就是消除传统编码的**局限性**。

> 这里的局限性一方面指编码范围的局限性：比如ASCII只能表示128个字符。还有编码兼容性方面的局限性：比如ISO8859代表的一系列编码字符集虽然可以表示大部分国家地区的字符，但是彼此的兼容性做的不好。Unicode的目标就如同其名称的含义一样：“实现字符编码统一”

> Unicode标准的实现方案有如下三种：**UTF-8**、**UTF-16和**UTF-32**.

**UTF-8**是变长编码，使用1到4个字节。UTF-8在设计时考虑到向前兼容，所以其前128个字符和ASCII完全一样，也就是说，所有ASCII同时也都符合UTF-8编码格式。其格式如下：
```
0xxxxxxx
110xxxxx 	10xxxxxx
1110xxxx 	10xxxxxx 	10xxxxxx
11110xxx 	10xxxxxx 	10xxxxxx 	10xxxxxx
```
字节首部为0的话，也就是前面说的ASCII了。此外，字节**首部连续1的个数**就代表了该字符编码后所占的字节数。目前全世界的网页编码绝大多数使用的就是UTF-8，占比接近90%。

**UTF-16**也是变长编码，但其最初是**固定16-bit宽度的定长编码**，主要因为Unicode涵盖的字符太多了。两字节更本不够用！

**UTF-32**是32-bit定长编码，优点：定长编码在处理效率上相对于变长编码要高，此外，可通过**索引**访问任意字符是其另一大优势；缺点也很明显：32bit太浪费了！存储效率太低！

> big-endian和little-endian？在多字节编码标准中可能会遇到这样的问题：假如一个字符用两个字节表示，那么当读取这个字符的时候，哪个字节表示高有效位？哪个表示低有效位呢？这就涉及到字节的存储顺序问题。在Unicode中UTF-16和UTF-32都会面临这个问题。通常用BOM（Byte Order Mark）来进行区分。BOM用一个"U+FEFF"来表示，这个值在
Unicode中是没有对应字符的。不仅可以用其来指定字节顺序，还可以表示字节流的编码方式。

```
System.out.println("len1:" + "a".getBytes("UTF16").length);
System.out.println("len2:" + "aa".getBytes("UTF16").length);
```
输出结果：
> len1:4

> len2:6

为什么是4和6，**不应该是2和4吗！？**。输出编码后的字节序列可以发现，起始的两个字节都是："fe ff"。


> Java的char类型用什么编码格式？Java语言规范规定了Java的char类型使用的是UTF-16。这就是为什么Java的char占用两个字节的原因。此外，Java标准库实现的对char与String的序列化规定使用UTF-8。Java的Class文件中的字符串常量与符号名字也都规定用UTF-8编码。这大概是当时设计者为了平衡运行时的时间效率（采用定长编码的UTF-16，当然，在设计java的时候UTF-16还是定长的）与外部存储的空间效率（采用变长的UTF-8编码）而做的取舍。

####  1.4 GBK
GBK是用于对简体中文进行编码。每个字符用两字节表示，同时兼容GB2312标准。

## 2. 可能发生的中文乱码
这一小节介绍软件开发中常见的中文编码乱码问题，在下面示例中：对于给定的一个包含中文的字符串"你好Java"，看一下都会出现哪些乱码问题。

#### 2.1 中文变问号，如：?????
```
"你好Java"  ------>  "??Java"
```
这种情况一般是由于**中文字符经ISO8859-1编码造成的。**下面是编码的具体过程：

原字符串："你好Java"

你 | 好 | J| a| v| a
---|---|---|---|---|---
4f60 | 597d | 4a| 61| 76| 61

经ISO8859-1编码后：

你 | 好 | J| a| v| a
---|---|---|---|---|---
3f | 3f | 4a| 61| 76| 61

编码后字符串："??Java"
```
String str = "你好Java";
System.out.println(byteToHexString(str.getBytes(CHARSET_ISO88591)));
System.out.println(new String(str.getBytes(CHARSET_ISO88591)));
输出：
3f 3f 4a 61 76 61
??Java
```
我们知道ISO8859-1是单字节编码，而对于汉字已经超出ISO8859-1的编码范围，会被转化为"3f"，我们查表可知，"3f"对应的字符正是"?"。

> 中文变问号的乱码情况是非常常见的，大部分开源软件的默认编码设置成了ISO8859-1，这点需要格外注意。

#### 2.2 中文变奇怪字符，如：ä½ å¥½ 或者 ÄãºÃ
```
"你好Java"  ------>  "ä½ å¥½Java"
```
原字符串："你好Java"

你 | 好 | J| a| v| a
---|---|---|---|---|---
4f60 | 597d | 4a| 61| 76| 61

经UTF-8编码后，一个中文用三个字节表示：

你 | 好 | J| a| v| a
---|---|---|---|---|---|---|---
e4 bd a0 | e5 a5 bd | 4a| 61| 76| 61

> 乱码原因：UTF8编码或GBK编码，再由ISO8859-1解码。对照ISO8859-1编码表后发现：e4 bd a0分别对应三个字符："ä½ ",e5 a5 bd分别对应三个字符"å¥½",

#### 2.3 中文变“复杂中文”如：浣犲ソ
下面依然是"你好Java"经过UTF-8编码后对应的字节序列：

你 | 好 | J| a| v| a
---|---|---|---|---|---|---|---
e4 bd a0 | e5 a5 bd | 4a| 61| 76| 61

在GBK表中查找：e4 bd对应字符："浣",a0 e5对应字符："犲",a5 bd对应字符："ソ"

> 同理，如果GBK编码的中文用UTF-8来解码的话，同样会出现乱码问题。

#### 2.4 中文变成一堆黑色菱形+问号，如：�����
首先问号+黑色菱形的字符是Unicode中的"REPLACEMENT CHARACTER",该字符的主要作用是用来表示不识别的字符。
所以产生乱码的原因可能有很多，下面通过原字符串："你好Java"，重现一种乱码方式：
```
原字符串：String str = "你好Java"

你 | 好 | J| a| v| a
---|---|---|---|---|---
4f60 | 597d | 4a| 61| 76| 61

UTF-16编码后

fe ff 4f 60 59 7d 0 4a 0 61 0 76 0 61
```
其中"fe ff"就是字节流起始的BOM标识符。"fe ff"在Unicode标准中属于"noncharacters",只用于内部使用。所以，
在输出该字节序列的时候，没有该码元对应的字符，对于不识别字符，就会用��替代。

## 3. Web开发中涉及到的中文编解码
Web中的数据大多通过http协议进行传输，所涉及到的一些编解码问题都围绕着http协议。下面以Tomcat作为Web服务器，
探讨下一个完整的请求响应流程中哪些地方会涉及到中文的编解码。
#### 3.1 url编解码
web环境中的中文乱码问题，实验如下：
```
jsp中的form表单：
<body>
	<form name="form" method="post" action="manager/codec/你好">
		<table>
			<tr>
				<td>用户名： <input type="text" name="name" id="name" />
				</td>
				<td>地址 <input type="text" name="address" id="address" />
				</td>
				<th><input type="submit" name="submit" value="保存" /></th>
			</tr>
		</table>
	</form>
</body>

后端使用SpringMVC的Controller：

@Controller()
@RequestMapping("/manager")
public class ManagerController {

    @RequestMapping("/test/{param}")
    @ResponseBody
    public String test(@PathVariable String param, HttpServletRequest request){
        String name = request.getParameter("name");
        System.out.println("name:" + name + ",param:" + param);
        return "test";
    }
}
```
表单中填入内容：
用户名：你好 Java
地址：123
提交请求，firebug中的显示的url如下：

> http://localhost:8080/fdyuntu-ssm/manager/codec/%E4%BD%A0%E5%A5%BD

查阅编码可以，firefox对url中出现的中文使用了UTF-8的编码方式。之所以url中出现%，这是因为根据URL编码规范，浏览器会将非ASCII字符编成16进制后，每个字节前需要加%。

后端控制台输出：
```
name:ä½ å¥½ Java,param:ä½ å¥½
```

可见无论是url中的中文信息或是post表单中的中文都出现了乱码现象，从前一节中关于乱码情况的分析来看，这里应该是**中文字符经过浏览器UTF-8编码后，Server端用ISO8859-1进行解码所致。**下面逐个分析url和post表单如何进行编解码的。

在tomcat中url的byte -> char的转换是在org.apache.catalina.connector.CoyoteAdapter类的convertURI(MessageBytes uri, Request request)方法中执行的，源码如下：

```
    protected void convertURI(MessageBytes uri, Request request)throws Exception {

        ByteChunk bc = uri.getByteChunk();
        int length = bc.getLength();
        CharChunk cc = uri.getCharChunk();
        cc.allocate(length, -1);
    
//这里获取的connector的URIEncoding属性，即server.xml文件中connector元素的URIEncoding属性
        String enc = connector.getURIEncoding();
        if (enc != null) {
            B2CConverter conv = request.getURIConverter();
            try {
                if (conv == null) {
                    conv = new B2CConverter(enc, true);
                    request.setURIConverter(conv);
                } else {
                    conv.recycle();
                }
            } catch (IOException e) {
                log.error("Invalid URI encoding; using HTTP default");
                connector.setURIEncoding(null);
            }
            if (conv != null) {
                try {
                    conv.convert(bc, cc, true);
                    uri.setChars(cc.getBuffer(), cc.getStart(), cc.getLength());
                    return;
                } catch (IOException ioe) {
                    request.getResponse().sendError(
                            HttpServletResponse.SC_BAD_REQUEST);
                }
            }
        }

        // 如果没有配置URIEncoding，则在ByteChunk中默认使用ISO8859-1。
        byte[] bbuf = bc.getBuffer();
        char[] cbuf = cc.getBuffer();
        int start = bc.getStart();
        for (int i = 0; i < length; i++) {
            cbuf[i] = (char) (bbuf[i + start] & 0xff);
        }
        uri.setChars(cbuf, 0, length);
    }
```
在org.apache.tomcat.util.buf.ByteChunk中可以看到默认编码的定义：
```
public final class ByteChunk implements Cloneable, Serializable {

    //。。。
    
    public static final Charset DEFAULT_CHARSET = B2CConverter.ISO_8859_1;
    
    //。。。
}
```
所以对于请求url中的中文，我们按UTF-8进行编码，在服务端却按ISO8859-1进行解码，所以出现乱码现象。我们可以再Tomcat的server.xml中指定url的编解码格式，如下：
```
<Connector  URIEncoding="UTF-8" 。。。>

```
> 此时重复上面实验，后端控制台输出：name:ä½ å¥½ Java,param:你好

虽然url中的参数可以正常显示了，但是form表单中的参数name依然乱码，下面进一步分析。
### 3.2 form表单元素的编解码
name参数的编码依然是乱码的，为啥？首先**定位form表单中参数是在哪里进行解码的。**Form表单中的字符解码时机是发生在第一次调用request.getParameter时，可以通过request.setCharacterEncoding设置。**需要注意的是setCharacterEncoding必须在getParameter之前调用！否则，setCharacterEncoding不会起作用。**

Tomcat中HttpServletRequest接口的实现类是**org.apache.catalina.connector.Request**。下面是Request类中getParameter源码：
```
    @Override
    public String getParameter(String name) {
        //判断参数是否被解析过
        if (!parametersParsed) {
            parseParameters();//第一次参数解析
        }
        
        return coyoteRequest.getParameters().getParameter(name);
    }

//下面是parseParameters部分源码

   protected void parseParameters() {
        
        //设为true，表示参数已解析过
        parametersParsed = true;
        //Parameters对象封装了form表单参数
        Parameters parameters = coyoteRequest.getParameters();
        
        boolean success = false;
        try {
            // Set this every time in case limit has been changed via JMX
            parameters.setLimit(getConnector().getMaxParameterCount());
        
            //获取字符编码格式
            String enc = getCharacterEncoding();

            boolean useBodyEncodingForURI = connector.getUseBodyEncodingForURI();
            if (enc != null) {
            //getCharacterEncoding不为null，则对应设置编码方式
                parameters.setEncoding(enc);
                if (useBodyEncodingForURI) {
                    parameters.setQueryStringEncoding(enc);
                }
            } else {
                //如果enc为null，则编码方式设置为DEFAULT_CHARACTER_ENCODING，也就是ISO8859-1
                parameters.setEncoding
                    (org.apache.coyote.Constants.DEFAULT_CHARACTER_ENCODING);
                if (useBodyEncodingForURI) {
                    parameters.setQueryStringEncoding
                    (org.apache.coyote.Constants.DEFAULT_CHARACTER_ENCODING);
                }
            }

            parameters.handleQueryParameters();
            
            。。。
        }
    }
```
从以上源码中可以看出为什么需要在第一次调用getParameter之前设置CharacterEncoding。因为第一次执行parseParameters时，会把parametersParsed变量设为true。所以parseParameters只会在第一次getParameter时调用。有时会出现这么一种怪像：通过request.getCharacterEncoding()得到的是我们认为正确的编码字符集，但是request.getParameter得到的依然是乱码。此时就需要考虑下我们调用setCharacterEncoding之前是否已经调用过getParameter方法了。

> 经过上面的分析后，对于form表单参数乱码问题就很好解决了，在第一次调用request.getParameter方法前，通过request.setCharacterEncoding("Expected_Encoding");设置即可。这一步可以用Servlet标准中的Filter实现，不过，常用的MVC框架中已经有现成的Filter实现了，比如SpringMVC中的org.springframework.web.filter.CharacterEncodingFilter,如下：

```
	@Override
	protected void doFilterInternal(
			HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		if (this.encoding != null && (this.forceEncoding || request.getCharacterEncoding() == null)) {
			request.setCharacterEncoding(this.encoding);//设置指定的编码
			if (this.forceEncoding) {
				response.setCharacterEncoding(this.encoding);
			}
		}
		filterChain.doFilter(request, response);
	}
```
### 3.3 JSP中涉及的编码
jsp中可以通过page指令指定一些编码参数，如下：
```
<%@ page language="java" contentType="text/html; charset=UTF-8"
	pageEncoding="UTF-8"%>
```
###### pageEncoding="UTF-8"在什么时候起作用？
在Servlet标准中，jsp最终也会被编译成一个servlet。**index.jsp->index_jsp.java**.pageEncoding="UTF-8"就是在这个解析过程中起作用的。

###### contentType="text/html; charset=UTF-8"的作用？
contentType是响应头中特定信息，主要的作用是告诉浏览器response中存放的主体对象类型和编码，这样浏览器就可以对指定类型进行正确解码，保证了数据在server和client端的一致性。当进行Servlet编程的时候，可以手动进行设置，如下：
```
response.setContentType("text/html; charset=UTF-8");
```
### 3.4 文件的上传和下载中涉及到的中文乱码
Web中的文件操作主要是上传和下载，这个过程也是依托于Http协议作为数据载体。所以，最终是否乱码重点在于是否正确的设置http的request、response的header中的相关字段。如ContentType、Content-Disposition的设定等。如下：
```
response.setHeader("Pragma", "no-cache");
response.setHeader("Cache-Control", "no-cache");
response.setDateHeader("Expires", 0);
response.setContentType("application/x-msdownload");
response.addHeader("Content-Disposition", "attachment; filename=\"" + fileName + "\"");
```
这里需要注意的是**Content-Disposition**的filename属性值，如果fileName含有中文，那么要格外注意fileName字符串的编码格式。在[rfc5987](https://tools.ietf.org/html/rfc5987)对于HTTP的Header中参数的编码做出了明确的规定：
> By default, message header field parameters in Hypertext Transfer  Protocol (HTTP) messages cannot carry characters outside the ISO-8859-1 character set.

也就是说默认情况下，Http的Header中的参数只能用ISO-8859-1字符集中的字符，那么是否意味着Content-Disposition中的fileName字符串也要转成ISO-8859-1了呢？答案是：NO！原因如下：Content-Disposition其实不属于Http/1.1标准。这在[RFC2616](https://tools.ietf.org/html/rfc2616#section-15.5)中有明确的说明。只因为其使用广泛，HTTP才对其支持。在[rfc6266](https://tools.ietf.org/html/rfc6266#section-4.3)中也详细介绍了Content-Disposition的filename参数含义和用法。下面是对于下载包含中文名称的文件时的解决方案。

###### 解决方案
最简单就是直接用ISO8859-1对文件名进行编码，大多数浏览器都支持。如下：
```
exportFileName.getBytes("UTF-8"),"ISO8859-1");//这里的UTF-8也可能是别的编码，主要依据系统默认的编码来设定。
```
或通过其它编码，如UTF-8。
```
response.addHeader("Content-Disposition",
                "attachment; filename*=UTF-8''" + URLEncoder.encode(exportFileName, "UTF8"));
```

## 4. 总结
编解码问题是多语言交互系统中必然要面对的问题，尤其对于中文环境中的开发者来说，在入门阶段或多或少都会遇到此类问题。乱码问题本质就是通信双方使用的标准不一致。所以，解决乱码问题的方法其实也很简单，统一下编解码标准即可。此外，深入理解各种编码标准的原理和关系也非常重要，在以后遇到类似问题的时候能够更加准确的判断出造成乱码的原因。