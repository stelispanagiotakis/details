---
title: "AWS, why do you hate me?"
date: "2024-08-26"
---

It's generally a good practice to differentiate between application endpoints than need to be accessible via Internet and endpoints that need to be accessible only internally via your private VPC.  
But what if I need one app's endpoint temporarily accessible to my CICD runner that's not in the VPC?  
e.g a _/health_ to know if the deployment succeeded.

If you're on AWS, you may be correctly thinking PrivateLink, VPC Endpoints etc  
But that may be overkill for a short-lived connection from our CICD.  
The apps I'm interested in run mostly in kubernetes.  
To access an internal endpoint from my local, I'd just `kubectl port-forward`

And my CICD already has an AWS user, needed to push images to ECR.  
If I assign that user to a group that has access to port-forward in the clusters that I need, I'm good right?  
Well, with a little caveat.

Suppose I build a docker image to run all this from  
I install awscli and kubectl, then supply the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars to be used to get access, configure the AWS accounts, get the proper KUBECONFIG, then try to port-forward from Kubernetes

Nope, it will not

```bash
32 memcache.go:265] couldn't get current server API group list: the server has asked for the client to provide credentials
```

Why? Well, let me tell you.

As we do for our local setups, we input our user credentials, then assume roles for each env that we need to access  
But if called via kubectl AND env vars are present, awscli will blatantly disregard our meticulous configuration, and use just the env vars to get access.  
And it will fail (because it lacks the AssumeRole config)  
This is known and it is not a bug. [source](https://github.com/aws/aws-cli/issues/3875)

So what do we do?  
Well, after I configure everything, I just

```bash
export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""
```

and what do you know?  
It freaking works!!!!
