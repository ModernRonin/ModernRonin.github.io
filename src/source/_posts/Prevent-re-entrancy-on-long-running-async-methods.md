---
title: Prevent re-entrancy on long running async methods
date: 2020-12-17 19:57:20
tags:
  - async
  - re-entrancy
  - long-running
  - tdd
---

Sometimes you got async methods you know will run for some time and you want to prevent them from being called while a previous call has not yet finished.

For example, imagine a tool that creates a visualization of the storage use per folder on your harddrive. In order to do that, it has to scan all folders recursively to get the size data. This is a very long task indeed, probably on the order of minutes. 

Now while that scanning process is running, it makes hardly any sense to start another one, in fact doing so would slow down the first. Of course, you can prevent that on the UI level, by just disabling the command bound to the "Scan" button once a scan has started and re-enabling it once the scan has finished.

But what if you are writing the API of the file-system scanner, but not the UI? In other words, you got no control over the UI, yet you still want to prevent such simultaneous calls to your scan method?

Let's say you expose the following API:

```csharp
public class FileSystemFacade
{
    readonly IFolder _rootFolder;

    // ... other stuff like the constructor etc.

    public Task<long> GetTotalSizeAsync() => _rootFolder.GetSizeAsync();
}
```

where `IFolder.GetSizeAsync()` is the method that can run for quite some time.

Remember that Tasks are not really anything special (although the compiler builds some magic around them when encountering `await` keywords), they are just regular reference types, classes. So how about when being called we just remember the task and when called again we first check if it's still running? If it is, we just return it, and only if it is no longer running we start a new one?

What would a test for this behavior look like?

```csharp
[Test]
public async Task GetTotalSizeAsync_reuses_running_tasks()
{
    var root= Substitute.For<IFolder>();
    var underTest= new FileSystemFacade(root);

    var t1 = new Task<long>(() => 13);
    var t2 = new Task<long>(() => 17);
    root.GetSizeAsync().Returns(t1, t2);  // first call returns t1, 2nd t2
    
    // on the first call we should get t1
    underTest.GetTotalSizeAsync().Should().Be(t1);

    // make a 2nd call while the 1st is still running
    // should still be t1
    underTest.GetTotalSizeAsync().Should().Be(t1); // HERE

    // now make t1 finish
    t1.Start();
    await t1;

    // make a 3rd call after the 1st and 2nd have finished
    // this should now be t2
    underTest.GetTotalSizeAsync().Should().Be(t2);

    // tidy up
    t2.Start();
    await t2;
}
```

With the existing implementation, the line marked with `// HERE` will fail. 

So far so good, we got an implementation idea and a failing test. Now let's make that test pass by changing our implementation:

```csharp
public class FileSystemFacade
{
    readonly IFolder _rootFolder;
    Task<long> _sizeTask;

    // ... other stuff like the constructor etc.

    public Task<long> GetTotalSizeAsync()
    {
        if (_sizeTask != null && !_sizeTask.IsCompleted) return _sizeTask;
        return _sizeTask = _rootFolder.GetSizeAsync();
    }
}
```

Et voilà, our test passes.

Of course, doing this means that the latest call to our method does not necessarily get the most up-to-date result. In many scenarios, though, like the disk size visualizer example used, this will not really matter. 