---
title: Get the env vars your app started with
date: "2024-09-14"
---

The setup should be familiar. You have an app that is ready to ship, so you put it in a docker image.  
Any config after that is done mostly via env vars.

For anything non-trivial like e.g - mounting and parsing secrets to env vars - making sure certain default values are overriden

you need an init script that calculates the var's desired value, then exports it like

```bash
export MY_CONFIG_VAR=calculated_value
```

And once done, you'd start the app with your equivalent of

```bash
exec java -jar myapp.jar ...
```

At some point after that, you need to know what the value to some of those variables was. So you exec inside the container and do a

```bash
printenv
```

but you can't find any of the vars you exported.  
Inconveniently, _that is to be expected_.  
The vars were exported for the shell that was running at startup time and that died once it handed over the PID to your app, not the shell that spawned when you run

```bash
docker exec
```

So? Is the info you need lost? Fortunately not.  
The processes store their _initial_ environment in `/proc/${PID}/environ`.  
So all you need is something like

```bash
cat /proc/1/environ | tr '\0' '\n' | sort
```

## Need the _current_ instead of the _initial_ environment?

That should be quite a bit trickier as a Google search will reveal.  
Good thing is, though, that processes altering their env vars is not (at least in my experience) a usual scenario.  
It's definitely not one I'd like to be in, if given the choice.

## Why use _exec_ ?

As in `exec java -jar myapp.jar` instead of just `java -jar myapp.jar`.  
Well, Docker is all about the process with PID 1.

- It's the one whose logs are shown in `docker logs -f` and the one that if killed, the container does too along with it.  
- It's also the one to receive and handle the signals, e.g SIGTERM from a `docker stop`.

Since we've used an init script to prepare the ground for our app, then it's the shell that has gotten PID 1.  
`exec` hands over the PID so now it's the app that has PID 1 and everything works as expected.
