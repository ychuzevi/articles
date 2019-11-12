# Be a good Consul client

Consul servers may be distributed but their numbers is limited (we use cluster of 5 per datacenter). And if you are not careful, Consul requests may all be forwarded to the cluster's leader. On the other hand, you loose a lot of Consul's benefits if you restrict too much the calls to Consul.

At Criteo, we have around 3500 services registered in Consul, running in 12 datacenters for a total of 270 thousands instances. The behavior of Consul clients is hence really important in order to scale.

## A brief history

Most of our applications are C# or Java. When we migrated to Consul, we already had SDKs for those 2 languages. We relied on an in-house discovery system based on static registration in a database. And we provided full mesh between services thanks to client side load balancing libraries. The load balancing layer was taking care of doing the health checks and retrieving servers' weight in order to implement a weighted round robin algorithm.

Our first step was to replace our discovery system by Consul by relying on [Catalog endpoint](https://www.consul.io/api/catalog.html). Once we added weight support to Consul (see [Pull Request](https://github.com/hashicorp/consul/pull/4468)), we were able to migrate to the [Health endpoint](https://www.consul.io/api/health.html) and let Consul takes care of the health checks.

## Our Consul clients

We handle Consul services registration – and any other Consul operation relying on write requests – on DevOps side (deployment tools, etc.). This means that our Consul clients only perform read-only requests. Writing a Consul client is therefore just a matter of calling the right HTTP API and to parse its JSON response.

The main point is to find the correct balance between number of requests and reactivity to change. During our Consul journey, we had to carefully monitor the number of call to Consul servers. A few times we reached the maximum load of Consul servers (and even crossed it – see previous article). In such case it is better to rate limit on client-side rather than bringing down Consul. But with the correct configuration – and a few Pull Request to Consul project in the process – we were always able to find a solution.

We also spent a bit of time carefully crafting error messages and other diagnostic tools. As Consul was more and more used as the backbone of our infrastructure, you need self-explanatory error messages and UI if you don't want people coming for you each time a service is down.

## Details that matters

As explained in the previous article, the first point is that we rely heavily on [blocking queries](https://www.consul.io/api/features/blocking.html) (long polling). This allows to be notified immediately when a service changes and otherwise to keep the number of request really low.

We also ensure that all our requests are made with [consistency parameter](https://www.consul.io/api/features/consistency.html) set to stale. This prevents the Consul servers to forward all the requests to the leader and ensure a better availability. To make sure that we are still serving fresh data, we are carefully monitoring the replication between the Consul servers. We actually went further and added an option to Consul [link = https://www.consul.io/docs/agent/options.html#discovery_max_stale].to also be able to change the default consistency behavior in order to have the same for projects not relying on SDKs (see [PR](https://github.com/hashicorp/consul/pull/3920)).

Another useful configuration is that we enable [agent caching](https://www.consul.io/api/features/caching.html). This allows the local Consul agent to directly responds to a request without relying on the Consul servers if a similar request was made in the near past. This is especially useful for our applications that have multiple instances running on the same servers.

