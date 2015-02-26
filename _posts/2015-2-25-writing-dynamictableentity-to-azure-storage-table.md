---
layout: post
title: "Writing DynamicTableEntity to Azure Storage Table"
category: c#, Azure, Storage Table
forreview: false
filename: "2015-2-25-writing-dynamictableentity-to-azure-storage-table.md"
---

There are ample of samples available to show how to insert an object/entity to Azure Storage Table. However, all the samples inherit from `TableEntity`

This sample shows how to insert custom entities to table when we don't have a class that inherits from `TableEntity`.

{% highlight csharp %}
void Main()
{
	var account = "";
	var key = "";
	var tableName = "";

        var storageAccount = GetStorageAccount(account, key);
	var cloudTableClient = storageAccount.CreateCloudTableClient();
	var table = cloudTableClient.GetTableReference(tableName);
	
	var partitionKey = "pk";
	var rowKey = "rk";
	
	//create the entity
	var entity = new DynamicTableEntity(partitionKey, rowKey, "*", 
		new Dictionary<string,EntityProperty>{
				{"Prop1", new EntityProperty("stringVal")},
				{"Prop2", new EntityProperty(DateTimeOffset.UtcNow)},
			});
	
	//save the entity
	table.Execute(TableOperation.InsertOrReplace(entity));
	
	//retrieve the entity
	table.Execute(TableOperation.Retrieve(partitionKey,rowKey)).Result.Dump();
}

static CloudStorageAccount GetStorageAccount(string accountName, string key, bool useHttps = true)
{
	var storageCredentials = new StorageCredentials(accountName, key);
	var storageAccount = new CloudStorageAccount(storageCredentials, useHttps: useHttps);
	return storageAccount;
}
{% endhighlight %}
This code makes use of `DynamicTableEntity` which can take properties and values as `IDictionary`.

This code is written for [LinQPad][2]. Introduction on how to use Azure Storage Table can be found [here][2].

[1]:http://www.linqpad.net/
[2]:http://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-tables/
