---
title: ConfigMap Confusion
date: "2024-09-16"
---

This has happened to me more than a few times...

We're mounting a ConfigMap to our container and using it to configure some aspect of the app inside it.  
Then the ConfigMap changes.  
But the app is unaware and does not react/reconfigure itself. Which component is at fault here and how can we fix it?

Until this bit me, I was under the impression that Configmaps mounted in containers do NOT auto-update their contents in the container.  
At least not without restarting the container itself.  
There are even projects that undertake parts of this responsibility.  
Like [Reloader](https://github.com/stakater/Reloader "Reloader") or [SpringBoot Config Watcher](https://docs.spring.io/spring-cloud-kubernetes/reference/spring-cloud-kubernetes-configuration-watcher.html "SpringBoot Config Watcher").

Well, I was wrong. As [Kubernetes docs](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/ "Kubernetes docs") state,

- if you've created env vars from a ConfigMap, you **cannot expect them to auto-update** when the ConfigMap is changed.
- But if you've mounted the ConfigMap as a volume, **the contents do auto-update**.
- **Unless** you've mounted it as a [subPath](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-specific-path-in-the-volume "subPath"). Then it won't auto-update either.

Confused yet? Let's dive a bit deeper then :-P  
It kind of makes sense that env vars created from a ConfigMap will not auto-update. They are created along with the deployment and never re-evaluated. So it'll take a `rollout restart` to get them refreshed. What about `subPath`, though? Well, as stated in [this](https://github.com/kubernetes/kubernetes/issues/50345#issuecomment-321346592 "this") issue, symlinks are involved. How? Well, as we found out:

- you mount a configmap that would create a file, e.g `app_params.yml` as a volume to a path in the container, e.g `/mnt/myconfig`
- in that path a subfolder is created, named `..data` and the contents of your ConfigMap are placed in there
- then a symlink is created from `/mnt/myconfig/..data/file_with_content_from_cm` to `/mnt/myconfig/app_params.yml`.

So your app reads from the symlink.  
When the Configmap is updated, the file contents in `..data` are updated too and transparently to the app, so does the symlink.  
So your app does have access to the updated file.  
But when you use a subPath, you're not mounting a folder, but a specific file.  
So the `..data` + `symlink` mechanism breaks and the contents of the file are not updated.

> This mechanism is NOT outlined anywhere that I've seen in Kubernetes docs. Only in scattered references in StackOverflow (found [this](https://stackoverflow.com/questions/62776362/k8s-configmap-mounted-inside-symbolic-link-to-data-directory "this") and [this](https://serverfault.com/questions/1062463/which-kubernetes-configmap-symlink-to-use "this") ). And [here](https://github.com/search?q=repo%3Akubernetes%2Fkubernetes+..data&type=code "Here")'s a search for `..data` in kubernetes' source. So it seems like it's here to stay but verify, then trust.

Cool, so we're good on the ConfigMap part. Does the app know, though? Likely not.  
Because, much like the env vars from ConfigMap, many apps are configured to look for and use config sources at startup only.

Is there a way to force them to re-configure? Preferably without restarting, right?  
As always, it depends on your language and framework of choice.  
If using Java/SpringBoot, you can enable the `/refresh` endpoint in your actuator.  
When you reach that endpoint, it'll force any classes you've annotated with `@RefreshScope` to re-evaluate and your app will get the fresh config.  
How will you know that a ConfigMap was updated in order to hit `/refresh/`?  
That's what SpringBoot ConfigWatcher does.  
>Interesting tidbit: when annotating with `@RefreshScope` Spring creates a bean and a proxy.  
It uses the proxy so it's free to replace the bean when a change happens. So... same thing Kubernetes does but on the app side!

But what if (am I tiring you already with the whatifs?) you don't want another new deployment just for watching ConfigMaps?  
Couldn't we get the event of a ConfigMap change within the pod and react there?  
What about using [Java Nio2 WatchService](https://www.baeldung.com/java-nio2-watchservice) to watch for filesystem events and then call `ContextRefresher`'s `refresh()` that does the same thing as the `/refresh` endpoint internally.  
That'll work, but beware. You cannot expect to catch changes to the actual file (`/mnt/myconfig/app_params.yml` from above).  
Because of the mechanism outlined, there won't be any.  
You can either

- watch the whole mounted folder. You'd catch a few more events and react needlessly, but you'd definitely get your result.
- watch the `..data` subfolder. You'd be more precise but also risk a bit with the tight coupling with an undocumented mechanism.

If you go the second route, you may want to restrict the events you act on (multiple pointless refreshes can't possibly be a good thing, right?). So you might want to watch only for `ENTRY_DELETE` events of files in `..data` with a specific pattern.  
Try this one

- `^\\.{2}\\d{4}_\\d{2}_\\d{2}_\\d{2}_\\d{2}_\\d{2}\\.\\d+$`

> The java specific part was donated by the friend who implemented it. I felt it needed to be here for completeness' sake.  
I dislike articles that go far enough to boast, then leave you hanging.  
[Here](https://medium.com/@xcoulon/kubernetes-configmap-hot-reload-in-action-with-viper-d413128a1c9a "Here") is an implementation in Go for hot-reloading ConfigMaps.  
And [here](https://blog.techiescamp.com/docs/time-taken-for-configmap-update/ "here") is the frequency and relevant setting Kubernetes uses to start serving an updated configmap
