---
title: "Playing with gossip protocols"
date: 2020-01-20T09:11:16+01:00
draft: false
---

For some reason, I'm a big fan of [gossip protocols](https://en.wikipedia.org/wiki/Gossip_protocol). I really don't know why but I think that the idea is great. I guess it still surprises me that they work at all. 

Basically a Gossip Protocol (or Epidemic protocol) is a way of communicating messages and kind of spreading information in distributed systems. It basically mimics how rumors or diseases are spread in societies. 

My experience with them was rather simple though, I played with the concept doing some extremely easy half implementations in my desktop machine during my Universitiy years, but never went much farther. 

Recently I had to teach in a "Introduction to distributed systems" class, and I was looking for exercises, so the attendants could get dirty on some DistSys related implementation, and well, I thought that the implementation of a simple gossip protocol could be both a great learning experience and fun as well, and also, we are living in the future and we have `docker`!

So, of course I created [my own implementation](https://github.com/lant/gossip) to see how hard it would be. 

## How a gossip protocol works

I started without doing much research, just with the knowledge I kept from that University class from 15 years ago. I did that on purpose to see exactly what kind of problems I would find. This was a playground from the start so I wanted to face the same doubts and challenges that the students in the course would have. 

Gossip protocols can be designed in lots of different ways. First questions that popped up into my mind: 

* Are the nodes spreading information when they receive it or periodically ? 
* Are the nodes publishing the information when they receive it? or are the nodes asking for information to the other nodes? 
* Is there any way to know that all the nodes have reached a consensus?
* How does a node know that the information its receiving is current, and not some old rumor that keeps on spreading? 
* And a very very important one, how does a node in these kind of clusters knows their peers ? 

So, after I had this list of questions I did the sane thing to do, which was look for bibliography and resources about gossip protocols and I validated that all my doubts were valid ones, all of them being hot topics in gossip protocols research. 

One of the key questions was the first one, how to spread the information. There are two classifications of gossips, *anti entropy* protocols and *rumor mongering* protocols. 

### Classification by dissemination strategy

What's the difference between them? It's basically their termination and their node states. 

The *anti entropy* strategies are simpler. They are also called *Simple Epidemics* and they are designed to run forever. On the other hand the *rumor mongering* strategies, or *Complex Epidemics* are designed to run until the change has been propagated to all the nodes. 

Both strategies we have the following node states: 
  * **Susceptible**: A node has not received the information yet.
  * **Infected**: A node has received the information. 

And *rumor mongering*  strategies add a third state: 
  * **Removed**: When a node has been infected and it's not actively propagating more information.


#### Anti-entropy
As stated before, this strategy keeps propagating the data indefinitely, this is a good choice for a system that will see lots of updates. 

Let's remember that the nodes have just two states (not infected yet / infected). Propagation can work in different ways:  

  * **Push based**: Nodes push their state to the rest of the nodes. They can do that immediately when they receive a new state, or they can just do it periodically. 
  * **Pull based**: Nodes ask other peers for the latest state.
  * **Hybrid**: A mix of both would be to have a pull strategy where nodes ask for the latest state to their peers, and once they have it they propagate it actively to some peers.

A common part of all of these strategies is that nodes are always (periodically) pushing starte or pulling state.

#### Rumor mongering
The inclusion of the third state in these strategies basically means that the propagation of the information stops when all the nodes in the cluster get to this third state. 

There is quite a lot of work that demonstrates that this works fine, have a look at [this article by Alberto Montsersor for example](https://onlinelibrary.wiley.com/doi/abs/10.1002/047134608X.W8353) if you're interested.


## How my implementation works

[At the time of writing this post](https://github.com/lant/gossip/ "Remember, we don't trust clock times in distributed systems") the implementation of the gossip protocol works like this: 

**Gossip strategy**: The implementation follows an *anti-entropy* design with a push based model. This basically means that each node that have received a message will propagate it **immediately** and **periodically**. 

**Timestamped messages**: I decided that the most convenient thing to do was to bring the system up with a command and then be able to send messages with another command. This basically means that the data input is done manually. Given this situation and that all the nodes are running in a `docker-compose` environment having timestamps in the messages is good enough. Every time the user sends a message to be propagated into the cluster using any of the nodes, this node that receives the message from the user adds a timestmap, then it propagates this tuple. The rest of the nodes in the cluster decide which value is newer (the one they had or the new that they are receiving) based on this and acts accordingly. 

While this approach is terribly convenient it might not work in a real scenario. Clocks might be skewed, so the comparisions between the messages could be totally wrong. Not only this, if the data to be propagated into the cluster came from another system (much faster than normal human user interaction) it would be prone to racing conditions. 

I'm probably going to explore different approaches to get rid of the timestamp, like Lamport clocks for example. Even if I want this project to be a playground I want it to be as close as possible to a real system. 

## Unexpected side problems
I found some difficulties during the development of the system. Not related to coding exactly, something more "lack of better options" or "lacking of tooling". Here they are: 

### Visualization of the system
I wanted to visualize how the system was sending messages from one node to the other. Basically a kind of UI that would let me have a look at what was going on in a distributed system.

I know that tracing in distsys is a complex topic. My first idea was using `jaeger` or `zipkin` to visualize the traces, but that's not trivial, and after some researching I realized that this is definitely not what I wanted. It was a lot of effort to not achieve what I was looking for. 

Of course another solution was doing something custom. But this is not something small, and most likely a project of its own. 
My ideal UI would represent the nodes in a canvas and it would be able to reproduce their state, the messages they send to each other and their content. 

There are lots of different problems in here: 

* The UI needs to store the events and replay them step by step. 
* Time in distributed systems is an impressively complex and chaotic topic. Can we rely on the timestamps to order the events? Nope. So, how can we visualize them in order if we cannot trust the timestamps? For this toy system we could of course, it's not critical, and probably not a problem if they get somewhat disorganized. 
* Who keeps the history? this could be done if instead of a RPC solution the gossip protocol was using a message broker. But then, if we have a message broker, do we really need a gossip protocol? 

So, conclusion, the project does not have a UI. If you want to see what's going on you have to rely on logs and not having too many `docker` instances running :-/

This issue can be an interesting project on its own as I said. 

Few things to consider: 

* I know that I could get `zipkin` / `jaeger` to work. But I still think this is too cumbersome, and the result is still not what I want. 
* Service mesh tooling would also help quite a lot, but I still think I'm adding too much complexity into this. 
* Parsing the logs and visualizing them is also an interesting approach that could be taken into account. The logs are structured, so they are easy to parse. 
* Had this been a true simulation system in a single computer would have helped a lot. It would just be a matter of storing all the events in a DB and then visualize the results. 

Which in fact, is probably the best solution for this specific case. Given that this is a playground and I don't need to get extremely dogmatic probably a feasible solution would be to send all the "events" into another system that just unifies everything. To make sure that the events do not get unsorted I could introduce a constent delay. Basically, slow down the system so that racing conditions are not a problem. 
It's ugly but definitely would help with the visualization of the system for didactic purposes. 

### P2P discovery 

For a gossip protocol to work a node needs to locate other nodes. Conceptually easy to understand, but pretty hard to get it right. 

For my first try I went for the easy solution. There was a "master" node and all the workers registered into it (the ip of the master node was a parameter). Of course, this is extremely simple and convenient to implement, but it's a pretty bad system design, it's basically a SPOF. 

I wanted the system to be resilient and as close to a production system as possible with the limited time I had, so I had a look at the P2P discovery state of the art. [This post by Nordan Jantell](https://jsantell.com/p2p-peer-discovery) gives a nice summary btw. 

There are some interesting options. Some of them are really cool, but I needed something for this specific project. Something that would run in the same network and inside docker containers. So, no global DHTs. 

This left me basically with 3 options: 

* **Broadcast packets**: This one is cool. When a node starts up it sends a `hello world` signal to the network broadcast address, so everybody receives that. What I don't like about this solution is the amount of network messages it sends in the beginning. Right, it's just when a node bootstraps, but I wanted to see if I could come up with something slightly different. 
* **Central node** that knows all the nodes. 
* **External system**. Something like `zookeeper`. At the end of the day this has the same implications as having a central node. 

I played a little bit and I went for something custom, I haven't tested it fully and I would not put this into production without some real tests first. Consider it just a playground.

My solution is a mixture of different things. Instead of having one single master node I had the idea of having a "group of master nodes". So, when a node starts up it doesn't try to contact one specific node, but one specific node of a group. This way you balance the load between them. When these master nodes bootstrap they also use the same nodes, in this way we make sure that there are no partitions in the groups of peers. 

This works for this case. I haven't studied it deeply to understand the different implications it might have in a real system. Just a fun toy.

So, to sum up: A guy just told me that there is a nice gossip protocol playground around. Feel free to have a look at the [repo in github!](https://github.com/lant/gossip)
