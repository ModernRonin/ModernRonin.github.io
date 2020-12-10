---
title: PlantUml
date: 2020-11-05 22:18:03
tags:
  - uml
  - diagramming
---

Lately, I've started to use [PlantUML](https://plantuml.com). 

If you are like me and prefer keyboard over mouse and want to be able to properly version-control your UML diagrams, then you should definitely check this out.

The easiest way to use it is to start a local server with 
```sh
docker run -d -p 8080:8080 plantuml/plantuml-server:tomcat
```

and then use VS Code with [this extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml) (don't forget to set the local server in the extensions settings).


