---
layout: post
title: Gemfire Server for Test Environments
---

If your application is integrating with a gemfire cache, you may have chosen to stub out this boundary when running beahviour, acceptance or system tests. Intially we were using a fake object (<a href="http://www.martinfowler.com/bliki/TestDouble.html" target="_blank">see test doubles</a>), however we felt that this wasn't very clear from our behaviour tests. We wanted to be able to control the data from our tests so that is was obvious what was going on to anyone who read them. Here is a short guide in setting up a cache server to allow your tests to exercise the code in your application that interacts with a gemfire cache.

Just as a side note, this can be done by creating a cache.xml file, similar to a clientCache.xml, however I prefer to keep as much as I can in Java code and out of XML.


- configuring server
- importance of distributed 0
- CacheItems and Serializable, getId()
- clientCache.xml server instead of locator
