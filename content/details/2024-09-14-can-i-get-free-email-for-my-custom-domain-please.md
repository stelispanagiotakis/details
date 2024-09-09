---
title: "Can I get Free Email for my custom Domain, please?"
date: "2024-09-14"
---

At some point you bought a domain. You made something of it, or (usually) not. Now you'd like an email address on that domain.  
It looks much nicer than your typical `xyz@gmail.com`, right?

Especially if you're a business, Google Workspace or Microsoft 365 is a no-brainer.  
You get your custom domain email, storage, collaboration apps, SSO and other tools for something like 5-10$/person/month.

But if you're like me (i.e you like the vanity of the thing but will not allow yourself the expense unless you know you'll actually use it or it'll make its money back somehow) you may be looking for something without monthly subscriptions, preferably free.

PLEASE resist the urge to self-host your email unless/until you're well-versed in the field and have good reputation IP ranges ready.  
Else, you're in for a fascinating/frustrating journey.  
Some people like this sort of thing, I don't judge :-P  
See [here](https://www.google.com/search?q=reddit+self+hosted+email&sca_esv=b67dbdab53dcd015&sca_upv=1&sxsrf=ADLYWIJ4JgMw-bUwZhXdDN9OqZg0luxFpg%3A1726344933337&ei=5e7lZqrKE_bXi-gPodbPwQo&ved=0ahUKEwjq0IqEoMOIAxX26wIHHSHrM6gQ4dUDCA8&uact=5&oq=reddit+self+hosted+email&gs_lp=Egxnd3Mtd2l6LXNlcnAiGHJlZGRpdCBzZWxmIGhvc3RlZCBlbWFpbDIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYRzIKEAAYsAMY1gQYR0iEFlDBBVjgD3ACeAGQAQCYAWygAWyqAQMwLjG4AQPIAQD4AQGYAgOgAnTCAgYQABgWGB7CAggQABiABBiiBJgDAIgGAZAGCJIHAzIuMaAH2gE&sclient=gws-wiz-serp "here") for inspiration/horror stories.  
And [here](https://www.reddit.com/r/selfhosted/s/Ja4VuDjoPw) for a proud success post, as well.

So? What's the other option?

**Email forwarding** is what I chose.

It's basically services that act as your mail server, receiving emails for the addresses you've set up and then forward them to your personal account.

- [Cloudflare](https://blog.cloudflare.com/introducing-email-routing "Cloudflare"): does this but they don't offically support sending from your custom address. I'd tried a hack for that, but it stopped working soon after.  
- [ImprovMX](https://improvmx.com/ "ImprovMX"): does this too, it's actually their primary business. They also support [sending](https://improvmx.com/guides/send-emails-using-gmail/ "sending") from Gmail. They've been around for more than a decade (that's a loong time in tech) and have generally good reviews.

You setup MX records in your domain's DNS, setup improvMX and then your Gmail for sending.

It's been working reliably for a while now.  
Will update if I notice issues.
