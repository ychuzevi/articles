# Discovery with Consul at scale

This article is the first one talking about our usage around Consul and the work in the Discovery team, a team in charge of running Consul, integrating all parts of Criteo in a streamlined way and provide the SDKs for all applications to properly talk to each other's.

In those articles, we will talk about the reasons of moving to Consul, the various issues we found using it, the patches we issued (more than 60 upstream contributions), the way we design our SDKs and the future directions we will embrace.

## Why moving to Consul?

More than three years ago, Criteo did choose Consul to discover services into our pretty large infrastructure.

Criteo used to rely on various systems to discover the services within the infrastructure: Chef queries (Chef is an automation system to deploy infrastructure), an in-house database-based system, and various other systems (Zookeeper, DNS, hardcoded paths).

All of those heterogenous systems shared a few drawbacks: latency, huge implementation costs (lots of libraries) and difficulty to communicate with each other. Consistency of load balancing systems was also an issue since many systems did declare several health checks in order to guess whether a machine/service was up/down.

Since Criteo was running for a long time with those systems, why moving to a new system?

Criteo has been historically using large applications on bare-metal servers, without any kind of virtualization.

But at that time, Criteo was starting to use Mesos in order to launch containerized apps on common hardware. It means that our deployment infrastructure (Chef) which was used to provision Mesos machines no longer had any knowledge about applications running on those machines.

The advantage of virtualization of infrastructure is to provide a way to quickly re-use hardware to run various workloads. This is pretty cool because it brings the following benefits:

 - ability to run different workloads at the same time on a machine, think about an application consuming 75% of memory while using 10% of CPU and another using 10% of memory and using 60% of CPU, those 2 apps might be collocated on a single hardware and run smoothly on a single machine.
 It has always been hard to do so historically because several applications tend to "pollute" filesystems and/or take the whole resources of the machine when something goes bad. But with the democratization of containerization, it is now possible on modern operating systems to provide some real isolation for most resources.
 - less hardware needed to provide the same level of resilience: when your machines are specialized, each of those machine needs spare hardware in case of failure, so when you have 30 services, you need at least (30 machines ready to take the place of a faulty one, but having non-dedicated hardware can let you only have a few machines in spare hardware and whenever one of those machine dies, replace it with one ready in the common pool)
 - ability to switch tasks during the day: a machine can be used during the daylight to perform some tasks, while being used to compute its results during the night.
 - less combinations of OS configurations to maintain for people in charge of infrastructure
 - less time to go to market when a new product is being launched since you can use free machines from the global pool

The list goes on, but that's some of the main reasons people tend to prefer cloud-based or containerization of applications. All of this goes with some additional complexity however: applications can move very fast from a machine to another, change their IP, their port, so traditional tools such as DNS and databases are hard to scale efficiently to allow those kinds of new workloads.

We will not detail all the features of Consul, but this software is basically a strongly consistent distributed KV, having features similar to products such as ETCD or Zookeeper, but a part of KV dedicated to Discovery oriented towards services. It also comes with built-in feature to detect failing nodes (which is a pretty hard task in a distributed environment) and distribute this information across the whole cluster.

Consul bring some very cool features:
 - no specific requirements in terms of network (you don't need to run a complicated Software Defined Network system), so all systems can talk with each other's regardles of the fact that those systems are running in the Cloud, in containers or in traditional bare-metal machines
 - service oriented: the machine is just a detail, everything is service centric, we don't care about being in a virtualized system, a real machine, being alone on the machine or sharing the machines with dozens of other services
 - ease of administration because it only requires another node to find all the nodes of the cluster: you don't need to provide a full list of systems, it can join "magically" other Consul enabled systems.
 - ability to detect failure of services using several health checks, so the app does not require to implement its own detection mechanism to detect failure (and human that do program systems have issues detecting and properly reacting to all possible failures)
 - ability of being notified almost immediately when services do change and to provide tools to easily generate mutable configurations using tools like consul-template
 - nice DNS gateway, which allow to quickly allow any app to interact with other Consul-declared systems whenever this app is Consul-aware.

## From nothing to Universal Discovery

For all those reasons, Consul was chosen to modify deeply how our high number of services discover themselves within our large infrastructures. The move was quite progressive, but still done at very high pace: since our infrastructure was mainly provisioned with chef, many services could be added automatically to Consul very quickly by adding only a few lines of code into our deployment code.

Once the services were properly added in Consul, with health checks, we started using the discovery mechanisms:
 - DNS for basic systems or infrastructure elements not-Consul ready
 - Consul's HTTP API for most of the business applications by modifying our SDKs which already used some kind of abstraction to find other services.

Since Criteo systems are highly automated for the most parts, the move was done quite fast (in a few weeks), but the real move was to consume all those services from the apps, meaning that application do target the correct services. Criteo had been using internal SDKs to find target of the microservices within the infrastructure for years, but moving paradigms still takes time.

## Mesos to Consul

At the same time, we worked on mechanisms to declare automatically applications running on Mesos, so those applications could be declared in Mesos frameworks (we mainly use Marathon as a framework for Mesos) and automatically registered into Consul at the same time with semantics similar to what had been provisioned in the other parts of infrastructure. It then becomes possible to run existing services in both bare-metal and within Mesos. Consul then hides the details and clients can find their target services running in mixed pools (50% bare-metal, 50% Mesos for instance).

Bringing mixed workloads brings its own issues however: a given service can now run either on a bare-metal server running all the CPUs of a machine and at the same time on smaller instances having less resources. It also means that the instances of a given service can have very different characteristics, whether the instance is running Windows, Linux, is deployed into a container or as a regular service. The need to describe those differences lead us to implement service metadata that could be retrieved by all the clients to take more appropriate decisions to target a given service.

This mechanism was historically set-up using weights in our load-balancers. Those weights would then result is more or less requests being sent on a given machine given its generation (recent servers used to be more performant, so more weight was allocated according to them). Consul did not provide built-in mechanism to do so (some hacks were relying on tags, but no real standardization was present), so we decided to implement our own with a new added feature: the ability to automatically adjust the weight depending of the state of a given service.

## Load-Balancer provisioning

Criteo is historically using two kinds of load-balancing in its architecture:

 - client-side load balancing (aka Service Mesh): in this mode, used only within its datacenters, the application uses an internal library (similar to systems such as Finatra) to find the nodes running their target service, and then target those instances of the service directly using HTTP. The benefits being that latency is reduced and it does not need any kind of server in the middle to route the traffic. With the number of instances of applications. We thus adapted those SDKs to use Consul instead of our legacy systems. Now all applications use a feature called blocking requests that allow being notified when the instances of a service do change (their status or the nodes).

 - Traditional load-balancing has also been used for years, running F5 or HaProxy. We decided to use the same exact mechanism: watching all changes in Consul in order to trigger commands when nodes for a given service do change. By providing information into Consul (using tags), it also becomes possible to discover automatically the services that need to be added to our various load-balancers stacks.

 Those 2 ways of provisioning do rely on the same exact semantics within Consul, we now have a system where you can use one client side load-balancing or the other and have the *same exact results*.

 Load-balancing is a complete beast: sometimes, even if you use weights to route your traffic, some nodes will be slower than others. Whatever the reason (maintenance, scheduled task, Garbage Collection), some instances of a given service will be able to sustain less load, or to reply bad answers. Heathchecks have been using traditionally in our both stacks to ensure the node is answering correctly. Those health checks were using a binary state: working or not working. Consul also adds a third state called warning. This state can be interpreted freely by the end user. But in our added support for weights, we allowed service owners to add a specific weight for the state warning. It thus become possible to use a service in a degraded state (warning), so it means that if we can trigger this status when the node is too heavily loaded, we could use it as a ternary state to allow too loaded nodes to recover. The big advantage of having a non-binary state is that it avoid having a datacenter being completely down all nodes are too loaded: in that case, the first overloaded nodes will receive less traffic, redirecting some of the traffic to others, but if other are also overloaded, then traffic will be served anyway (at worse, all nodes are in warning state).

# Towards more advanced discovery and new patterns

In the next articles, we will go deeper onto those subjects and see how industrialization of discovery will help Criteo and might help you changing radically how systems can talk / discover themselves and how it can change the way organizations and teams work in an era of micro-services.

We will explain how:
 * we implemented libraries to integrate Consul in a streamlined way
 * how we introduce Service Mesh with Consul Connect
 * How we use what we call Inversion Of Control to create new patterns around infrastructure