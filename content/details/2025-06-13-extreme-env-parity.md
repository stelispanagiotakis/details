---
title: "Extreme Env Parity"
date: "2025-06-13"
---

You need/want a local k8s environment to develop your container images, helm charts etc.  
And you're pedantic about env parity. Your envs should differ only where they absolutely need to.  
So that you can re-use your manifests/charts as usual and when you eventually push to a shared env, you do so with confidence.

You don't expect issues with the workloads. You can build or pull the containers.  
Persistence may be an issue but your local k8s solution should already have a storage class for you to use.

The interesting part come when considering DNS and certificates.

The scene:

- You have a backend service and a fronted service that you deploy to k8s for PROD.

- They need access to a postgres db and an S3 bucket

- You need to access the frontend via HTTPS

For DNS for your local machine, you can hack it into your `/etc/hosts`.  
Now, from your browser you can access your s3 service ( say minio for example) via `s3.mylocal.k8s`  
How about HTTPS? `mkcert` is there for you. Create a CA, trust it, get a certificate, load it onto your ingresses.

But within the cluster? How can the backend access the S3 service with HTTPS?  
DNS that resolves `127.0.0.1 s3.mylocal.k8s` does not cut it there.  
And also the cert ( if we could get to the service to see it) will not be trusted.  
So?

You could play with the backend pod's DNS settings etc but that's more of a hack in my opinion and it deviates from whatever you're doing in the proper envs.  
The best way I've found to get this done is to actually edit coredns to serve the proper records, pointing `s3.mylocal.k8s` to your ingress controller.

`kubectl -n kube-system edit cm coredns`

and there append

```bash
rewrite name s3.mylocal.k8s traefik.traefik.svc.cluster.local
```

and restart the coredns deployment.

That should take care of DNS in a civilized manner.

How about trusting the certs within the backend service?

You could mount the certificate in the pods and do an `update-ca-certificates` to trust them ( or even bake them into the `dev` docker images) but that's too much deviation for my OCD.

What I ended up doing :  
First of all make sure my ingress serves the fullchain cert.  

```bash
# make sure the cert is first, CA is second
cat mycert.pem myRootCA.pem > full.pem
kubectl create secret tls my-full-secret --key=./mykey.pem --cert=./full.pem
```

Then override the entrypoint of the backend and use `openssl` to get the full chain certificate served by my ingress.  
And finally run `update-ca-certificates` to trust it.

```bash
    spec:
      containers:
      - command:
        - sh
        args:
        - -c
        - apk add openssl
          && cd /usr/local/share/ca-certificates
          && openssl s_client -showcerts -verify 5 -connect s3.mylocal.k8s:443 < /dev/null | awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/{ if(/BEGIN CERTIFICATE/){a++}; out="cert"a".pem"; print >out}'
          && update-ca-certificates
          && /entrypoint.sh
```

Of course , overriding the command your backend runs is a deviation in itself.  
But it's the least invasive one I could find .  
If one were going into politics, they could argue that it doesn't break env parity, it restores it.  
But we ain't about that here :-)
