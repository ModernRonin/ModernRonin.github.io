---
title: MessagePack without attributes
date: 2021-06-13 12:31:59
tags:
  - messagepack
  - no attributes
  - binary serialization
---

A while ago I was looking into [MessagePack](https://github.com/neuecc/MessagePack-CSharp) to use for binary serialization. 

There were a huge number of domain types that used to be serialized with JSON.NET and also with protobuf. I needed to introduce binary serialization usable with SignalR and the only binary protocol SignalR supports is MessagePack. 

The reason for looking into binary serialization was to save traffic - Azure SignalR transparently splits your traffic into 2KB messages, but you pay per message. And in that project there are some situations where messages, in JSON, would count in the *multiple megabyte* range. 

After looking into this I found some good and some bad:

## The Good
MessagePack is really very good at generating very small serialization destillates and it's incredibly fast. It leaves even protobuf in the dust, and JSON is not even in the same ballpark. 

## The Bad
However, there is a catch. MessagePack offers three ways to use it: th default resolver, which I'll call `Fully Attributed`, `Contractless` and `Typeless`. 

`Fully Attributed` gives you the best performance, but it requires you to attribute every type and every single serializable property of your types and, if like in that project, you use polymorphy, every interface, too. 

To make things worse, the attributes you place to describe polymorphy are such that you need to specify all possible subtypes with attributes on the base-type, thus introducing a dependency violating one of the oldest principles of OOD.

`Contractless`is better in that regard, you save the attributes on the properties, but still have to attribute for polymorphy.

`Typeless` doesn't require any attributes at all. Well. Almost. If, like in that project, your types have private members then you explicitly need to ignore them via attributes because `Typeless` serializes private members and there is no way to turn this off, except via said attributes. 

There is another big problem with `Typeless`. Because `Typeless` basically serializes the full type info including assembly, a malicious sender could trick your deserializer into instantiating arbitrary types with arbitrary properties, thus basically executing code in the runtime context of your deserializer. For that project this didn't matter, though, because all serialization happens only on the server, clients just deserialize.

From a performance perspective, `Contractless` is a bit slower and generates about 40-50% bigger destillates than `Fully Attributed`. `Typeless` is yet slower and generates a LOT bigger destillates - depending on your type-model potentially even bigger than JSON.

Interestingly, using the regular .NET deflate on top of MessagePack generates better results than using MessagePack's built-in compression option, both in size and speed, at least for typical big examples of that project's data. 

## Outcome
While the attributing requirements of `Fully Attributed` and `Contractless` are, in my opinion a generally bad design decision, for that project they were completely inacceptable: it would have meant touching about a hundred types and adding another layer of noise to the already abundant mess of JSON and Protobuf attribute declarations. 

So I went with Typeless combined with Deflate.

This combination generated about the same size as JSON + Deflate, but about 13 times faster for serialization and 23 times faster for deserialization. 

The size gain compared to the uncompressed JSON from before was about factor 10-15, so the original goal of cutting costs was well-met, with the added benefit of a massive performance boost.

(Keep in mind that these numbers might look totally different for your domain model. If you ever need to make a similar decision, you will have to measure  performance with representative examples of your model.)


## A fourth way
Nevertheless, I felt dissatisfied. Just because of that ridiculous attribute mechanism we had not been able to reap the full benefit of size reduction. 

So I started to experiment a bit and in the end came up with a fourth way of using MessagePack. I call it `Attributeless`. 

This method requires zero attributes on your part, all configuration happens via a fluent builder. Yet the resulting destillates are only about 10% bigger than the what `Fully Attributed` achieves (compared to Typeless which generates 4-5 times bigger destillates).
The price you pay for this combination of OOD convenience and small destillate size is serialization/deserialization performance: `Attributeless` is 4-6 times slower than `Fully Attributed` and about 2 times slower than `Typeless`. It's still a hell of a lot faster than JSON.

You can look at detailed benchmark results at the project's [github repo](https://github.com/ModernRonin/MessagePack.Attributeless) where you will also find documentation.


## How to pick a serializer
It makes sense to visualize requirements as a three-dimensional coordinate system. One axis represents speed, another size and the last development convenience/adherence to OOD. Different serialization methods occupy different locations in this coordinate system.

`JSON` sits in the corner of maximum development convenience (and, if used properly, doesn't introduce any coupling violations), but very low performance both in speed and size.

`Fully Attributed` is the polar opposite: you get really extreme fast speed and very small size, but development convenience is basically non-existent. 

`Contractless` is really only worth any consideration if your models don't use polymorphy. Then it is as convenient as JSON at only slighly slower speeds and bigger sizes than Fully Attributed. 

`Typeless` cannot (or, more accurately: really, really should not) be used if you don't control the source of serialization, but if you do, you get similar convenience to JSON (but not quite) with speed between JSON and Fully Attributed, and sizes about the same as JSON.

`Attributeless` gives you the same convenience as JSON at almost the same small size as Fully Attributed, at a speed between Typeless and JSON.

I would have liked to create a 3d chart to visualize this, but didn't finy any quick way. If anyone has ideas, let me know :-)