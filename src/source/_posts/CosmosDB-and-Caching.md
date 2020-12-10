---
title: CosmosDB and Caching
date: 2020-09-01 21:26:09
tags:
  - CosmosDb
  - Azure
  - Redis
  - Caching
---


Iff you ever work with CosmosDb and are concerned about high volume accesses:

CosmosDb is fast, no doubt, but because it's never hosted in the same network as your app-services, Azure functions etc., there is still significant network lag. Because of this, it makes a huge difference to put a redis instance in front. 

For me, in a current project, the performance gain was enormous: reads went down from 300-500ms to 50-100ms per read.

One caveat is this works well only for non-query reads - like "get this specific document". For server-side queries, this doesn't work. So depending on your usage, your mileage may vary. In my scenario, the non-query reads overwhelmingly outnumbered the query-reads statistically, so adding redis was a big win. 

As for implementation, assuming you got something like an `IRepository` that abstracts access CosmosDb, you just add another implementation of `IRepository`, `RedisCachingRepositoryDecorator`. For each write this decorator also writes to the cache and for each read it first asks the cache; if it gets a hit, it returns that hit. If no (this should really only happen when Redis restarts or starts throwing out old data), it proceeds to ask CosmosDb, updates the cache and returns the result.

Here's a sketch:

```csharp
public interface ICache         // this interface is a custom abstraction over the cache that's not as Redis-specific as IDatabase
{
    Task PutAsync<T>(string key, T value, TimeSpan timeOut);
    Task<T> GetAsync<T>(string key);
    Task InvalidateAsync(params string[] keys);
}


public interface IIdentifiable
{
    [JsonProperty(PropertyName = "id")]
    string Id { get; }
}

public interface IRepository<TDocument> where TDocument : class, IIdentifiable
{
    Task<TDocument> GetAsync(string key);
    Task PutAsync(TDocument record);
    Task<string[]> DeleteAsync(Expression<Func<TDocument, bool>> filter);
}

public class RedisCachingRepositoryDecorator<TDocument> : IRepository<TDocument>
       where TDocument : class, IIdentifiable
{
    readonly ICache _cache;
    readonly IRepository<TDocument> _decoree;
    readonly TimeSpan _expiry;

    public CachingDocumentRepositoryDecorator(IRepository<TDocument> decoree, ICache cache, TimeSpan expiry)
    {
        _decoree = decoree;
        _cache = cache;
        _expiry = expiry;
    }

    public async Task<TDocument> GetAsync(string key)
    {
        var cacheKey = MakeCacheKey(key);
        var result = await _cache.GetAsync<TDocument>(cacheKey);
        if (result != default) return result;
        result = await _decoree.GetAsync(key);
        await _cache.PutAsync(cacheKey, result, _expiry);
        return result;
    }
    
    public Task PutAsync(TDocument record) =>
        Task.WhenAll(_cache.PutAsync(MakeCacheKey(record.Id), record, _expiry), _decoree.PutAsync(record));

    public async Task<string[]> DeleteAsync(Expression<Func<TDocument, bool>> filter)
    {
        var ids = await _decoree.DeleteAsync(filter);
        await _cache.InvalidateAsync(ids.Select(MakeCacheKey).ToArray());
        return ids;
    }

    string MakeCacheKey(string id) => $"{typeof(TDocument).FullName}_{id}";
}
```

 



 