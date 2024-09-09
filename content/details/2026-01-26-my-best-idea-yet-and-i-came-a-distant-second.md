---
title: "My best idea yet, and I came a distant second"
date: 2026-01-26
---

I've been using `traefik` in various contexts over the years.  
It's a bit much at first glance but once you get to know it, it can do anything ( at least anything I've thrown at it) and it's extensible via plugins.  
The winning feature it had when first introduced was `letsencrypt` support.  
You could have the ingress controller manage your certificates and inject them in any ingress you instructed it to. I've been using that feature for a long time.  

Recently, one of the projects was getting a bit more serious, so I thought I'd look into reliability a bit more.  
Naturally, my first thought was to run 2 traefik replicas to have HA.  
But I'd already been using `letsencrypt` support, so I had a persistent volume mounted to keep the `acme.json` file it uses to store certificates.  
This would have to be mounted to both replicas. How to do that?

- `ReadWriteMany PVC`: it was not available ( actually it was, but my provider's implementation was giving me weird errors, so I jumped at the chance to remove it rather than build on it.).  
- `Object storage`: I could add a shared volume [backed by object storage](https://details.systematize.it/2024/09/07/s3-can-be-persistent/) the proper way, with a CSI driver. I found one that [seems best maintained](https://github.com/veloxpack/csi-driver-rclone), then noticed the [remounting issue](https://github.com/veloxpack/csi-driver-rclone/issues/58), which seems interesting enough to deter me from tethering my ingress to it.  
- We could have a sidecar mounted to each replica that **syncs with object storage** but this too seems like too much work for something like this. Put a pin on that and let's revisit only if needed.  
- Since it's just one relatively small file, I could just do away with storage and **sync it to a db**. But this too needs quite a bit of work, let's not spend so much time if we have other options.

Then came the great idea.  
If you need a DB, you have one built into k8s and you also have a system to authenticate with it and a cli to facilitate that (`etcd` is _accessible_ via `kubectl`).  
So I could store the info in a k8s object that would then be stored to etcd. It's a cert so that would be a k8s secret, right? Create a traefik sidecar that has an RBAC role to only `get/patch/update` one specific k8s secret (if you also want to `create` you can't limit to specific resourceNames) and have it do that every time there is a change to the `acme.json` file. Yep, that'll work and it'll be the easiest and most robust of the existing solutions. And come to think of it, this would be a nice little side-project. I bet everyone would need a tool to manage their certif...  

And then it hit me... I'd just described `cert-manager`.  

Oh, well, let's use that then :-D
