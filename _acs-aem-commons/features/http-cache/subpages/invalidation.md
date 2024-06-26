---
layout: acs-aem-commons_subpage
title: Http Cache - Invalidation
---

[<< back to HTTP Cache Table of Contents](../index.html)

<iframe width="560" height="315" src="https://www.youtube.com/embed/DC1-DGY-uk4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Preventing stale content is very important. In the HTTP cache, it is the responsibility of the developer to do this, as the HTTP cache is so flexible and extensible, it is impossible for it to know when to invalidate itself. That is why the developer must instruct the HTTP cache when to flush the content.
However, there are some OOTB helper classes out there, next to the HttpCacheEngine and HttpCacheStore services which can be referenced to invalidate the cache as well.

HttpCacheInvalidationJobConsumer will invalidate a path (and references  of the path if configured) by consuming a sling job with the topic `CacheInvalidationJobConstants.TOPIC_HTTP_CACHE_INVALIDATION_JOB`. 
So in theory you can write your own component, that will based on your business logic needs create a sling job with this topic, and the path (key `CacheInvalidationJobConstants.PAYLOAD_KEY_DATA_CHANGE_PATH`) that needs to be invalidated.

OOTB we provided the JCRNodeChangeEventHandler below. You can leverage this or use this as example to create your own variant.
Custom cache invalidation events could be sling event handlers for JCR specific events, sling filter to trap replication events, workflow steps, etc. 

#### Configuring JCR Node change invalidator (JCRNodeChangeEventHandler) Since v2.5.0/3.14.0

Sample cache invalidation job creator. Watches for changes in JCR nodes and creates sling jobs for cache invalidation.  

Define a `sling:OsgiConfig` `/apps/mysite/config/com.adobe.acs.commons.httpcache.invalidator.event.JCRNodeChangeEventHandler.xml`

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="sling:OsgiConfig"
    event.filter="(|(path=/content*)(path=/etc*))"
 />
{% endhighlight %}

- `event.filter` JCR paths to be watched for changes. Expressed in LDAP syntax.

#### Enabling Reference-based invalidations (HttpCacheInvalidationJobConsumer). Since v2.5.0/3.1.0

Invalidate resources that reference the change resource.

Define a `sling:OsgiConfig` `/apps/mysite/config/com.adobe.acs.commons.httpcache.invalidator.HttpCacheInvalidationJobConsumer.xml`

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0" xmlns:cq="http://www.day.com/jcr/cq/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
    jcr:primaryType="sling:OsgiConfig" httpcache.config.invalidation.references="{Boolean}true" />
{% endhighlight %}

#### Custom invalidation

The HttpCacheEngine service has 2 methods for cache invalidation.

1: isPathPotentialToInvalidate(String path).
This executes the regex statement from all cache config's `httpcache.config.invalidation.oak.paths` values, and returns if there is at least one match.
This is good to narrow down a filter and save performance in your service / servlet / workflow.

2: invalidateCache(String keyString) 

What happens:

* CacheKey with the String constructor get's constructed using buildCacheKey(String resourcePath) from the factory.
* Calls isInvalidatedBy on the entries' CacheKey objects in ALL stores.
* The AbstractCacheKey provides an implementation for isInvalidatedBy that checks if the resourcePath is equal. 
* This makes sense because if the underlying resource changes, you generally want to invalidate.
* If true, it will clear the corresponding entry from the store

Note: Be careful overriding isInvalidatedBy, this can lead to the OOTB `HttpCacheInvalidationJobConsumer` not working.
