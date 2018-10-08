---
layout: post
title: Hard and soft Layers
---

# Hard and Soft Layers

I've overall found [Hard and Soft Layers](http://wiki.c2.com/?AlternateHardAndSoftLayers) are
a very strong way to structure complex systems, such as a more complex infrastructure or a
complex code base. I would like to write down more concrete examples I've encountered
so far, because the description on the C2-Wiki is rather abstract. 

In the following, I'll show you my understanding of this pattern and my conclusions from there,
and then I'll follow up with more concrete examples. This way all of you impatient readers can leave
quickly and I can ramble some more during the examples. On the other hand, you may find yourself jumping
to the examples first and then you can try to discover why I like the pattern in the examples. Up to you. 

## What are hard and soft layers?

The idea is to think of the system in different layers of various kinds. Some of these layers
are **hard** in the sense they are *more difficult* to change. Changing these layers might either
require deeper experience with the system you're configuring or you might need to be aware of
remote, unexpected and potentially not immediately visible side effects. Hard layers tend to expose
a lot of power to the person working on it, but it also requires a lot of reponsibility.

However, each of these hard layers should be followed or topped - or shielded by a **soft** layer.
Soft layers should be designed in a way so they are *easy* and ideally *safe* to change. An ideal
soft layer gives the users of this soft layer all the power they need, with very little chances
of blowing their leg off accidientally. 

I think this is a good pattern for at least 2 reasons, I might add more in further edits:

 - This gives us opportunities for **delegation and learning** inside a team. The more experienced 
   guys with a technology can create and maintain the hard layers they are good with. The less
   experienced guys should be able to handle a lot of incoming tasks and traffic in the
   corresponding soft layers. This in turn frees up time of the more experienced operators to
   make more profound progress. 
   
   But it goes on from there. It allows operators working on the system to understand the effects
   they need to achieve first without being overwhelmed by the potentially massive complexity
   of the hard layer. As you'll see later, our configuration management allows to configure 
   a linux firewall without throwing the whole book of iptables at you in raw form.
   
   However, you can dive into the more complex layer and follow a red thread around in order to
   gradually discover the harder layer - "oh this is how ports are whitelisted, that isn't too hard".
   And this gradual learning eases people into learning, understanding and internalizing the
   harder layer until they are able to change or maintain it on their own. As a member of a former
   team said - "You kinda just wander down the rabbit hole at a pace, and after a year you realize
   just how deep you ventured without noticing".
   
   And overall, I like patterns which allow people to learn and grow.
  
 - This pattern increases overall **safety** and decreases *risk across entire teams*. Changes in hard 
   layers tend to have less safety nets, larger blast radiuses and generally more destructive
   capabilities than changes in soft layers. Changing the wrong value in our soft terraform
   layer might delete an application server after a big warning. That's not too terrifying, honestly.
   Things might slow down for a while, but the service will probably survive. And we can rebuild it
   within a few minutes.
   
   On the other hand, if you change the wrong stuff in our hard terraform
   layers and circumvent the build server (and... **why would you do that!**) - you might delete
   all productive database servers in important deployments. That's quite a bit more terrifying, 
   because restoring those from backup is hard and we'll get nasty phone calls. Lots of nasty
   phone calls, very quickly. 
   
   And safety is good, not just for preventing catastrophes. A safe system allows you to move 
   faster, because you have to be less careful. We can confidently push changes into the soft
   layers and we don't have to agonize and think hard about these changes. Our system and our
   safety layers will tell us if the changes are good or bad - or in other words, sometimes it's
   better to try three times and fail on the build server instead of spending a week analyzing. 
   
## Examples

### Linux Firewalls

We're running a couple of systems without configurable firewalls by the hosting provider. On these
systems we're using the linux firewall as a general security measure, since only dropped traffic
is good traffic. For legacy reasons, and lack of time to make a transition, we're configuring 
iptables directly, instead of using firewalld, but that's besides the point, and actually
makes this example better. 

Overall, direct configuration of iptables can be kinda tricky in my opinion, even if you 
only set the INPUT and FORWARD policies to drop, and then use a number of rules to `-j ACCEPT`. 
That last sentence is a lot of reduction of complexity of what iptables can be used and
abused for and introduces a lot of structure you just have to know or learn. Or maybe someone
else is doing it entirely different. And even then, it's still fiddly to remember all the
details of `-p tcp -m tcp -dport 443 -j ACCEPT` just to whitelist that highly uncommon port.

I think that qualifies raw iptables as a whitelisting firewall as a somewhat hard layer. Other
crazy things iptables can do aren't better.

This is why we have written our iptables cookbook in the configuration management. Using this
cookbook, the madness of whitelisting http + https to the world reduces to:

```ruby
default['acme-iptables']['public-http']['allow'] = true
```

That's it. That whitelists http + https from all ipv4 and ipv6 addresses and drops
all other public traffic. It's expressive, to the point and hard to get wrong. And
the cookbook behind all of this ensures and has been tested extensively to allow
exactly this traffic, and nothing else.

And on top of  that - totally invisible to the user of the cookbook - using this cookbook
ensures our monitoring  still has access, ensures we still have management access
to the box and it establishes a panic disable for the firewall, as long as the node has
acccess to the chef server. And both the management access and the access to
the chef server are hardcoded in 2 or 3 ways so it's really hard to break.

That's a really good *soft layer*. You write down what you need, and it's hard to mess up
too badly even if you try. 

## Terraform

Another pretty example is our terraform setup. We are currently running 2 layers 
in our terraform setup. We have our *shared modules*. These shared modules define
the structure of application clusters. On top of that we have the actual
*module definitions*. These module definitions parametrize the shared modules and
end up in the actual infrastructure running an application cluster. 

For example, the shared module defines how app01, app02, master01 and es01 are
networked. app01 and app02 are connected for hibernate caches, app01, app02, master01
are in a security group for mysql traffic. However, master01 and es01 are 
firewalled from each other - the mysql database and the elasticsearch cluster have
no business talking to each other. 

Based on this, our terraform modules are a real nightmare of remote, invisible action
and sadly I don't see any good way to avoid that:

 - Change the wrong security groups and you're possibly making a future security
   incident far worse since firewalls don't prevent further lateral movement of an
   attacker.
 - Change the wrong volume / disk setup and in the best case you'll immediately
   fail a database recovery, or you might cause a bunch of annoying out-of-service
   work for someone a couple of month down the road.
 - Change some of the lvm / disk setup scripting and you'll make the latter case
   a lot worse
   
And the list goes on and on. I'd like this layer to be less magical and intertwined
with other layers - and there are tasks that reduce the complexity - but at the moment
that's what it is. It's a rough layer to handle once you have a couple of productive
systems running on one of the shared modules. 

But at the same time. Need to scale up a cluster by an application server? That's not hard:

```hcl
...
   app06_enabled = true
...
```

Need to clone a system for a customer specific task? Well copy the module configuration over and
adjust:

```hcl
...
customer = "random-org"
...
```

And that's just it. It'll take care of monitoring, firewalls, integration with the configuration
management, bootstrapping the secret management, work around a bunch of known issues. This is a
nice soft layer, and most changes to our terraform configuration occur in this layer. As it should. 
