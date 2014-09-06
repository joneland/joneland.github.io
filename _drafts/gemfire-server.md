---
layout: post
title: Gemfire Server for Test Environments
---

If your application is integrating with a gemfire cache, you may have chosen to stub out this boundary when running beahviour, acceptance or system tests. Intially we were using a fake object (<a href="http://www.martinfowler.com/bliki/TestDouble.html" target="_blank">see test doubles</a>) in our test to avoid interaction with a gemfire cache, however we were concerned that the data being returned from the fake object was not visible from our behaviour tests. It went something like this (although I will use a Grocery store as an example instead of our actual domain):

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

- Anyone writing new tests needed to have knowledge of the fake object
- The fake object had to live in the main source directory, and as such is part of your production code
- A mechanism is required to switch between the fake object and the real implementation

As such, we decided to build our own gemfire cache server.

> This can be achieved by creating a cache.xml file and wrapping the regions with a server tag, however I prefer to keep as much as I can in Java code.

### Starting Cache Server

Starting up a cache server is simple enough (and the things that are not so obvious are explained below).

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
			cache.createRegionFactory(LOCAL).create(region.name());
		}
	}

	public void start() throws IOException {
		cache.getCacheServers().get(0).start();
	}

}
{% endhighlight %}

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

These are added to the cache server using the region factory with LOCAL state. As the cache server will not talk across a network, changes to a region will not need to be reflected in peers.

#### Connecting to the Cache Server
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

### Populating Regions
With the cache server up and running, the only thing left to do is populate it. There are only two rules that must be followed when populating a region:

- The object that is added to the cache must implement Serializable
- The object must be retrievable by an ID

With that in mind, an interface is used to ensure both of those rules are covered.

{% highlight java linenos %}
public interface CacheItem extends Serializable {
	int getId();
}
{% endhighlight %}

Then within the cache server code, a populate method can be exposed to allow a class implementing CacheItem to be added to a region.

{% highlight java linenos %}
public void populate(CacheRegion region, CacheItem cacheItem) {
	cache.getRegion(region.name()).put(cacheItem.getId(), cacheItem);
}
{% endhighlight %}

For this grocery store cache example, it would make sense to populate the cache with a grocery. The override of toString is to allow the contents of a cache region to be asserted in a test.

{% highlight java linenos %}
public class Grocery implements CacheItem {
	private static final long serialVersionUID = -1145880717859373291L;

	private int id;
	private String name;
	private String price;

	public Grocery(int id, String name, String price) {
		this.id = id;
		this.name = name;
		this.price = price;
	}

	@Override
	public int getId() {
		return id;
	}

	@Override
	public String toString() {
		return format("id: %d, name: %s, price: %s", id, name, price);
	}
}
{% endhighlight %}

### Printing Items in a Region
Due to issues with external systems, we ended up using this cache in our smoke environment (our first integrated environment we deploy to). We found it useful to be able to view the contents of a cache region at any given time. Retrieving the contents of a region on the cache is also straight forward.

{% highlight java linenos %}
public String print(CacheRegion region) {
	StringBuilder cacheItems = new StringBuilder();
	for (Entry<Object, Object> cacheItem : cache.getRegion(region.name()).entrySet()) {
		cacheItems.append(cacheItem.getValue() + "\n");
	}
	return cacheItems.toString();
}
{% endhighlight %}

### Clearing a Region
As the region is essentially a map, clearing the region is performed by retrieving the region and calling clear.

{% highlight java linenos %}
public void clear(CacheRegion region) {
	cache.getRegion(region.name()).clear();
}
{% endhighlight %}

### Wrapping Up
An example of this cache server can be found at <a href="https://github.com/joneland/gemfire-sandbox" target="_blank">https://github.com/joneland/gemfire-sandbox</a>.

The example includes a runner and exposes some of the cache methods via JMX.

Hopefully this post demonstrates how simple it is to get a gemfire cache server up and running for test enrvironments.