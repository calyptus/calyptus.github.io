---
layout: post
title: Large Object Storage for NHibernate - Part 2 - Storage Options
date: 2009-03-29 05:24:06.000000000 +02:00
categories:
  - seb
tags:
  - dotnet
  - blob
  - clob
  - xlob
  - domain-driven-design
  - nhibernate
  - cas
  - mysql
  - postgresql
  - sqlserver
  - oracle
  - streaming
---
*This is part 2 of a series describing Large Object Storage (BLOB) in a Domain Driven fashion. Be sure to read <a href="http://blog.calyptus.eu/seb/2009/03/large-object-storage-for-nhibernate-and-ddd-part-1-blobs-clobs-and-xlobs/">Part 1 about the new base classes introduced by this project</a>.*

## Physical Storage Considerations
So, what are you options of storing large data objects in your relational database? This is actually not an easy problem to solve. Because a relational database is designed for small pieces of well structured data. Making a table, row or column too large will cause various problems with fragmenting, indexing and table scans.

Because of this, vendors have implemented data columns that store large data separately from the rest of the row. This typically means they can't be used for indices or searches. They're still internal to the RDMS and are fully covered by ACID transactions and backup procedures. You would typically keep the large object data on the same discs as the actual database itself. This can limit your overall performance and scalability. Additionally, the vendor API might support streaming for reading, while not supporting streaming for writing.

To remedy this situation vendors have come up with various ways of storing your data externally to your database (typically in a file system) while storing references to your data in the database and allowing you to access it and manage access control through your RDMS. This typically means that operations on these files are not covered by ACID transactions<sup>1</sup> and backup procedures. External storage allows you to save disc space, since multiple rows and tables can share a reference to the same data file in a true denormalized fashion. <a href="http://en.wikipedia.org/wiki/Content-addressable_storage">Content-addressable storage</a> (CAS) solutions are especially suitable for this kind of storage. You can use external storage with or without RDMS integration.

So to sum up your physical storage options:
 * In-table RDMS storage
 * Out-of-table RDMS storage
 * External storage using RDMS integration
 * External storage using NHibernate client

Because of these issues, various vendors have implemented more than one solution and there isn't a consistent best-practise of working with large data. You have to chose the storage solution that is most appropriate for your particular requirements.

*<sup>1</sup> In this series I will only cover complete replacements of data rather than changes of data. This is done by exchanging one blob object for another as described in <a href="http://blog.calyptus.eu/seb/2009/03/large-object-storage-for-nhibernate-and-ddd-part-1-blobs-clobs-and-xlobs/">Part 1</a>. Therefore ACID transactions on individual data changes aren't going to be important for external storage. The entire blob will be written to storage. If the entire transaction succeeds, the reference will be changed. Otherwise the reference will remain at the old data.*

## Data Transfer Considerations
Accessing in-row data is usually sent with the rest of the data result of the query. This means that it is not typically viable for streaming because the entire row is always read in to memory.

For out-of-table storage some vendors doesn't send large object data with the rest of the row result. That means that it can be requested and streamed in pieces. However, because ADO.NET doesn't offer a requirement and API for this, it is usually done in vendor specific implementations. Some vendors require the data reader to remain open while reading the stream. This makes it unsuitable for NHibernate since we would like to work with our entities in a disconnected fashion. So in this case, we would have to query that row and column again to open the data connection when needed (lazy loading).

When external storage is used, only a reference to the data is sent with the query result. The actual data transfer is usually done over a protocol completely separate to the RDMS connection. Sometimes it isn't even communicating with the same machine as the database. This makes it a very scalable solution. It will also allow us to open that connection and stream the data without querying the row and column of the database again.

Because of the inconsistent ways of accessing the data, we will addressing this at the client level in a vendor specific fashion. More on this in Part 3 - NHibernate Mappings.

## Small In-Table Data Types
All RDMS has small in-row data types. In-row binary and text data. Such as VARBINARY(size) or VARCHAR(size). These are typically limited to around 4000-8000 bytes of data and are therefore not suitable for large objects. You would typically just map these to memory using byte[] and string. If you currently only have small amounts of data but expect it to scale, you can start off using one of these small data types and map it to Blob and Clob objects and then scale as you need it.

Large object storage options are highly vendor specific. I'll cover a few common vendors.

## Microsoft SQL Server
<a href="http://msdn.microsoft.com/en-us/library/ms188362.aspx">VARBINARY(MAX)</a>, <a href="http://msdn.microsoft.com/en-us/library/ms187993.aspx">VARCHAR(MAX)</a> - These types is used to store in-table binary and text data at up to 2 GB. The practical performance and scalability limitations involved in storing data in-table usually means that you want to keep data in these columns to a few MB. Using the <a href="http://msdn.microsoft.com/en-us/library/3517w44b(VS.80).aspx">UPDATETEXT</a> command in SQL Server you can write changes to the database in chunks.

<a href="http://msdn.microsoft.com/en-us/library/ms187993.aspx">XML(DOCUMENT), XML(CONTENT)</a> - You can use the XML data type to store up to 2 GB of XML data per column. The same practical limitations as for VARBINARY and VARCHAR applies. You can specify either DOCUMENT or CONTENT to indicate whether the data has to comply to either a full XML document or an XML fragment. The Xlob base class allows for both complete documents and fragments.

<em><a href="http://msdn.microsoft.com/en-us/library/ms187993.aspx">IMAGE, TEXT and NTEXT</a> - These data types are now deprecated and will be removed in future versions of SQL Server. Use VARCHAR(MAX) or VARBINARY(MAX) instead.</em>

<a href="http://technet.microsoft.com/en-us/library/bb933993.aspx">FILESTREAM</a> - If your data is more more than 1 MB on average, you should consider the new FILESTREAM data type introduced in SQL Server 2008. It stores data out-of-table in the NTFS file system. The data size is limited only by the local NTFS file system. Each row that uses a FILESTREAM column must have UNIQUEIDENTIFIER. The FILESTREAM column is completely integrated with SQL Server, it's backup facilities and it's client software.

<a href="http://www.codeplex.com/sqlrbs">Microsoft SQL Remote Blob Storage (RBS)</a> - Microsoft has introduced a new plug-in API for external storage used together with SQL Server. This will allow any storage solution provider to hook into Microsoft's common API. It's installed both on the server and the client. The server handles garbage collecting and manages the references to various BLOBs in the storage solution. This is a flexible and highly scalable solution and it integrates nicely into the SQL Server product. If you want to leave the external storage API to the client, read on to External Storage.

## Oracle
<a href="http://download.oracle.com/docs/cd/B19306_01/server.102/b14220/datatype.htm#i3237">BLOB, CLOB, NCLOB</a> and <a href="http://download.oracle.com/docs/cd/B19306_01/server.102/b14220/datatype.htm#i13446">XMLType</a> - These are out-of-table data types for storing binary, text and XML data up to 4 GB. Oracle 10g and above supports up to 8 terabytes of storage depending on your CHUNK setting for the table. NCLOB stores text data in a Unicode national character set.

Oracles LOB types are all stored out-of-table and referenced using Lob locators. This makes them suitable for the disconnected environment used by NHibernate.

Oracle allows for XML operations to take place on the server which, in the future, could be used to speed up operations of a XmlReader generated by a Xlob.

*<a href="http://download.oracle.com/docs/cd/B19306_01/server.102/b14220/datatype.htm#i4146">LONG and LONG RAW</a> - These data types are now deprecated. They can store 2 GB of data. Use CLOB or BLOB instead.*

<a href="http://download.oracle.com/docs/cd/B19306_01/appdev.102/b14249/adlob_intro.htm#sthref36">BFILE</a> - Oracle has reference type that points to files on the local file system. You can read these files (up to 4GB) via the Oracle API. You can't write to them though. You can change the reference to another file in the file system. So if you create a reference to an existing file using Blob.Create("filepath"), the NHibernate mappings will be able to change out the reference to the new file. You can also open up a directory where NHibernate can store new files. In both cases, both the Oracle server and client will need access to this directory. BFILEs are an external storage solution. Oracle doesn't handle write transactions, garbage-collecting of files nor backup procedures.

## PostgreSQL
<a href="http://www.postgresql.org/docs/8.3/interactive/datatype-binary.html">BYTEA</A>, <a href="http://www.postgresql.org/docs/8.3/interactive/datatype-character.html">TEXT</a> and <a href="http://www.postgresql.org/docs/8.3/interactive/datatype-xml.html">XML</a> - Used for in-table binary, text and XML data respectively. Current APIs doesn't support streaming of these types. They will have to be read in to memory all at once.

<a href="http://www.postgresql.org/docs/8.3/interactive/storage-toast.html">TOAST</a> - PostgreSQL normally stores it's data in tuples of 8 kb which doesn't allow the above data types to be very large. Using TOAST large columns are automatically stored out-of-table. It also has mechanisms for compressing data and trying to fit it in to rows if possible. TOAST isn't it's own data type but can be used to expand BYTEA, TEXT and XML columns to a maximum of 1 GB.

<a href="http://www.postgresql.org/docs/8.3/interactive/largeobjects.html">Large Objects</a> - PostgreSQL supports the notion of Large Objects. These are stored out of table but within the management of the RDMS itself. Each new object is given it's own ID and it is this ID that is referenced in the data tables. These objects are read and manipulated using a special API. Each object can be referenced several times and across tables. As far as I know it is not garbage-collected nor handled by backup solutions. So this solution can be compared to other external solutions even though it is managed by PostgreSQL itself. Since this solution shares objects for the entire database you will have to incorporate your own custom garbage collecting solution.

Large Objects are useful when you need to store data larger than 1 GB. The documented limit is 2 GB but in practice you can store files of several GB depending on the file system. The Large Object API will also allow you to stream the data instead of reading it all into memory. Therefore this is the preferred solution for storing large data on PostgreSQL.

## MySQL
<a href="http://dev.mysql.com/doc/refman/5.1/en/blob.html">BLOB and TEXT</a> - These columns are used to store binary and text data up to 4 GB. MySQL doesn't have a column for XML data. TEXT is the recommended column type for XML. These columns are stored out-of-table but MySQL doesn't support streaming of data. This means that each object will have to be read into memory in it's entirety.

<a href="http://www.primebase.org/">PrimeBase Technologies</a> are currently working on a <a href="http://www.blobstreaming.org/">Blob streaming infrastructure</a> over HTTP to be integrated into MySQL. It uses their <a href="http://www.primebase.org/">XT storage engine</a>.

For other storage engines, you will need to look to external storage.

## External Storage
If you prefer to decouple your large object storage solution from the database you can use a completely external storage solution. In this case, you would store a reference to the data blob in your relational table. Usually as a fixed length binary or GUID/UUID. The data is stored in a completely external solution with no communication with the database. This makes this solution completely vendor independent and highly scalable.

The NHibernate.Lob client handles the communication with both the external storage solution as well as the database. Your client should on certain intervals (nightly?) let NHibernate.Lob scan all mapped tables for external references. It will then garbage collect the data blobs in the external storage that are no longer referenced.

The <a href="/source-code/">NHibernate.Lob</a> project includes a common API for external storage solutions for use with NHibernate. Included is also a file-system based <a href="http://en.wikipedia.org/wiki/Content-addressable_storage">CAS</a> storage option to get you started. High-end <a href="http://en.wikipedia.org/wiki/Content-addressable_storage">CAS</a> solutions such as EMC's Centera or Caringo's CAStor are very suitable for this kind of storage if you have extreme scalability or accessibility needs. They're also useful if you need to comply with local regulations that require you to never delete data.

## Text and XML Types

Clob and Xlob are structured data since they have a specific format (Text and XML). These can be stored in various ways depending on your vendor's specific data columns. Text can be stored in various different character sets. XML can be serialized as binary XML in storage or saved using various character sets. If your vendor does provide a specific Text or XML data type that is suitable for large objects I would recommend that you use it. This will allow the RDMS to handle the format and serialization constraints. Any compliant software can handle and display the data without further user interaction.

However, if you use external storage or want to utilize the various compression options mentioned in Part 5, you can store your Clob and Xlob data in any binary column as well. There by letting the client determine the serialization format.

## Getting Started
The full source code to Calyptus.Lob and appropriate NHibernate mappings are available at our <a href="http://github.com/calyptus/calyptus.lob/">Calyptus.Lob project at GitHub</a>.

## More in This Series
In the next part of this series I'm going to describe how you can use the <a href="http://blog.calyptus.eu/source-code/">NHibernate.Lob</a> project to map up these storage options to your <a href="/seb/2009/03/large-object-storage-for-nhibernate-and-ddd-part-1-blobs-clobs-and-xlobs/">Blobs, Clobs and Xlobs</a> in your NHibernate Entities.

<a href="/seb/2009/03/large-object-storage-for-nhibernate-and-ddd-part-1-blobs-clobs-and-xlobs/">Part 1 - BLOBs, CLOBs and XLOBs</a>

 * *Part 2 - Storage Options*
 * Part 3 - NHibernate Mappings
 * Part 4 - External Storage
 * Part 5 - Compression Options
