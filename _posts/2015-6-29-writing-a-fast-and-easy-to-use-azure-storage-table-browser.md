---
layout: post
title: "Writing a Fast and Easy to use Azure Storage Table Browser"
category: c#, Azure, Azure Storage, Tool
forreview: false
filename: "2015-6-29-writing-a-fast-and-easy-to-use-azure-storage-table-browser.md"
---
	
There are many tools that let you query data from [Azure Storage Table](https://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-tables/) including `Visual Studio`. But none of the tools are easy and fast as `SQL Server Management Studio` which most of us are comfortable with. 

Though data looks very similar to the data on `RDBMS` once it's on UI, none of the tools provide an intuitive user experience. Most of the tools I see around are either paid or if free, need lot of clicking around to get things done. 

Half the world which uses `Azure Storage` uses `Azure Storage Explorer`, which doesn't even let you copy text (authors of Azure Storage Explorer, if you are reading this don't hesitate to fix it..!). Every developer I have come across is looking for a better tool, but most of them were asked to use this tool by a senior dev in the project and the leniage continued. Even I was handed over a copy of this tool and probably I used it for six months.

And one day I decided to write up my own tool out of the frustration of using this tool. For the readers who want to dive in to the code right away, here it is: [Azure Table Browser](https://github.com/amithegde/AzureTableBrowser).  Executable Binaries [here](https://github.com/amithegde/AzureTableBrowser/raw/master/Binaries.zip). 

This is the first part of a tool which does CRUD operations on `Azure Storage` and supports read only operations on table as of now. If the tool gets some traction, I plan to open source more code... :)

In this post, I will explain about two important sections of this tool which make it useful and possible.

Converting table result received as [DynamicTableEntity](https://msdn.microsoft.com/library/azure/microsoft.windowsazure.storage.table.dynamictableentity.aspx) to a flat structure.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

If you look through the class, all the properties of the entity (other than the default properties like PartitionKey) are exposed as 

{% highlight csharp %}

public IDictionary<string,EntityProperty> Properties { get; set; }

{% endhighlight %}


`EntityProperty` class exposes the value as set of properties, for example here is the code for `BooleanValue` property.

{% highlight csharp %}

		public bool? BooleanValue
		{
			get
			{
				if (!this.IsNull)
				{
					this.EnforceType(EdmType.Boolean);
				}
				return (bool?)this.PropertyAsObject;
			}
			set
			{
				if (value.HasValue)
				{
					this.EnforceType(EdmType.Boolean);
				}
				this.PropertyAsObject = value;
			}
			
{% endhighlight %}

There are many other such properties such as `DoubleValue`, `Int32Value` etc. While pulling data through `DynamicTableEntity` we would not be knowing the types of each of the properties and this makes us go through a set of type comparisons before we can extract the value of the property.

I wrote a method [ToDynamicList](https://github.com/amithegde/AzureTableBrowser/blob/master/src/AzureTableBrowser/AzureTableBrowser/Extensions/EnumerableExtensions.cs) which does all these comparisons, extracts values for all the properties along with default properties (such as `PartitionKey`) and  converts them to an [ExpandoObject](https://msdn.microsoft.com/en-us/library/system.dynamic.expandoobject%28v=vs.110%29.aspx). Now this becomes a flat structure instead of being a two level structure. `ToDynamicList` returns resulting list as a `List<dynamic>` which can be bound to grid on the UI. The result could be passed through a `Json` serializer in case of a web application.

This method should have been made available on the `Storage Library` itself instead of making developers write this method, as `dynamic` type has been around for long time and libraries like [dapper-dot-net](https://github.com/StackExchange/dapper-dot-net) have shown it to be pretty powerful. May be they would consider it if many request come in.. :)

Supporting client-side LINQ queries
-----------------------------------

`Azure Storage` supports [OData](http://www.odata.org/) query format which comes with a [limited set of query operators](http://msdn.microsoft.com/en-us/library/windowsazure/ff683669.aspx). Though this works well for filtering data over `HTTP`, we can't leverage on the advanced capabilities of [LINQ](https://msdn.microsoft.com/en-us/library/bb397926.aspx) to filter and aggregate the data. So I wrote a wrapper around this to perform `LINQ ` operations on the data received from server. Here is the original project [TextToLINQ](https://github.com/amithegde/TextToLINQ) with some example usage. And the [code](https://github.com/amithegde/AzureTableBrowser/blob/master/src/AzureTableBrowser/AzureTableBrowser/Helpers/TextToLinq.cs) if you want to look at the core of it right away.

The idea behind this class is simple and is reusable. given a compile-able c# `LINQ` expression
- it will enclose it in a program

- compiles an assembly

- instantiates an object of the assembly

- passes the collection to the assembly

- collects the result

Now on to [Azure Table Browser](https://github.com/amithegde/AzureTableBrowser)..! Use it and let me know how you feel.