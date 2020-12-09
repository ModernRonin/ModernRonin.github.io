---
title: Azure Functions and the CosmosDB change feed
date: 2020-08-10 14:30:09
tags: 
  - azure
  - azure functions
  - cosmosdb
---

If you ever work with an Azure Function that is connected to/triggered by a CosmosDb change feed:

Whenever you want to reset the CosmosDB container/collection during development, **you need to stop the function first, then reset the container, then start it**. 

If the function is running when you are resetting the container, the function will no longer get any change updates, but won't actually tell you that via logs.

Somewhat related, you **must NOT share the same storage account between multiple function apps*. While the docs state that you generally can do this and while this is usually true, if your functions are triggered by a cosmosdb change feed, then each function app must have its own storage account, contrary to the documentation. Otherwise, you'll get all kinds of errors regarding acquiring leases and only some of your functions will be triggered.