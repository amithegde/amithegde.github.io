---
layout: post
title: "Replacing Azure Storage Table Entity with Optimistic Concurrency"
category: c#, Azure, Azure Storage
forreview: false
filename: "2015-7-6-replacing-azure-storage-table-entity-with-optimistic-concurrency.md"
---

Azure Storage Table offers [Optimistic Concurrency Control (OCC)](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) for TableEntities. Here is a detailed [blog post](http://azure.microsoft.com/blog/2014/09/08/managing-concurrency-in-microsoft-azure-storage-2/) on azure and concurrency control.

In this post let's write a reusable C# method which replaces an entity using OCC, in the sense, if someone else has updated the entity before we could update it, we will have to pull the entity again, make our changes to the entity and write it to the storage.

In short, If the ETag maintained on the server is different from the one on the entity sent from client, azure storage service doesn't update the entity, instead it throws an exception with HTTP status code 409.

Let us define the helper method to replace entity:

{% highlight csharp %}

/// <summary>
/// Tries to replace an entity with Optimistic Concurrency Control with retry.
/// It updates the entity using the <see cref="entityUpdateAction"/> and tries to replace it on storage
/// If the replace fails (doesn't check for specific HTTP exception code), retrieves the entity again,
/// and tries to replace it on storage again after updating the properties using <see cref="entityUpdateAction"/>
/// </summary>
/// <typeparam name="T">Entity type</typeparam>
/// <param name="tableName">table name to replace entity</param>
/// <param name="entity">entity to replace</param>
/// <param name="entityUpdateAction"><see cref="Action"/> to update the entity properties</param>
/// <param name="retryCount">Retry count to try replacing. Default is 3</param>
public void ReplaceEntityOptimistically<T>(string tableName, T entity, Action<T> entityUpdateAction, int retryCount = 3)
    where T : TableEntitity, new()
	{
    for (int retryIndex = 0; retryIndex < retryCount; retryIndex++)
    {
        try
        {
            //update entity properties
            entityUpdateAction(entity);

            //try to replace
            ReplaceEntity(tableName, entity);

            break;
        }
        catch (Exception ex)
        {
            if (retryIndex == retryCount - 1)
            {
                Logger.LogException(ex,
                    "Could not Replace Entity. Failed Entity: {0}"
                        .ToFormat(entity));
                break;
            }

            //retrieve the entity again and try to replace it again
            entity = RetrieveEntity<T>(tableName, entity.PartitionKey, entity.RowKey);

            if (entity.IsNull())
            {
                break;
            }
        }
    }
}

public void ReplaceEntity<T>(string tableName, T entity) where T : TableEntity, new()
{
    CloudTable table = TableClient.GetTableReference(tableName);
    TableOperation tableOperation = TableOperation.Replace(entity);
    table.Execute(tableOperation);
}
		
{% endhighlight %}

As you can see, method `ReplaceEntityOptimistically` iterates over `retryCount` and tries to update the entity. If fails, it pulls a latest copy of the entity from server, makes necessary changes on the entity using `entityUpdateAction` and tries to update the entity again. `RetrieveEntity` method is a trivial method to retrieve the entity again from server.

Now, using this method is simple. All we need is an action to make changes to the entity properties. For example:

{% highlight csharp %}

MyEntity myObject = GetEntity(); // get an entity from server

Action<MyEntity> entityUpdateAction = (entity) => { entity.SomeProp = "some value"; };
ReplaceEntityOptimistically("myTableName", myObject, entityUpdateAction);

{% endhighlight %}