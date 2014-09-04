---
layout: post
title: Gemfire Server for Test Environments
---

If your application is integrating with a gemfire cache, you may have chosen to stub out this boundary when running beahviour, acceptance or system tests. Intially we were using a fake object (<a href="http://www.martinfowler.com/bliki/TestDouble.html" target="_blank">see test doubles</a>) in our test to avoid interaction with a gemfire cache, however we agreed that the data being returned from the fake object was not visible from our behaviour tests. It went something like this, although I will use a Grocery store as an example instead of our actual domain.

{% highlight java linenos %}
public class FakeGroceryDetailsService implements GroceryDetailsService {
	private Map<String, GroceryDetails> groceryDetails;

	{
		groceryDetails = new HashMap<String, GroceryDetails>();
		groceryDetails.put("101", new GroceryDetails("Apple", "0.30"));
		groceryDetails.put("102", new GroceryDetails("Banana", "0.15"));
	}

	@Override
	public GroceryDetails getGroceryDetailsUsing(String groceryId) {
		return groceryDetails.get(groceryId);
	}
}
{% endhighlight %}

There were a few issues with this:

- Anyone writing new tests needed to have knowledge of this class
- This class had to live in the main source directory, and as such is part of your production code
- A mechanism to switch between this fake object and the real implementation is required

As such, we decided to build our own gemfire cache server.

> This can be done by creating a cache.xml file and wrapping the regions with a server tag, however I prefer to keep as much as I can in Java code.

### Starting Cache Server
- importance of distributed 0

### Adding Regions
- CacheItems and Serializable, getId()

### Point to Server
- clientCache.xml points to server instead of locator
