---
title: "Dev Ready Bare Metal k8s"
date: "2025-12-13"
---

We're building a shared development environment in k8s  
The typical cloud managed k8s services are not available.  
We need ample resources and low-as-possible price for k8s.  

Requirements? We have them

- Private access. Given private access to the machine's local IP address, it needs to be reachable and publish services to that interface.
- Public Access. Given a public IP assigned to its second interface, we need to be able to selectively publish services to the internet.
- Data is not important. We care about them as far as they are needed for the services to keep running properly.

Wanna try a Bare Metal server for this?

## OS

Talos is a great option if it supports your hardware. For our case it wasn't suitable as it does not support Soft RAID (see e.g [here)](https://github.com/siderolabs/talos/discussions/8654) so I thought we'd forego the flexibility of upgrades etc for something with some disk fault tolerance.  
Ubuntu it is.

## k8s

Nowadays there are many projects that will do for an easy k8s installation.  
I went with something already familiar, `k3s`. I ended up with an install script like

`curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=stable \ sh -s - --node-ip="SERVER.PRIVATE.IP.ADDRESS" \`  
`--advertise-address="SERVER.PRIVATE.IP.ADDRESS" \`  
`--disable=traefik,metrics-server,servicelb \`  
`--default-local-storage-path=/path/to/my/raid/volume`

Ok, so we've got a kubeconfig we can use to access the cluster from our local terminal.

## Storage

If we actually cared about data, we might need network storage and a csi driver to attach it to the cluster ( ceph if feasible, nfs if masochist). Since we can get away with a few good backups that can be stored elsewhere and used to restore functionality, the default storage class of k3s should be ok ( configured to store in the soft-raid backed partition). And also, per case, hostPath should also be an option.

## Public / Private Ingress

That's where I had the most fun.  
No automagic LBs spawning every time I christen a k8s service `type: LoadBalancer`.  
That would be great for the private Ingresses.  
The public ones though?  
I couldn't get k3s to network over one interface and ingress through another.  
There may be an incantation I'm missing there but it seemed counter-intuitive too, so I abandoned the thought.  
Let's sort out the private part and get back to the public one.

### Private Ingress

MetalLB was quite helpful for this.  
Creates an IP Pool (outside the DHCP range) and whenever I created a `type: LoadBalancer` service it happily reserved an IP from that pool and advertised it.  
So I installed traefik, got it its own private IP from that pool and were doing fine.  
Now about publicly exposing services?  

### Public Ingress

Briefly , I considered using a utility like `socat` to pipe all traffic from the IP of the private NIC ( or the metalLB one reserved for traefik) to the public IP of the second interface.  
It worked but apart from being an ugly hack, that would expose all services to the public, like wholesale.  
Even if no DNS pointed to it, a determined outsider could guess a few host header values and enter.  
But, traefik has another feature I like.  
You can configure it to bind to specific [hostPorts](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml#L826).  
But that's not going to work together with the previous setup with MetalLB.  
Fine, **2 separate traefik installations** then.  
One binds to ports 80 and 443 of the host machine and exposes all the ingresses it serves to public IP ( you'd need to also set the `hostIP` to the public one to block it from also using the private one )  
Another one that gets a private IP from MetalLB and is accessible privately.  
Each installation is configured to serve only Ingresses of a specific ingressClass ( e.g `traefik-internal` and `traefik-external`)  
And depending on the ingressClass we use, the service is assigned to the proper traefik instance and exposed privately or publicly.  

### ArgoCD

Hey, are we doing ArgoCD too? If so, we'll bump into another little issue that might aggravate our collective OCD.  
The `traefik-external` service will never get a "proper" external IP, so all Ingresses it serves will be always showing as " In Progress". We could add annotations to instruct ArgoCD to ignore them but, couldn't we just fix them?  
Yes, [we could](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml#L1008-L1033)

```yaml
service:
    enabled: true
    type: ClusterIP
    externalIPs:
        - "OUR.PUBLIC.IP.ADDRESS"`
```

This populates the `traefik-external` service with our correct public IP and hence sets the correct status for the ingresses

Now pair that with a couple of external-dns installations to update our internal and external DNS servers and we're golden :-)
