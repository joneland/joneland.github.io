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

> This can be achieved by creating a cache.xml file and wrapping the regions with a server tag, however I prefer to keep as much as I can in Java code.

### Starting Cache Server

Starting up a cache server is simple enough.

{% highlight java linenos %}
public class GroceryStoreCache {
	private static final String DISABLE_MULTICAST = "0";

	private Cache cache;

	public GroceryStoreCache(int serverPort, CacheRegion... regions) {
		cache = new CacheFactory().set("mcast-port", DISABLE_MULTICAST).create();
		configureCacheServer(serverPort);
		configureRegions(regions);
	}

	private void configureCacheServer(int serverPort) {
		CacheServer cacheServer = cache.addCacheServer();
		cacheServer.setPort(serverPort);
	}

	private void configureRegions(CacheRegion... regions) {
		for (CacheRegion region : regions) {
			cache.createRegionFactory(RegionShortcut.LOCAL).create(region.name());
		}
	}

	public void start() throws IOException {
		cache.getCacheServers().get(0).start();
	}

}
{% endhighlight %}

There are a couple things to note in the code snippet above.

#### Multicast Port
The multicast port is set to zero to prevent the cache server discovering other gemfire cache instances running on a network. With the gemfire default license, there is a restriction in place that prevents more than three instances of a cache being started using the same multicast port. By setting to zero, the cache will not conflict with any other gemfire cache servers you have running on your build server.

#### Adding Regions
Regions are simply strings so for this example CacheRegion is just an enum.

{% highlight java linenos %}
public enum CacheRegion {
	EMPLOYEES,
	GROCERIES
}
{% endhighlight %}

These are added to the cache server using the region factory with LOCAL state. Local state is sufficient for this test environment as the cache server does not talk to other peers and as such any changes to regions do not need to be reflected across the network.

#### Pointing to Cache Server
There is one subtle difference when connecting to a cache server. Instead of using a locator, a server is used instead. As the majority of gemfire users will be using a client-cache.xml file to define pools and regions, I will include the XML changes required for the client.

Previously your client-cache.xml file may have contained a pool element like this:

{% highlight xml linenos %}
<pool name="client">
	<locator host="localhost" port="10000" />
</pool>
{% endhighlight %}

This just needs updating to be:

{% highlight xml linenos %}
<pool name="client">
	<server host="localhost" port="10000" />
</pool>
{% endhighlight %}

### Populating Region
- CacheItems and Serializable, getId()

### Point to Server
- clientCache.xml points to server instead of locator
