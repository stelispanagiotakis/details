---
title: "Cache-Control"  
date: "2025-12-28"
---

There are two main problems in programming:

- Invalidating Cache
- Naming things
- Off-by-one errors

Let's do cache in the web today :-)

Suppose you have a static frontend that you need to serve.  
And you care about having globally available with low latency.  
That's the prime use case for a CDN, right?  
So you deploy a CDN ( Cloudflare, Cloudfront etc).  
And you point it to a source ( an S3 bucket with your static files or a publicly accessible web-server serving your files).  

And then you **release a new version**.  
How does the CDN know it needs to go back to your source to get the new files?
And since the app is static, it gets downloaded to your users' browser, right?  
How does the browser know to ask the CDN for the latest files?

You use headers.  
`Cache-Control: no-cache` ( although not too intuitively named) instructs the browser and/or the CDN that it CAN cache the files but before using them, it must re-validate with its source. ( The value that disables caching and requires always talking to the source is `no-store`). So we have the mechanism to ask for new files.  
But which files are new / have changed?  
The `ETag` header is in play here. Your source needs to create a hash of each file it serves ( usually the hash contains a combination of size and last changed timestamp. It varies between implementations but it ultimately does the same thing). That's how the second check happens. If the file has a different ETag, it gets re-downloaded. If not the cached version is served.  

We're good on the CDN part, but there's an edge case for the browser.  
What if the files are updated **while** you're using the page?
There's a trick that uses the service workers to detect a new version and invalidate the cache / request the new version.  
But it's framework/implementation specific and not my area of expertise, so I'll lazily leave the googling to the reader.
