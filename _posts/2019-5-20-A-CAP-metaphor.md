---
layout: post
---

*Disclaimer*: Metaphors are great for learning. If you want to learn more about this, there are a lot of great write ups out there that may better introduce this topic.

## Introduction
The CAP theorem states we can only have two out of three properties in a shared data distributed system - between strong consistency, availability and partition tolerance.

![CAP image]({{ site.baseurl }}/images/CAP.jpeg)

- Consistency: Reading from the system always returns the latest write.
- Availability: The system guarantees a response at all times.
- Partition tolerance: The system is distributed in multiple machines, in case of failure.

## First story
Let's suppose Bob, a single person who is able to speak, is such a system.

![Bob image]({{ site.baseurl }}/images/bob.png)

- *I walk in*
- Me: Hey Bob! Just came over to tell you Tim likes apples.
- Bob: Well that's nice to know!
- *I walk away*
- *Kit walks in*
- Kit: Hey Bob! What's up?
- Bob: Hey man, Tim likes apples.
- Kit: Wow, thanks for letting me know.

We can say that as long as Bob is not sleeping he is *available* (to answer), and as long as he is paying attention he is *consistent* (with the facts he keeps on his mind). But he is not *partition tolerant*. Why? Let's understand what that means in the context of the next story.

## Second story
Now let's team up Bob with a clone of himself.

![Bob and Clone image]({{ site.baseurl }}/images/bob2.png)

- *I walk in*
- Me: Hey Bob! Just came over to tell you Tim likes apples.
- Bob: Well that's nice to know!
- *I walk away*
- Bob: Hey Clone, Tim likes apples.
- Clone: Aweeeeesome man.
- *Kit walks in*
- Kit: Hey Bob! What's up?
- Bob: Hey man, Tim likes apples.
- Kit: Wow, thanks for letting me know.
- *Pete walks in*
- Pete: Hey, do you happen to know what Tim likes to eat?
- Clone: He likes apples.

As long as both Bob and his Clone are *available* (to answer) and *consistent* (to some degree, as we will see) we get to know stuff from both of them!

Regarding consistency, do take notice on the interaction between Bob and the Clone as Bob shares his knowledge. If we did not give it time to happen, the Clone would not know about this information. The way we put it, these guys are not strongly consistent, as both may only have partial information at a given point in time. This information will be shared on best effort and this can be called *eventual consistency*. This sharing of information is called *replication*.

Suppose now that Bob is struck by lightning before he gets to share information with the Clone. Remember when we said Bob alone can not achieve *partition tolerance*? With the help of the Clone, he can. While Bob is having a rough time recovering from his shock, the Clone will still be able to share his knowledge.

Or not. It depends on what the Clone chooses to do.

## Second story - ending one

- *Mike walks in*
- Mike: Hey Clone, it seems Bob was struck by lightning.
- Clone: Indeed.
- Mike: So, what's up with Tim?
- Clone: Not sure, got to wait for Bob to tell me what he heard about him.

Choosing *consistency* means we do NOT share our knowledge unless Bob was able to share everything before the thunder hit him.

## Second story - ending two

- *Mike walks in*
- Mike: Hey Clone, it seems Bob was struck by lightning.
- Clone: Indeed.
- Mike: So, what's up with Tim?
- Clone: Last time I heard he liked Bananas the most.

Choosing *availability* means we share our knowledge anyway, even if it is not the latest.

## Summary

This is a summary of the characteristics we observed here. Different or additional ones are most likely observed in similar systems.

###### CA
Bob alone at his best.

###### CP
Bob and Clone refuse to share knowledge unless they shared everything they learned before.

###### AP
Bob and Clone always share what they know regardless each other.

###### Both CP and CA (replication)
Bob and Clone will always attempt to share knowledge between themselves whenever they know new stuff.

That's it!