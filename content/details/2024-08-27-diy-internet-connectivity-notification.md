---
title: "DIY internet connectivity notification"
date: "2024-08-27"
---

WFH is great!  
But every once in a while, internet goes down mid-workday and you're not sure when it will be back up.  
You're forced to leave in a hurry for some nearby cafe.  
How can you know it's back up so you can return?  
I solved this once and while I'm not proud of how it worked, it worked!

Prerequisite: A host in that network and a Slack workspace or other service you're allowed to create a webhook for.

```bash
#!/usr/bin/env bash
while true;
do
    IP=$(curl -4 ifconfig.io)
    curl -X POST -H 'Content-type: application/json'
         --data "{"text":"Internet is UP. IP: ${IP}"}"  
         https://hooks.slack.com/services/xxxxx/yyy/zzz
    sleep 300
done
```
