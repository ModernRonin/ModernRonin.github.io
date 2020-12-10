---
title: Testing with Flurl
date: 2020-10-13 23:09:04
tags:
  - TDD
  - REST APIs
  - Flurl
---

When developing against REST APIs without dedicated client libraries, I very much like to use [Flurl.Http](https://flurl.dev/), because it makes it pretty easy and intuitive to test your code, also under several error conditions.

Assume the following production method:

```csharp
public Task<Dto> GetTenantLicenseAsync(long tenantId, CancellationToken ct = default) =>
    _baseAddress.AppendPathSegments("tenants", tenantId, "lic").GetJsonAsync<Dto>(ct);
```

(Note that for the purposes of this example, the code leaves out authentication, but you can work with this just as well.)

First, note how nicely you build the requests. This already is courtesy of Flurl. But that in itself would not much distinguish it from the likes of [RestSharp](https://restsharp.dev/) and similar libraries. The really interesting part comes when you want to test your code. Let's start with a test for the [happy path](https://en.wikipedia.org/wiki/Happy_path):

```csharp
[Test]
public async Task GetTenantLicenseAsync()
{
    var response = new Dto
    {
        CurrentNumberOfItemsInProject = new Dictionary<string, uint> {["p1"] = 12},
        CurrentNumberOfProjects = 1,
        CurrentlyUsedItemStorage = 1000,
        MaximumItemSize = 200,
        MaximumNumberOfItemsPerProject = 100,
        MaximumNumberOfProjects = 2,
        MaximumTotalSize = 2000,
        TenantId = 13
    };
    _http.RespondWithJson(response);

    var result = await _underTest.GetTenantLicenseAsync(13);
    
    result.Should().BeEquivalentTo(response);
    _http.ShouldHaveCalled("http://api/tenants/13/lic").WithVerb(HttpMethod.Get);
}
```
As you see, it's really quite simple to setup responses and check whether the right calls were made. 

Now let's look at simulating errors:
```csharp
[Test]
public void Call_resulting_in_Unauthorized_throws()
{
    _http.RespondWith(string.Empty, 401);

    Func<Task> action = () => _underTest.GetTenantLicenseAsync(13);

    action.Should().Throw<FlurlHttpException>();
}
```

In our example, the production code doesn't do anything to handle the exception, so we expect it to be thrown, but you could just as well test your sophisticated error-handling behavior this way. 


However, there are certain error conditions you cannot easily simulate even with Flurl:
* no network connectivity
* DNS resolution issues
* generally, most low-level socket related problems

If it's enough for you to test a general "network not available" handler, then you can help yourself by setting up your client with an invalid base-address. This will trigger a `SocketError.HostNotFound`, so if your code doesn't distinguish between different socket-level error-conditions, you got a sufficient testing strategy with this.


