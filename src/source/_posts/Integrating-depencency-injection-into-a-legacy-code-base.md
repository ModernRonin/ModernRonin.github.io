---
title: Integrating dependency injection into a legacy code base
date: 2020-12-13 18:55:11
tags:
  - refactoring
  - software design
  - legacy code
  - long
  - ReSharper
---


Nobody likes it, but most of us eventually encounter a legacy code-base without tests and without IOC where everything revolves around the dreaded singleton pattern.

Now how do you work with such a beast? Refactoring the whole code-base to use IOC is usually as unrealistic as test-covering all existing code. For the testing aspect, accepted wisdom says to cover all new code and refactor code that you touch. 

But how do we deal with these singletons and the lack of IOC? Is there also a way to introduce it gradually, to create an island of sanity that can slowly grow over time?

In this article, I'll show you how I dealt with this just recently. I use [Autofac](https://autofac.org/) because this has been my preferred container framework for years. But any other IOC framework that has a concept analogous to [Autofac's modules](https://autofaccn.readthedocs.io/en/latest/configuration/modules.html) should enable the same pattern.

## Initial situation
Let's start with what I found before the refactoring:

There was a type called `Repository`, split over 8 partial classes, all together coming in at around 7000 lines of code. That type was really not a repository at all, but a typical [God object](https://wiki.c2.com/?GodClass). It contained only static members. At least 30-40% of calls into that class came from static members of other types or by singletons.

For reasons unrelated to this discussion, I had to change some functionality related to sending notifications (yes, I know - I told you it was a god-object) which was what prompted me to make that portion of behavior testable.

```csharp
// for this article I don't replicate the partial classes,
// they really don't make any difference
public static class Repository 
{
    // imagine a HUGE list of private fields here

    public static void SendTextNotification(Guid sender, Guid[] recipients, string text)
    {
        // ...
    }
    public static void SendTextNotification(Guid[] recipients, string text)
    {
        SendTextNotification(this.CurrentUserId, recipients, text);
    }
    public static void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state)
    {
        // ...
    }
    // imagine around 20 other Send... methods here

    // and don't forget the methods related to document upload, download
    // internal graph manipulation, caching, application versioning 
    // and updating and so on
}
```

## Convert static access to Singleton
The first thing to do is to turn the static `Repository` class into a singleton. While this doesn't seem like much of an improvement, it enables further refactoring down the line. 

```csharp
public class Repository 
{
    // from now on I leave out all members not relevant to the current discussion
    public void SendTextNotification(Guid sender, Guid[] recipients, string text) {...}
    public void SendTextNotification(Guid[] recipients, string text) 
        => SendTextNotification(this.CurrentUserId, recipients, text);
    public void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state) {...}

    public static Repository Instance {get;}= new Repository();
}
```
Basically: just remove all `static modifiers`, then add the canonical `Instance` property. 

Now of course you have to go over all callers and change calls from `Repository.Send...` to `Repository.Instance.Send...`. That sounds painful if like me you have literally 1000s of references, but ReSharper's extremely handy [structural search and replace](https://www.jetbrains.com/help/resharper/Navigation_and_Search__Structural_Search_and_Replace.html) allows to do this pretty quickly. 


## Change to access via interface
Now we can make consumers of `Repository`'s API access it via an interface:

```csharp
public interface IRepository 
{
    // just pull all public members from Repository here
}

public class Repository 
{
    public void SendTextNotification(Guid sender, Guid[] recipients, string text) {...}
    public void SendTextNotification(Guid[] recipients, string text) 
        => SendTextNotification(this.CurrentUserId, recipients, text);
    public void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state) {...}

    public static IRepository Instance {get;}= new Repository();
}
```
Note how the `Instance` property now returns the interface instead of the concrete type.

## Create new type
So far so good, but it would still be an enormous drag to test changes to the `Send*` methods. We have to separate somehow the sending functionality from `Repository` into a new type. 

To do this, we create a new type `Sender` and copy all related methods from `Repository` there. 

It may seem tempting to move the methods right away, but trust me, it's not a good idea. In the many years I have been doing such refactorings, in at least 5 out of 10 times moving from the start eventually led to problems and forced me to revert and re-start the process with copying, thus losing more time than if I'd copied from the start.

So our `Repository` stays unchanged for now and our new `Sender` looks like this:
```csharp
public class Sender 
{
    public void SendTextNotification(Guid sender, Guid[] recipients, string text) {...}
    public void SendTextNotification(Guid[] recipients, string text) 
        => SendTextNotification(this.CurrentUserId, recipients, text);
    public void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state) {...}
}
```

Unfortunately, it turns out `Sender` won't compile because it depends on `CurrentUserId`. 

You would have to be rather lucky to not encounter such a problem. Private members that are *only used by the `Send*` methods* are no problem, you can just copy them, too. But members used by `Send*` methods *and* other members of `Repository` (or even exposed in `IRepository`) are more complicated. For the sake of this article, I will assume just one such member. If there are more, up to a certain limit, you can still go the same route. Beyond 4-5, though, you probably will have to group the methods you want to extract into another type according to the dependencies they have on `Repository`.

## Deal with back-tracing dependencies
Now, because `Repository` is a singleton, we could just replace the reference `this.CurrentUserId` with `Repository.CurrentUserId`. However, this would defeat the purpose of the whole exercise, which is to carve out a *clean island of code* from `Repository`.

Instead, we extract another interface from `IRepository` (if `CurrentUserId` was not part of `IRepository`, but only, say, a private member of `Repository`, you would have to make it `public` and extract the new interface from `Repository` instead):

```csharp
public interface ICurrentUserHolder
{
    Guid CurrentUserId {get;}
}

public interface IRepository : ICurrentUserHolder 
{
    // ...
}
```

You might object that `ICurrentUserHolder` is a bad name and poor naming often is a symptom of bad type composition. And probably you would be right: having a type to hold the id of the current user is poor design. However, this is a hopefully temporary workaround to get us out of *much worse* design. And in the longer run, either the interface will turn into a more reasonable form, with other members and a better name, or maybe the need for it will disappear altogether by changing the contract to always require consumers of the `Send*` APIs to supply the sender. 

Now we only need to adjust our newly minted `Sender` type to get this injected:

```csharp
public class Sender 
{
    readonly ICurrentUserHolder _currentUserHolder;
    public Sender(ICurrentUserHolder currentUserHolder)
        => _currentUserHolder= currentUserHolder;

    public void SendTextNotification(Guid sender, Guid[] recipients, string text) {...}
    public void SendTextNotification(Guid[] recipients, string text) 
        => SendTextNotification(_currentUserHolder_.CurrentUserId, recipients, text);
    public void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state) {...}
}
```

Et voilà, it compiles. 

## Test-cover and perform desired functionality changes
This is now the time to bring it under test and perform any desired changes in behavior. 


## Make Production code use Sender
We still have a few issues to solve:
* `Sender` is not yet being used by any production code, in place of the old corresponding methods of `Repository`
* `Sender` effectively depends on `Repository` and we need to solve construction of `Sender` instances at runtime
* we still haven't fulfilled my promise from the outset of the article, to introduce an island of code using a container

So next we extract an interface from `Sender`, mirroring its public interface:
```csharp
public interface ISender 
{
    void SendTextNotification(Guid sender, Guid[] recipients, string text);
    void SendTextNotification(Guid[] recipients, string text);
    void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state);
}

public class Sender : ISender
{
    readonly ICurrentUserHolder _currentUserHolder;
    public Sender(ICurrentUserHolder currentUserHolder)
        => _currentUserHolder= currentUserHolder;

    public void SendTextNotification(Guid sender, Guid[] recipients, string text) {...}
    public void SendTextNotification(Guid[] recipients, string text) 
        => SendTextNotification(_currentUserHolder_.CurrentUserId, recipients, text);
    public void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state) {...}
}
```

Then we inject this interface into our `Repository` and make it use it:

```csharp
public class Repository 
{
    public ISender Sender {get;}            // note we use a public property

    public Repository(ISender sender)
        => Sender= sender;
    public void SendTextNotification(Guid sender, Guid[] recipients, string text) 
        => Sender.SendTextNotification(sender, recipients, text);
    public void SendTextNotification(Guid[] recipients, string text) 
        => Sender.SendTextNotification(recipients, text);
    public void SendMouseNotification(Guid sender, Guid[] recipients, MouseState state)
        => Sender.SendMouseNotification(sender, recipients, text);

    public static IRepository Instance {get;}= new Repository(); // compile time error 
}

public interface IRepository 
{
    ISender Sender {get;}                   // we need to pull this here, too
    // ...
}
```

There is a problem now, we can't instantiate our singleton anymore. This problem will go away in a minute. We really don't want our `Repository` to include all those `Send*` methods, after all that's part of why we created `ISender` in the first place. We want our callers to use `ISender` directly instead.

The solution for both issues is really the same:

First, we create a new static type called `Globals` and move the `.Instance` property from `Repository` there, renaming it simultaneously:

```csharp
public static class Globals
{
    public static IRepository Repository {get;}= new Repository(); // compile-time error 
}
```

You can do this automatically with R#, updating all references correctly. 

Next we inline all calls to any `Send*` method of `Repository` (again, using R#). This step is why we used a public property `Repository.Sender` before. Had we used a private field, this step now would lead to lots of compile-time errors.

After this, calls will look like:

```csharp
Globals.Repository.Sender.SendTextNotification(sender, recipients, text);
```

`Repository` and `IRepository` no longer need the `Send*` methods.

```csharp
public class Repository 
{
    // there is still a HUGE list of private fields here, but none 
    // related to sending notifications anymore

    public ISender Sender {get;}

    public Repository(ISender sender)
        => Sender= sender;

    // Repository now contains only the methods related to document upload, download
    // internal graph manipulation, caching, application versioning 
    // and updating and so on
}
```

Next we add another property to `Globals`:

```csharp
public static class Globals
{
    public static IRepository Repository {get;}= new Repository(); // compile time error 
    public static ISender Sender {get;}= new Sender(); // compile-time error
}
```
Don't worry about the compile-time errors, they are going to go away shortly.


Now we turn again to our friend [structural search and replace](https://www.jetbrains.com/help/resharper/Navigation_and_Search__Structural_Search_and_Replace.html) and replace all references to the `Send*` methods like

```csharp
Globals.Repository.Sender.SendTextNotification(sender, recipients, text);
```

with

```csharp
Globals.Sender.SendTextNotification(sender, recipients, text);
```

and, at long last, we can remove the `Sender` property from `Repository` and `IRepository` again, and `Repository` thus has got a default constructor again.

With this we have solved the first of the problems mentioned further up, production code everywhere now uses our clean, well-tested `Sender`, and `Repository` knows nothing about sending anymore.


## Introduce IOC container
Finally, we can introduce a container, solving the remaining two issues.

First, we create an Autofac module:

```csharp
public class IslandModule : Module
{
    protected override void Load(ContainerBuilder builder)
    {
        builder.RegisterType<Repository>().AsSelf().AsImplementedInterfaces();
        builder.RegisterType<Sender>().AsSelf().AsImplementedInterfaces();
    }
}
```

Then we adjust `Globals`:

```csharp
public static class Globals
{
    static readonly Lazy<Globals> _instance = new Lazy<Globals>();
    readonly IContainer _container

    public Globals()
    {
        var builder = new ContainerBuilder();
        builder.RegisterModule(new IslandModule());
        _container = builder.Build();
    }

    static T Resolve<T>() => _instance.Value._container.Resolve<T>()

    public static IRepository Repository => Resolve<IRepository>();

    public static ISender Sender => Resolve<ISender>();
}
```

Note how `Globals` now internally uses a container to resolve those types migrated to it. 

Over time, we can add more and more types as we break them out from the god-object `Repository` (or elsewhere); for new types, we can work under the assumption of a container right away; and eventually, in the future, when we will have refactored all types to use IOC, we will be able to eliminate `Globals` itself, too.

## A few more questions
I'll end this article with a few remarks in question-and-answer style.

### Why call the holder of the singletons `Globals`?

Because it keeps everyone using it honest and serves as a reminder what is really happening when we access static members or instances from other types. 

### Why is `_instance` in `Globals` modelled as `Lazy<>`?

Because `Lazy<>`'s value generation is intrinsically thread-safe and race-conditions do not lead to creation of multiple instances. This really is the kind of problem you wouldn't ever have if you used a container from the beginning, but as we are talking about a migration strategy from legacy code here, we must ensure thread-safety of singletons. 

### What to do when mutual dependencies between extracted types and original legacy-types are not easily avoidable?

For example, what if `Repository` would need to call some `ISender` methods? Then we'd have to inject `ISender` into `Repository` and 
thus create a circular dependency between `Repository` and `Sender`. 

The purist answer that this is a sign of poor design does not help in this context because this is really something that happens often until refactoring of the legacy types has progressed far enough - in practice this usually means many months because we don't get to refactor everything at once. (And even if we did, it wouldn't be wise to do it without intermediate steps, but that's a subject for another article.)

When I encounter such a situation and there is no easy disentangling of responsibilities and thus dependencies that would not require a bigger refactoring effort than I can justify for my current scope, then I use `Lazy<>` or `Func<>` injections. For example:

```csharp
public interface IRepository : ICurrentUserHolder
{
    // ...
}

public class Repository : IRepository
{
    readonly Lazy<ISender> _sender;
    public Repository(Lazy<ISender> sender)
        => _sender= sender;
    
    ISender Sender => _sender.Value;

    void SomeMethodDependingOnISender()
    {
        Sender.SendTextNotification(...);
    }
}

public class Sender : ISender 
{
    readonly ICurrentUserHolder _currentUserHolder;

    public Sender(ICurrentUserHolder currentUserHolder)
        => _currentUserHolder= currentUserHolder;
    
    // ...
}
```

This solves the circular dependency by changing the time when one of the two instances has to be created: instead of requiring an `ISender` instance at construction time of `Repository`, it is needed only once `Repository` accesses it for the first time.

Whether to use `Lazy<>` or `Func<>` depends usually on whether or not the instance can change. If the instance is going to be created only once and effectively be a singleton for the duration of runtime, then `Lazy<>` is the better choice. If the instance can change, then `Lazy<>` would introduce bugs and `Func<>` is more correct.

On a side note, that both `Func<T>` and `Lazy<T>` are resolved with no additional work, for any registered `T`, is one of many reasons I like Autofac so much as a container. 


