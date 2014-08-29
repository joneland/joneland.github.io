---
layout: post
title: Gemfire Server for Test Environments
---

If your application is integrating with a gemfire cache, you may have chosen to stub out this boundary when running beahviour, acceptance or system tests. Intially we were using a fake object (<a href="http://www.martinfowler.com/bliki/TestDouble.html" target="_blank">see test doubles</a>) to avoid interaction with a gemfire cache, however we agreed that the data being returned from the fake object was not visible from our behaviour tests. It went something like this, although I will use a Grocery store as an example instead of our actual domain.

{% highlight java linenos %}
public class FakeGroceryDetailsService implements GroceryDetailsService {
	private Map<String, GroceryDetails> groceryDetails;

	{
		groceryDetails = new HashMap<String, GroceryDetails>();
		groceryDetails.put("101", new GroceryDetails("Apple", "£0.30"));
		groceryDetails.put("102", new GroceryDetails("Banana", "£0.15"));
	}

	@Override
	public GroceryDetails getGroceryDetailsUsing(String groceryId) {
		return groceryDetails.get(groceryId);
	}
}
{% endhighlight %}

We wanted to be able to control the data from our tests so that it the test scenario was clear to anyone read them. As such, we decided to start, populate, and stop our own gemfire cache from the tests. Here is a short guide in setting up a cache server to allow your tests to exercise the code in your application that interacts with a gemfire cache.

_Just as a side note, this can be done by creating a cache.xml file, similar to a clientCache.xml, however I prefer to keep as much as I can in Java code and out of XML._

### Starting Cache Server
- importance of distributed 0

### Adding Regions
- CacheItems and Serializable, getId()

### Point to Server
- clientCache.xml points to server instead of locator
