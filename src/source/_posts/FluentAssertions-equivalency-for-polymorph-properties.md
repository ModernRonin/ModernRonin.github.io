---
title: FluentAssertions equivalency for polymorph properties
date: 2021-01-14 20:40:09
tags:
  - FluentAssertions
  - TDD
  - polymorph
  - equivalency
---

While I couldn't imagine a life without the excellent [FluentAssertions](https://fluentassertions.com/) library, there are a few bits I find rather counter-intuitive. 

The other day I stumbled across one of them once again and thought this actually makes for a good blog post because quite likely it will bite you in the ass, too, eventually, dear reader, or maybe already even has **without you noticing**.

Take the following data-structures (simplified from my actual use case, for the sake of clarity):

```csharp
public interface IAction
{
    string TargetId { get; }
}
public class RenameAction : IAction
{
    public string TargetId { get; set; }
    public string NewName { get; set; }
}
public class DeleteAction : IAction
{
    public string TargetId { get; set; }
}
public class Command
{
    public IList<IAction> Actions { get; set; }
}
```

and now let's write a simple test using FluentAssertions:

```csharp
var input = new Command
{
    Actions = new List<IAction>
    {
        new RenameAction
        {
            TargetId = "alpha",
            NewName = "changed"
        },
        new DeleteAction {TargetId = "bravo"}
    }

input.Should()
    .BeEquivalentTo(new Command
    {
        Actions = new List<IAction>
        {
            new RenameAction {TargetId = "alpha"},
            new RenameAction {TargetId = "bravo"}
        }
    });
```

Does this test pass or fail?

If you are like me, you expect it to fail because after all `input.Actions` contains one `RenameAction` and one `DeleteAction` while the comparand for `.BeEquivalentTo()` contains two `RenameAction`s.

Alas, this test actually passes. 

The reason for this is that **FluentAssertions by default does not care about the runtime types when doing an equivalency comparison**. This allows you, for example, to compare a returned interface against a concrete type, or comparisons against anonymous types.

There is a remedy. You just have to add a configuration parameter to the equivalency check:

```csharp
var input = new Command
{
    Actions = new List<IAction>
    {
        new RenameAction
        {
            TargetId = "alpha",
            NewName = "changed"
        },
        new DeleteAction {TargetId = "bravo"}
    }

input.Should()
    .NotBeEquivalentTo(
        new Command
        {
            Actions = new List<IAction>
            {
                new RenameAction {TargetId = "alpha"},
                new RenameAction {TargetId = "bravo"}
            }
        }, cfg => cfg.RespectingRuntimeTypes());
```

Now the test will fail.

Personally, I find this extremely counter-intuitive - so much so that I know I have come already multiple times upon this, but each time I do it baffles me again for a second or two.

I would find it much more intuitive if the default behavior was to check the runtime types, and you'd have to actively turn off this behavior. Why? Because that way the likelihood of writing a test that passes where it should actually fail would be smaller.

Of course, as long as we are doing proper TDD, this cannot happen because we are writing the test before the production code, so we notice the discrepancy immediately. This is actually how I noticed it this time around. However, reality is not always so kind, and often we have to bring code under test that has already been implemented. In these scenarios, the default behavior of FluentAssertions violates the [Principle of least surprise](https://wiki.c2.com/?PrincipleOfLeastAstonishment).