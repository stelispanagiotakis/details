---
title: "Port-forward non k8s service via k8s"  
date: "2024-09-08"
---

So, suppose my app that runs on k8s needs a postgres db to write to.  
I'll set up an RDS that is reachable only within my VPC.  
Then a dev needs to connect to that DB from their local to troubleshoot some weird issue.  
What do I do?  
The tried-and-true solution is of course some VPN.  
If you have the time and resources to do it right, you're golden.  
If not, a VPN is a surefire way to headaches. (new authentication pool, 2FA , maintenance, split DNS, slow connection speeds etc)  

Can't I just fake it for simple non-prod scenarios?

Why don't I just port-forward the DB through k8s, just as I port-forward k8s services?  
Well, you can only forward services that have endpoints within k8s.  
OK, I'll manually create the [endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#endpoints) for a service.  
But that'll only work if you can use an IP for that external service. You can't use CNAMEs with Endpoints. And for RDS you can't rely on IPs.  
You get a CNAME that does not promise to resolve to the same IPs. Being realistic, the private IP of the RDS instance may never change, but that's a risk you'll have to evaluate.

So? Is there a better alternative?  
There is an alternative. Whether it is better, or even good, is left to the reader.

As with most things we spend time thinking and building, there's a linux command that's been doing what you need for quite some time. In this case it's [simpleproxy](https://manpages.ubuntu.com/manpages/trusty/man1/simpleproxy.1.html).

The syntax is intuitive

```bash
simpleproxy -d -L 5432 -R your-rds-cname.rds.amazonaws.com:5432
```

This will effectively proxy any requests to port 5432 of the host where the command is run to your RDS instance's port.  
Now, let's put that in a container and deploy it in a k8s cluster that has access to the RDS and expose its port 5432 via a service

If we now port-forward the service via k8s, we can access the private RDS instance via local.

>note: feedback received, will be spending some time with [socat](https://www.redhat.com/sysadmin/getting-started-socat)
