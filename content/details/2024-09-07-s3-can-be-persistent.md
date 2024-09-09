---
title: "S3 can be persistent"
date: "2024-09-07"
---

I've always hated adding persistence in k8s workloads.  
There are valid reasons to avoid it, right?  
Adding a PV means people can store stuff there that they expect to find, regardless of where the app is deployed/redeployed (which could be in another region/zone/cluster/node etc where the original pv is not present)  
Thus adding another step in the deployment process that comes usually as an afterthought and may be easily overlooked and poorly documented.

But still, every once in a while you have to do it.  
One way I've found that is somewhat acceptable to the above intricacies uses S3.

AWS recently announced [S3 mountpoint](https://github.com/awslabs/mountpoint-s3).  
You can use it to easily mount an S3 bucket to a directory in your system or container.  
But to add this to a docker image you'd need to install the binary, set it up etc.  
Which means editing the dockerfile and the init script.  
If it's your app and you're fine with that, go right ahead.  
If not though, you might like to use a sidecar.

The sidecar would have

- a docker image with `mount-s3` installed and proper permissions to access the S3 bucket you want to use (using IRSA with the serviceAccount of the deployment probably)
- a volume of type `emptyDir` mounted at some directory in the container with a [mountPropagation](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation) of Bidirectional
- an init script to mount the S3 bucket to the directory of the `emptyDir` volume

And the main container would

- mount the same volume with a `mountPropagation` of `HostToContainer`.
- be able to store files in that directory and have everything there written to the S3 bucket, without the main container ever being altered.

Enough talk, show me the code

```docker
#dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y wget
RUN wget https://s3.amazonaws.com/mountpoint-s3-release/latest/x86_64/mount-s3.deb
         && apt install -y ./mount-s3.deb
         && rm mount-s3.deb
WORKDIR /app
COPY mount.sh .
ENTRYPOINT [“bash”,”-c”,”/app/mount.sh”]
```

```bash
#mount.sh
mkdir -p "/${BUCKET_TO_MOUNT}"
mount-s3 "${BUCKET_TO_MOUNT}" "/${BUCKET_TO_MOUNT}"
sleep infinity
```

```kubernetes
# snippet from deployment
spec:
      serviceAccountName: sa-that-has-access-to-your-bucket
      containers:
        - name: s3mounter
          securityContext:
            privileged: true
            capabilities:
              add:
                - SYS_ADMIN
          env:
            - name: BUCKET_TO_MOUNT
              value: your-bucket-name
          volumeMounts:
            - name: s3bucket
              mountPath: /mnt/buckets
              mountPropagation: Bidirectional
        - name: s3user
          image: ubuntu
          command:
            - "/usr/bin/sleep"
            - "infinity"
          volumeMounts:
            - name: s3bucket
              mountPath: /mnt/buckets
              mountPropagation: HostToContainer
      volumes:
        - name: s3bucket
          emptyDir: {}
```

- Is this fit for every purpose? Likely not.  
- Is it safe? Well, any privileged container is inherently not safe. Read the warning in the MountPropagation article above.  
- Are there better ways? Of course. There are actually even better ways to do what I described above. See [here](https://github.com/awslabs/mountpoint-s3-csi-driver).

But this approach too has its merits and seemed worthy of being documented.
