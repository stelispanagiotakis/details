---
title: "Why quote environment variable values in kubernetes manifests?"
date: "2024-08-26"
---

This a another discussion I have frequently.

- Why bother changing something, however small, if it works as it is?  
- Well, it's a best practice. If we can spend a very limited amount of time to make trivial changes that will protect us from mishaps, carelessness and changes beyond our control, is it not worth it?

This argument, on its own, is usually not very effective against a stubborn engineer in a hurry.  
So, whenever possible, I try to give more concrete examples.  
Here's one that I was happy to find just in time for such a discussion.

- Why should I take the time to quote all environment variables in a kubernetes manifest? If I leave them unquoted, they'll be interpreted as strings, which is what I want.

In a now deleted reddit post, available [here](https://web.archive.org/web/20230430055713/https://www.reddit.com/r/kubernetes/comments/1328x9p/psa_shortsha_container_names_guard_your_strings/) via WayBackMachine, the people at [kubefirst](https://details.42web.io/) had a field representing a container's sha, a **string**.  
No special characters etc  
They left it unquoted.  
At some point it got the value `1025e36`  
All numbers and `e`  
So it was calculated as a number in scientific notation. **1025\*10^36**

So yeah, quoting a string is usually not required, but if you can, please save us all a headache :-)
