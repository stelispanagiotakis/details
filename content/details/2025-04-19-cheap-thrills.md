---
title: "Cheap Thrills"
date: "2025-04-19"
---

Consider the case of a hobby project or a startup with no funding or clients for the foreseeable future.  
We need dirt cheap infra for development and demos.

What's the lowest price for:

- somewhere I can run a couple of containers of my services  
- a db to save my data, managed if possible  
- a Load Balancer to expose my app to the world  
- DNS so I can slap a name on the thing.

AWS Free Tier (or equivalent) is always on the table.  
But then you need to be vigilant of usage and attentive to details (and who's got time for the details, right? :-P)  
A simple VPC + subnets + NATs in AWS ends up costing somewhere near 15$ / month when accounting for Public IPs and traffic.  
And then we factor in our apps' needs.

Let's say ~ 20$ / month is the price to beat.  
That's not too bad already, is it? In a year you've spent 250$.  
If you have some way to monetization, you don't mind spending a bit more.  
If you don't, you shut it all down and think hard on your decisions.

But can we do better? ie cheaper?  
The solution I've grown to love is a bit complex but more than satisfies the price criterion.

- Compute: [Rackspace Spot](https://spot.rackspace.com). ~2$/ month gets you a capable k8s node. Yes, it's a bid in auction so you might be evicted or end up paying more. Let's assume double. Still a good deal.

- LB: One usually chooses the LB offered by the provider of their compute. For Rackspace that'd be 10$/ month. But if we expose the workload via Cloudflare Tunnels, it's free. And we get an SSL cert and optional authentication in front of the app.

- DB: 3 options. `a:` AWS free tier RDS for 1y. `b:` Cloudflare D1 has a good always free tier, and `c:` one could always just deploy a Postgres container in Rackspace and do frequent backups until data is actually worth better protection.

- DNS: If going with LB via Cloudflare, DNS needs to be there too.

So 2-4$ / month. Not too shabby, right?
