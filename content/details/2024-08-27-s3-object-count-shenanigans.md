---
title: "S3 Object Count shenanigans"
date: "2024-08-27"
---

Suppose you have access to an S3 bucket in some AWS account.  
You need to get the files to another bucket (same or different AWS account I think makes no difference to my eventual point)

You do your due diligence and count the objects in the origin bucket

```bash
aws s3 ls s3://originBucket --recursive --summarize | grep "Total Objects:"
```

You get the total count and start your sync

```bash
aws s3 sync s3://originBucket s3://destinationBucket --delete --exact-timestamps
```

Great. It completed successfully. Now let's count the objects in the destination bucket

```bash
aws s3 ls s3://destinationBucket  --recursive --summarize | grep "Total Objects:"
```

And they differ by a very very small number.  
You might run the sync again, just to be sure. Nope, nothing to sync.  
You might even start comparing subpaths fileCount between buckets to drill down to the ones with a difference.  
And finally find some folders that report a diff by 1 between buckets.  
Despite having the exact same number of files.

**What gives?**

Well, in my case it was ~200.000 objects that no matter what I did differed by 2.  
I [eventually found out](https://stackoverflow.com/questions/67790428/why-are-my-aws-s3-object-counts-different) that

> _When a user clicks the "Create folder" button in the S3 management console, it creates a zero-length object with the same name as the folder. This 'forces' S3 to display the folder even when it is empty. Amazon S3 does not actually need folders to be created (in fact, they don't actually exist!). Depending upon how you copied the objects between buckets, these zero-length objects might not have been copied._

So I was chasing ghosts created by someone who innocently clicked _Create Folder_ in S3 Console twice some years back.

Oh, well...
