---
layout: post
title: Large Object Storage for NHibernate - Part 1 - BLOBs, CLOBs and XLOBs
date: 2009-03-12 02:25:38.000000000 +01:00
categories:
  - seb
tags:
  - blob
  - clob
  - xlob
  - domain-driven-design
  - nhibernate
  - streaming
---
####This is the first in a series of posts describing the design considerations involved with storing Binary Large OBject (BLOB) data with NHibernate and how it led me to start a project I'm currently calling <a href="http://blog.calyptus.eu/source-code/">NHibernate.Lob</a>.

*Note that the samples here are focused mainly on NHibernate but the pattern can be applied to many different persistence models. I'm considering support for DB4o for example.*

##Lazy Streaming of Data
The typical way to store binary data in NHibernate entities would be as a byte[] array. After all the basic premise of NHibernate entities is that the data is stored in-memory in the first level cache. For smaller binary data this is just fine. We don't even really our columns to be lazy.

If we start adding larger data the first problem one might notice is that <em>the data is loaded every time the entity is loaded</em>. This is a common question around NHibernate user groups. This can quite easily be solved by separating it out to a lazy loaded entity or using lazy columns.

If we add even larger files we start wasting precious memory. This is especially problematic in high concurrency applications and web applications. A (very) common scenario would be to store image data together with an entity. At this point we shouldn't ever keep the entire file in memory. Instead we should stream the data piece by piece from the persistent storage to whatever we want to use it for.

At this point, the actual data is never stored in the in-memory entity. Only pointer data is stored about where to find the information. It goes beyond the concept of lazy loading since only a piece of the data in available in memory at any point.

*Note that this is NOT really related to the concept of a document database. We're talking about large serialized objects (500 kb+ if I had to give a number) such as images, videos or large document files.*

*I also mention the term: pointers. In the context of this article series I don't mean memory pointers but rather a reference to where one can find the real complete data. This may be in-memory, on disk, remote or distributed etc.*

##Streaming Data Types - The Current State of ADO.NET
So what data type will we use as the pointer to this data? Our domain model is suppose to be persistence ignorant so one of the common .NET types would be nice. There are typically three common structured types of large data stored in modern databases: Raw binary data, Text and XML. Binary Large OBjects are typically called BLOBs. Text or Character Large OBjects are sometimes called CLOBs. How you store the data and in which column types is very RDMS provider specific. From now on I will call these three types as just LOBs.

Now, the in-memory types for these would typically be byte[], string and XmlDocument. The streamed versions would be Stream, TextReader and XmlReader. However, this gives us some problems. The contract of these three abstract classes are more than just pointers to where to get the data. They also contain the current reading position of the stream. This means that we can only read from that entity ONCE during it's life time. They also implement IDisposable and keep a data reading connection open and expect to be closed and disposed of.

There's really no common way for working with streamed data in ADO.NET since everything. In fact, the typical example for dealing with LOBs in ADO.NET involves reading the full data into memory using IDataReader.GetBytes(...). Some providers have supplied there own solutions to this issue (such as <a href="http://msdn.microsoft.com/en-us/library/system.data.idatarecord.getbytes.aspx">OracleLob</a> and <a href="http://msdn.microsoft.com/en-us/library/system.data.idatarecord.getbytes.aspx">SqlBytes</a>). The most common solution seems to be to inherit Stream in their custom solutions. You can still read it several times by first cloning the Lob object but it isn't really a nice solution for a domain model. They also imply that the connection is already open. What we really need is a type from which we can create readers.

Thankfully our friends on the Java end of things have already thought about this. In Java there are <a href="http://java.sun.com/j2se/1.4.2/docs/api/java/sql/Blob.html">Blob</a> and <a href="http://java.sun.com/j2se/1.4.2/docs/api/java/sql/Clob.html">Clob</a> interfaces which fits just this purpose. They can create both reader and writer streams. It is also nicely implemented in both JDBC and Hibernate.

*Another issue with TextReader and XmlReader is that we have no way to write to them but this is not really an issue as I will describe at the end of this article.*

##Introducing New Data Types - Blob, Clob and Xlob
So to remedy this situation I've suggested that three new base classes are added to our .NET domain models. The contracts of these are pretty simple.

{% highlight c# %}
namespace Calyptus.Lob
{
	public abstract class Blob
	{
		public abstract Stream OpenReader();
		public virtual void WriteTo(Stream output);
	}

	public abstract class Clob
	{
		public abstract TextReader OpenReader();
		public virtual void WriteTo(TextWriter writer);
		public virtual void WriteTo(Stream output, Encoding encoding);
	}

	public abstract class Xlob
	{
		public abstract XmlReader OpenReader();
		public virtual void WriteTo(TextWriter writer);
		public virtual void WriteTo(XmlWriter writer);
		public virtual void WriteTo(Stream output, Encoding encoding);
	}
}
{% endhighlight %}

Basically there's a PULL and a PUSH method to get the data from LOB. The WriteTo methods are NOT away to write data to the LOBs. It's a way to PUSH the data from the LOB into a writer.

Why an abstract base class instead of an interface? This is a common debate in .NET. But since this pattern is overwhelmingly used most often in the .NET Framework (Stream, TextReader and XmlReader are a few examples) I figured it'd be best to keep that trend. It also allows for virtual methods to be added later (such as Java's getBytes, position and length) without recompilation of inheritors.

I'm sure that these contracts are going to be very much debated since it involves the core of the domain model which, in the NHibernate world, should be persistence ignorant. You can still easily switch out the ORM and let that ORM handle these new types. The best would be if Microsoft's Patterns and Practises team introduced these new types as a common practise and perhaps even into a System.Data.Lobs namespace.

*Some of you may be thinking that this pattern makes the domain model aware of it's repository. But it really doesn't. No more than lazy loaded entities and collection does. You can even save it to another repository. More on that later.*

##Writing to Blobs - Don't
You may have noticed that unlike the Java interface I didn't put any way to write to the LOBs in the base contract. This is because you shouldn't persist anything until a Flush (or SaveChanges) style event. If it's a new entity, the row doesn't exists and there may not be anything to write the data to. It could also not even be part of the row. It may be stored as it's own "entity" and shared by multiple other entities. In this case you would override their data. Data should be written all together in an atomic manner.

So how do I change the data? You replace the LOB pointer (the Blob, Clob or Xlob objects) with something that points to some other data source with the new data. This can be from a file, a stream, memory, or maybe a custom implementation which combines or converts data on-the-fly. This will allow you to build pipelining patterns. It can even be an other LOB in your database. NHibernate will tell your LOB object when and where to write itself to.

This is also the same way Hibernate handles Blob and Clob in Java. It actually throws exceptions if you try to write to it's Blobs or Clobs.

##Finally Some Code

Let's start by defining a domain model. Let's just stick to one single entity called Product. With a binary image file, a long description text and an XML file which contains further specifications.

{% highlight c# %}
public class Product
{
	public int ID { get; set; }
	public string Title { get; set; }
	public Blob Image { get; set; }
	public Clob Description { get; set; }
	public Xlob Specifications { get; set; }
}
{% endhighlight %}

To read from these three LOBs you would use either the PULL or PUSH patterns (OpenReader or WriteTo). The following sample fetches a Product from the database. It then writes the image data to a HttpResponse. Then it writes the specifications to disk using a custom XmlWriter. Finally it reads the first line of the description.
{% highlight c# %}
using (ISession session = sessionFactory.OpenSession())
{
	Product product = session.Get<Product>(100);

	Response.Clear();
	Response.ContentType = "image/jpeg";
	Response.BufferOutput = false;
	product.Image.WriteTo(Response.OutputStream);

	using (XmlWriter writer = XmlWriter.Create(@"C:\MyFiles\SomeData.xml"))
	{
		product.Specifications.WriteTo(writer);
	}

	using (TextReader reader = product.Description.OpenReader())
	{
		string firstLine = reader.ReadLine();
	}
}
{% endhighlight %}

Changing the LOB data involves replacing the instance with another one. You can do this by using one of the built-in implementations using the static overloaded Blob.Create(), Clob.Create() and Xlob.Create() methods. The data can come from files, streams, memory, the web or your own implementations. You could for example create your own implementation which combines two files into one on the fly as it is written to the database.

The following sample loads a product from the database, replaces the image with a file from disk, replaces the description with an in-memory string, replaces the specifications with one from the web and then saves it all to the database.

{% highlight c# %}
using (ISession session = sessionFactory.OpenSession())
using (ITransaction transaction = session.BeginTransaction())
{
	Product product = session.Get<Product>(100);
	product.Image = Blob.Create(@"C:\MyFolder\MyImage.jpg");
	product.Description = Clob.Create("My short description.");
	product.Specifications = Xlob.Create(new Uri("http://domain/document.xml"));
	transaction.Commit();
}
{% endhighlight %}

Note that in the above sample it's not a reference to the file and the web that is stored in the database. The actual data is read and stored. <em>Depending on your application, the use of WebRequests could be prohibited or a potential security issue to load unknown remote XML documents.</em>

Note that you can also use the implicit casting of the LOB types to implicitly cast some known types. There are also Blob.Empty, Clob.Empty and Xlob.Empty singletons that you can use to insert empty data. This is not null. The following sample implicitly casts a Stream to a Blob, a String to a Clob and removes the product's specification by replacing it with an empty one.

{% highlight c# %}
using (ISession session = sessionFactory.OpenSession())
using (ITransaction transaction = session.BeginTransaction())
{
	Product product = session.Get<Product>(100);
	product.Image = Request.Files["uploadedImage"].InputStream;
	product.Description = Request.Form["description"];
	product.Specifications = Xlob.Empty;
	transaction.Commit();
}
{% endhighlight %}

If you prefer a less anemic domain model you could keep the LOB internal to the class and do reads and writes with custom logic.

{% highlight c# %}
public class Product
{
	public int ID { get; set; }
	private Blob image;

	public void ChangeImage(Stream input)
	{
		this.image = input;
	}

	public void CopyImageFrom(Product product)
	{
		this.image = product.image;
	}

	public void WriteImageTo(Stream output)
	{
		this.image.WriteTo(output);
	}
}
{% endhighlight %}

Note that the stream used in ChangeImage() will not be read and disposed of until your Session is Flushed. Depending on your application design this pattern may not be useful.

*In the current version, the StreamBlob class which wraps a Stream as a Blob can only be read once if the Stream is not seekable. Therefore each instance can only be saved to one entity and not reused. In future versions it may replace the internal stream pointer to the one in the repository once the first one is saved. The same goes for the TextReader and XmlReader wrappers.*

##Getting Started
The full source code to Calyptus.Lob and appropriate NHibernate mappings are available at our <a href="http://github.com/calyptus/calyptus.lob/">Calyptus.Lob project on GitHub</a>. I'll make some official builds once the code stabilizes.

##Coming up
 * <a href="http://blog.calyptus.eu/seb/2009/03/large-object-storage-for-nhibernate-part-2-storage-options/">Part 2 - Storage Options</a>
 * Part 3 - NHibernate Mappings
 * Part 4 - External Storage
 * Part 5 - Compression Options
