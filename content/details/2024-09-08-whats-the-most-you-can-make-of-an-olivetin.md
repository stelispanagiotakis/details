---
title: What's the most you can make of an olivetin?
date: "2024-09-08"
---

No, not that kind of olivetin, [this](https://github.com/OliveTin/OliveTin) one :-P

OliveTin is a simple web UI that can be configured via yaml to run cli commands. You add a section such as

```yaml
actions:
- title: Restart backend
  icon: restart
  shell: docker restart main-backend
```

and in its UI you'll find an icon with your title that, once pressed, will execute the shell command you specifed. You can even add arguments

```yaml
- title: Ping host
  shell: ping {{ host }}
  icon: ping
  arguments:
    - name: host
      title: host
      type: ascii_identifier
      default: example.com
```

My only slight complaint is that while it sort of supports [authorization](https://docs.olivetin.app/_security_configuration.html) it does not support authentication. If you're deploying it to be publicly accessible, you'd need to configure your ingress to authenticate the user and only then redirect them to the app.

EDIT: it has now been [updated](https://docs.olivetin.app/security/oauth2.html) to support both local users and OAuth2

**So, why use it?**

For me it's proven to be an immensely valuable tool to streamline processes.  
Business uses an environment.
Afterwards they need to reset it. Resetting is manual, time consuming and error-prone.  
They offload this to another department that offloads it to Engineering.  
And as an engineer, you're stuck waiting for the trigger to travel down that chain that instructs you to login to a host and execute the same 2-3 commands.  
You take your time to get around to it (it's not your top priority and you're not really motivated after all).  
All the while Business is blocked and asking up and down the chain about the reset.  
Not optimal.

With OliveTin and a sprinkle of authentication you can enable your users (business or not) to self-serve by providing access to powerful cli automations via a friendly UI. They are happier and you save your time and sanity.

After all, the definition of DevOps has nothing to do with developers and source repositories. It's about streamlining and automating processes to facilitate quick iterations.

And it's there that OliveTin shines.
