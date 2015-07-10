---
layout: post
title: "Pretty Dump LINQPad Extension method for DynamicTableEntity"
category: c#, Azure, Azure Storage, Extensions
forreview: false
filename: "2015-7-10-pretty-dump-linqpad-extension-method-for-dynamictableentity.md"
---

Often, I write my [Azure Storage Table](https://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-tables/) queries on [LINQPad](https://www.linqpad.net/) and copy over to Visual Studio. I find this workflow faster as I can quickly execute the queries and see the results using the `Dump()` extension method. (Many people have tried to port it to Visual Studio and if you are looking for something similar look [here](http://stackoverflow.com/questions/2699466/linqpad-dump-extension-method-i-want-one) and [here](https://github.com/fragilerus/fragilerus-linqpad-visualizer))

I generally use [DynamicTableEntity](https://msdn.microsoft.com/library/azure/microsoft.windowsazure.storage.table.dynamictableentity.aspx) on `LINQPad` as it saves some time. But since `DynamicTableEntity` contains all the properties as `IDictionary<string,EntityProperty>` calling `Dump()` on it won't print an easy to read output.

But wait, I discussed about [ToDynamicList](https://github.com/amithegde/AzureTableBrowser/blob/master/src/AzureTableBrowser/AzureTableBrowser/Extensions/EnumerableExtensions.cs) method I wrote for [Azure Table Browser](https://github.com/amithegde/AzureTableBrowser) on [this post](http://www.amithegde.com/2015/06/writing-a-fast-and-easy-to-use-azure-storage-table-browser.html). So passing the `DynamicTableEntity` collection through this method before dumping will convert it to a flat list. Quick and easy..!

But calling `ToDynamicList()` every time before calling `Dump()` is tedious. can we just override the original `Dump()` method? Well, I did not find a strait-forward way to do it; though, I kind of got it working.

{% highlight csharp %}

public static void Dump(this object obj)
{
	if (obj.GetType() == typeof(List<DynamicTableEntity>))
	{
		var data = obj as List<DynamicTableEntity>;

		if (data == null)
		{
			obj.Dump();
		}
		else
		{
			data.ToDynamicList().Dump("Storage Table Entities");
		}
	}
	else
	{
		obj.Dump(obj.GetType().Name);
	}
}

{% endhighlight %} 

One limitation here, is I need to do a `ToList()` before calling `Dump()` since the type is not `IEnumerable<DynamicTableEntity>` at run-time,  easy hack was to do a `ToList()`.

Paste this custom `Dump()` extension method on `LINQPad`'s `My Extensions` file along with `ToDynamicList` extension method, and resolve the dependencies. You are all set to use it. Here is a simple driver program to show how to use it:

{% highlight csharp %}

void Main()
{
	var account = "";
	var key = "";
	var tableName = "";

	var cloudTable = GetStorageAccount(account, key).CreateCloudTableClient().GetTableReference(tableName);
	var tableQuery = new TableQuery<DynamicTableEntity>().Where(filter: string.Empty);

	var timer = Stopwatch.StartNew();
	
	//do a ToList() on it
	var data = cloudTable.ExecuteQuery(tableQuery).ToList();
	data.Dump();
	
	timer.Stop();
	timer.Dump();
}

static CloudStorageAccount GetStorageAccount(string accountName, string key, bool useHttps = true)
{
	var storageCredentials = new StorageCredentials(accountName, key);
	var storageAccount = new CloudStorageAccount(storageCredentials, useHttps: useHttps);
	return storageAccount;
}

{% endhighlight %}

I have created a gist with complete code [here](https://gist.github.com/amithegde/93b658784fcafb1a5676). And if you find a better way to override `Dump()`, do let me know in the comments... :)