# Be a good Consul client

This is the third article of our series on Consul and service discovery at Criteo. We will now have a look at the client-side.

Consul servers may be distributed but their numbers are limited (we use a cluster of 5 per datacenter). And if you are not careful, Consul requests may all be forwarded to the same server. On the other hand, you lose a lot of Consul's benefits if you restrict too much the calls to Consul.

At Criteo, we have around 4000 services registered in Consul, running in 12 data centers for a total of 280 thousand instances. The behavior of Consul clients is hence really important to scale.

## A brief history

Most of our applications are .Net or JVM. When we migrated to Consul, we already had SDKs for C# and Java. We relied on an in-house discovery system based on static registration in a database. And we provided full mesh between services thanks to client-side load balancing libraries. The load balancing layer was taking care of doing health checks. It was also retrieving the server’s weight to implement a weighted round-robin algorithm.

Our first step was to replace our discovery system by Consul by relying on [Catalog endpoint](https://www.consul.io/api/catalog.html). Once we added weight support to Consul (see [Pull Request](https://github.com/hashicorp/consul/pull/4468)), we were able to migrate to the [Health endpoint](https://www.consul.io/api/health.html). This allowed us to let Consul take care of the health checks.

## Our Consul clients

We handle Consul services registration – and any other Consul operation relying on write requests – on DevOps side (deployment tools, etc.). This means that our Consul clients only perform read-only requests. Writing a Consul client is thus just a matter of calling the right HTTP API and parse its JSON response.

The main point is to find the correct balance between the number of requests and reactivity to change. During our Consul journey, we had to carefully monitor the number of call to Consul servers. A few times we reached the maximum load of Consul servers (and even crossed it – see the previous article). In such a case, it is better to rate limit on client-side rather than bringing down Consul. But with the correct configuration – and a few Pull Request to Consul project in the process – we were always able to find a solution.

We also spent a bit of time crafting error messages and diagnostic tools. As Consul was more and more used as the backbone of our infrastructure, you need self-explanatory error messages and UI if you don't want people coming for you each time a service is down.

## Details that matter

As explained in the previous article, the first point is that we rely heavily on [blocking queries](https://www.consul.io/api/features/blocking.html) (long polling). Clients are notified immediately when a service changes. And when there is no change, the number of requests sent to the servers is minimal. It is important to perform some throttling. Otherwise, a service being continuously updated (because its health checks are flapping for example) can trigger a crazy number of requests from all its watchers. We followed Hashicorp recommendation and used a [token bucket algorithm](https://en.wikipedia.org/wiki/Token_bucket).

We make sure that requests are made with [consistency parameter](https://www.consul.io/api/features/consistency.html) set to stale. Consul consensus protocol is Raft, which is based on a leader/follower model. This consistency option prevents the Consul servers to forward all the requests to the leader and ensures a better availability. It doesn’t mean we are serving stale data: we are closely monitoring the replication between Consul servers. We went further and added an option to Consul to change the [default consistency behavior](https://www.consul.io/docs/agent/options.html#discovery_max_stale) (see [Pull Request](https://github.com/hashicorp/consul/pull/3920)) and force it to stale. It is helpful for projects not relying on managed SDKs.

Another useful option is [agent caching](https://www.consul.io/api/features/caching.html). It allows the local Consul agent to answer a request without relying on the Consul servers if a similar request was made in the near past. This helps when many applications are running on the same server. An important use case is of course containerized applications. Applications often have some common dependencies (e.g. metric service, databases, etc.). We also have an application that runs multiple instances per server (mostly IIS) and the gain is even bigger.

## Next steps

We are pleased with our current Consul integration. We are still working on adding new features and a big topic is Consul Connect which will the subject of a dedicated article.

Besides bringing new features, it will also provide natively some functionalities currently implemented in each client (the main one being load balancing). This will lead to thinner SDKs and make it even more easy to develop a client for a new language.