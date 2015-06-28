---
layout: post
title: "Decoupling TableEntity while using Azure Storage with .Net"
category: c#, Azure, Azure Storage, Patterns
forreview: false
filename: "2015-6-28-decoupling-tableentity-while-using-azure-storage-with-dot-net.md"
---

[Azure Storage](http://azure.microsoft.com/en-us/services/storage/) has a nice documentation [here](https://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-tables/#create-a-table) and is easy to get started with.

All the examples on the documentation mention that `Entities` on `Azure Storage` map to classes derived from [TableEntity](https://msdn.microsoft.com/en-us/library/microsoft.windowsazure.storage.table.tableentity.aspx). However, this would not fit well with some of the design goals. This post is all about how to abstract this requirement to derive each class from `TableEntity` and keep everything tidy.

`TableEntity` class implements `ITableEntity` and it has [two methods](https://msdn.microsoft.com/en-us/library/microsoft.windowsazure.storage.table.itableentity_methods.aspx) to read/write data to the object. We will be leveraging the same to come up with a solution.

Let's create a base class, which we can use to derive all Entity classes.

{% highlight csharp %}
public class StorageTableEntityBase
{
public string ETag { get; set; }

public string PartitionKey { get; set; }
public string RowKey { get; set; }
public DateTimeOffset Timestamp { get; set; }

#region ctor

public StorageTableEntityBase()
{

}

public StorageTableEntityBase(string partitionKey, string rowKey)
{
    PartitionKey = partitionKey;
    RowKey = rowKey;
}

#endregion
}
{% highlight csharp %}

PartitionKey, RowKey, Timestamp and ETag are the default properties each entity on Azure Storage possess. `StorageTableEntityBase` class defined above lets us to contain these properties, copied over from the entity.

Let us create an adapter class which implements `ITableEntity`:

{% highlight csharp %}
internal class AzStorageEntityAdapter<T> : ITableEntity where T : StorageTableEntityBase, new()
{
#region Properties
/// <summary>
/// Gets or sets the entity's partition key
/// </summary>
public string PartitionKey
{
    get { return InnerObject.PartitionKey; }
    set { InnerObject.PartitionKey = value; }
}

/// <summary>
/// Gets or sets the entity's row key.
/// </summary>
public string RowKey
{
    get { return InnerObject.RowKey; }
    set { InnerObject.RowKey = value; }
}

/// <summary>
/// Gets or sets the entity's Timestamp.
/// </summary>
public DateTimeOffset Timestamp
{
    get { return InnerObject.Timestamp; }
    set { InnerObject.Timestamp = value; }
}

/// <summary>
/// Gets or sets the entity's current ETag.
/// Set this value to '*' in order to blindly overwrite an entity as part of an update operation.
/// </summary>
public string ETag
{
    get { return InnerObject.ETag; }
    set { InnerObject.ETag = value; }
}

/// <summary>
/// Place holder for the original entity
/// </summary>
public T InnerObject { get; set; } 
#endregion

#region Ctor
public AzStorageEntityAdapter()
{
    // If you would like to work with objects that do not have a default Ctor you can use (T)Activator.CreateInstance(typeof(T));
    this.InnerObject = new T();
}

public AzStorageEntityAdapter(T innerObject)
{
    this.InnerObject = innerObject;
} 
#endregion

#region Methods

public virtual void ReadEntity(IDictionary<string, EntityProperty> properties, OperationContext operationContext)
{
    TableEntity.ReadUserObject(this.InnerObject, properties, operationContext);
}

public virtual IDictionary<string, EntityProperty> WriteEntity(OperationContext operationContext)
{
    return TableEntity.WriteUserObject(this.InnerObject, operationContext);
} 

#endregion
}

{% highlight csharp %}

I have marked the class as `internal` to limit its access to current assembly. This class copies over properties such as `PartitionKey`, implements `ReadEntity` and `WriteEntity` methods to copy over rest of the properties to a temporary object `InnerObject`.

We can now define our `Entity Class` as follows:

{% highlight csharp %}
public class UserEntity : StorageTableEntityBase
{
    public string UserName { get; set; }
    public string Email { get; set; }
}
{% highlight csharp %}

This is all the code we need to get it working. Now we can write a method to read from Storage Table as follows:

{% highlight csharp %}
public T RetrieveEntity<T>(string tableName, string partitionKey, string rowKey)
        where T : StorageTableEntityBase, new()
{
    CloudTable table = TableClient.GetTableReference(tableName);
    TableResult tableResult = table.Execute(TableOperation.Retrieve<AzStorageEntityAdapter<T>>(partitionKey, rowKey));
    if (tableResult.Result != null)
    {
        return ((AzStorageEntityAdapter<T>)tableResult.Result).InnerObject;
    }
    return default(T);
}
{% highlight csharp %}

I have not included boilerplate methods instantiate `TableClient`,  which is a trivial part and is available on the [documentation](https://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-tables/#create-a-table).