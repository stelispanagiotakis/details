---
title: "If my java memory serves me well..."
date: "2025-12-13"
---

You've got a backend in Java deployed in k8s and every once in a while it OOMs.  
What a surprise :-P

How do you go about fixing this?  
Well, give it more memory since it's out!  
About half the time this will help.  
But in case you're not lucky and you need to fix the underlying issue, let's take the engineering way around.  
It's full of twists, turns and _wasted_ time.  
But if you need it, it's indispensable.

OK, first of all let's get some ground terminology right.  
After all Java memory management is a deep topic.  
[Here](https://www.baeldung.com/java-jvm-memory-types "Here") is an article from Baeldung giving us the high level overview.

One issue I've seen is that when deploying Java apps we seem focused on providing enough Heap memory.  
The rest of the areas usually don't get as much attention.  
The official recommendation is to define max heap as a percentage of total memory available.  
After all, Java has been container aware since 11 and can detect how much RAM it's been allocated by the deployment. So we can skip the hardcoding of `xms` , `xmx` and aim to provide 3/4 of total ram to heap.  
It's also a good practice to set minMemory=maxMemory.  
More info and specific variables to use [here](https://www.baeldung.com/java-jvm-parameters-rampercentage).

This alone might save you if inadvertently your settings were starving some memory area other than heap.  
But we better start gathering some data on these areas and their usage.  
In short ( more details [here](https://www.baeldung.com/spring-boot-prometheus)) we'd need to install, enable and expose a prometheus monitoring endpoint.  
Then scrape the data and push it to a prometheus instance to be consumed by a Grafana dashboard.  
A short guide can be found [here](https://medium.com/simform-engineering/revolutionize-monitoring-empowering-spring-boot-applications-with-prometheus-and-grafana-e99c5c7248cf).  
Alternatively, one might like to try Grafana Cloud. I've lately grown very fond of their free tier, which has gotten me very far in a few situations.  
So now we have our dashboards and can be sure all the memory areas are getting enough memory, or which it is that may be headed for issues.

We're already in a much better ( or at least more observable) state than before.  
But what happens if a couple of months forward you still get OOMs and you have no clue why?  
**Profiling**. It used to be a tool for a break-glass kind of situation. Enable and expose JMX, then connect with a compatible client to see where the memory is being allocated. With tools like `pyroscope` this can be continuous.  
`pyroscope`'s agent needs to be installed and configured for your service ( quickstart [here](https://grafana.com/docs/pyroscope/latest/configure-client/language-sdks/java/#:~:text=To%20configure%20the%20Java%20SDK,to%20configure%20HTTP%20Basic%20authentication.) ). It then sends data to its server component, which produces [flamegraphs](https://grafana.com/docs/pyroscope/latest/introduction/flamegraphs/). What answers can these graphs provide?  

`During some period of time how much memory was allocated and which functions called for its allocation, in a hierarchical order?`  

This should get you pretty far in determining what caused the OOM.  

I'm sure there's so much more nuance to the subject than what's mentioned above. But as a very high level overview for an Ops perspective, I can hope it's useful and (especially if found during an investigation) will leave you in a somewhat better place than it found you :-)
