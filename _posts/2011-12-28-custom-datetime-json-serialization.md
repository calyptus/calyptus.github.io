---
layout: post
title: Custom DateTime JSON Format for .NET JavaScriptSerializer
date: 2011-12-28 10:21:32.000000000 +01:00
categories:
  - seb
tags:
  - dotnet
  - json
---
The DateTime serialization format in the built-in JSON serializer in .NET is particularly inconvenient as it requires a regular expression to parse and is not human-readable which makes debugging a pain.

## ISO 8601
JSON doesn't have a standard Date format per say. By design. Instead, the JavaScript defines toJSON methods so that objects can convert themselves to a JSON compatible format. The ECMAScript specification 5th Edition (15.9.5.4) defines that toJSON of a Date object should return a ISO 8601 formatted string in universal time (UTC). E.g: "2011-12-28T08:20:31.917Z"

This is a convenient format that specifically avoids confusion and ambiguity when passing it between systems. To turn it back into a Date object in JavaScript we can simply pass it to the Date constructor: E.g: Date("2011-12-28T08:20:31.917Z")

## Custom Serialization Using JavaScriptConverter
.NET's JavaScriptSerializer does provide an extension point for custom serialization. By calling RegisterConverters and passing custom JavaScriptConverter instances you can control how custom classes are serialized. APIs such as Script Services allow you to register such converters in Web.config. (AFAIK ASP.NET MVC doesn't yet provide a hook to add JavaScriptConverters so you have to use <a href="http://weblogs.asp.net/rashid/archive/2009/03/23/submitting-my-first-bug-after-asp-net-mvc-1-0-rtm-release.aspx">a custom ActionResult</a>.)

Unfortunately the JavaScriptConverter API will only allow you to convert an object into another object (IDictionary&lt;string, object&gt;). Not into a custom string format.

It is at this point you might be looking for other serializers such as Json.NET. There are certainly many much faster and less crappy serializers out there. However, there is a benefit to using the built-in one. Mainly it may be already integrated into existing system that you can't change. Luckily, as long as they allow you to register custom JavaScriptConverters I have the solution for you.

## The Hack
The implementation of JavaScriptSerializer does string serialization for several types but Uri in particular. Uri is a class instead of a struct and fortunately is not sealed. That means we can inherit from it to implement the IDictionary&lt;string, object&gt; interface. That will allowing us to pass it in our custom JavaScriptConverter. Since it inherits from Uri it will be serialized as a string. It will do some URI encoding but this doesn't affect the ISO 8601 format.

Deserialization is handled by the default implementation.

You can also use this hack to serialize custom enum types as strings instead of numbers or objects.

## The Code
{% highlight c# %}
public class DateTimeJavaScriptConverter : JavaScriptConverter
{
	public override object Deserialize(IDictionary<string, object> dictionary, Type type, JavaScriptSerializer serializer)
	{
		return new JavaScriptSerializer().ConvertToType(dictionary, type);
	}
	
	public override IDictionary<string, object> Serialize(object obj, JavaScriptSerializer serializer)
	{
		if (!(obj is DateTime)) return null;
		return new CustomString(((DateTime)obj).ToUniversalTime().ToString("O"));
	}
	
	public override IEnumerable<Type> SupportedTypes
	{
	  get { return new[] { typeof(DateTime) }; }
	}
	
	private class CustomString : Uri, IDictionary<string, object>
	{
		public CustomString(string str)
		  : base(str, UriKind.Relative)
		{
		}
		
		void IDictionary<string, object>.Add(string key, object value)
		{
			throw new NotImplementedException();
		}
		
		bool IDictionary<string, object>.ContainsKey(string key)
		{
			throw new NotImplementedException();
		}
		
		ICollection<string> IDictionary<string, object>.Keys
		{
			get { throw new NotImplementedException(); }
		}
		
		bool IDictionary<string, object>.Remove(string key)
		{
			throw new NotImplementedException();
		}
		
		bool IDictionary<string, object>.TryGetValue(string key, out object value)
		{
			throw new NotImplementedException();
		}
		
		ICollection<object> IDictionary<string, object>.Values
		{
			get { throw new NotImplementedException(); }
		}
		
		object IDictionary<string, object>.this[string key]
		{
			get
			{
			  throw new NotImplementedException();
			}
			set
			{
				throw new NotImplementedException();
			}
		}
		
		void ICollection<KeyValuePair<string, object>>.Add(KeyValuePair<string, object> item)
		{
		  throw new NotImplementedException();
		}
		
		void ICollection<KeyValuePair<string, object>>.Clear()
		{
		  throw new NotImplementedException();
		}
		
		bool ICollection<KeyValuePair<string, object>>.Contains(KeyValuePair<string, object> item)
		{
		  throw new NotImplementedException();
		}
		
		void ICollection<KeyValuePair<string, object>>.CopyTo(KeyValuePair<string, object>[] array, int arrayIndex)
		{
		  throw new NotImplementedException();
		}
		
		int ICollection<KeyValuePair<string, object>>.Count
		{
		  get { throw new NotImplementedException(); }
		}
		
		bool ICollection<KeyValuePair<string, object>>.IsReadOnly
		{
		  get { throw new NotImplementedException(); }
		}
		
		bool ICollection<KeyValuePair<string, object>>.Remove(KeyValuePair<string, object> item)
		{
		  throw new NotImplementedException();
		}
		
		IEnumerator<KeyValuePair<string, object>> IEnumerable<KeyValuePair<string, object>>.GetEnumerator()
		{
		  throw new NotImplementedException();
		}
		
		IEnumerator IEnumerable.GetEnumerator()
		{
		  throw new NotImplementedException();
		}
	}
}
{% endhighlight %}
