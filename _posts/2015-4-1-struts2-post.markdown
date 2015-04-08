---
layout: post
title:  "Struts2怎么使用自定义的日期类型转换器？"
date:   2015-4-1
---

<p class="intro"><span class="dropcap">其</span>实非常简单，我们只需要自定义一个DateConverter类，这个类继承StrutsTypeConverter这一个抽象类，实现抽象类的convertFromString和convertToString方法。</p>
一个简单的自定义日期类型转换器：

	import java.text.DateFormat;
	import java.text.SimpleDateFormat;
	import java.util.Date;
	import java.util.Map;

	import org.apache.struts2.util.StrutsTypeConverter;

	mport com.opensymphony.xwork2.conversion.TypeConversionException;

	public class DateConverter extends StrutsTypeConverter {
		private  final DateFormat[] dfs = {		
			new SimpleDateFormat("yyyy.MM.dd"),
			new SimpleDateFormat("yyyy-MM-dd"),
			new SimpleDateFormat("yyyy/MM/dd") }; 
		public Object convertFromString (Map context, String[] values, 
				Class toType) {
			String dateStr = values[0];// 获取日期的字符串
			for (int i=0;i<dfs.length;i++) {// 遍历日期支持格式，进行转换
				try {
					return dfs[i].parse(dateStr);
				} catch (Exception e) {				
					continue;
				}		
			}		
			throw new TypeConversionException();		
		}
		public String convertToString (Map context, Object object) {
			Date date = (Date) object;
			// 输出的格式是yyyy-MM-dd
			return new SimpleDateFormat("yyyy-MM-dd").format(date);
		}
	}

接下来我们需要注册一下转换器,可以在src目录下创建文件xwork-conversion.properties来配置日期类型转换器`java.util.Date=包名.DateConverter`。而在javabean和action页面都只需是date类型，从前台jsp页面得到的字符串型数据会自动转换为日期型。

Q: 当我们将日期插入数据库时也许会出现java.util.Date cannot be cast to java.sql.Date问题

A:我们需要进行java.util.date和java.sql.date之间的转换
	
`ps.setObject(1, new java.sql.Date(user.getDate().getTime()));`