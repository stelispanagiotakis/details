---
title: "Empathy for Engineers"  
date: "2025-03-15"
---

I've been in a couple of leadership seminars. One of the concepts that arose as central for good leadership was empathy.

But what's empathy? Let's use [this](https://greatergood.berkeley.edu/topic/empathy/definition#:~:text=Emotion%20researchers%20generally%20define%20empathy,might%20be%20thinking%20or%20feeling) definition.

> ...the ability to sense other people's emotions, coupled with the ability to imagine what someone else might be thinking or feeling.

If someone says `I'm sad` and I hear that, am I now empathetic towards them?  
Empathy is not superficial, but an in-depth understanding of both the state (`sad`) and its underlying causes, ideally coupled with the ability to predict future behavior.

> Caveat: It's also a lot of work, takes a lot of time and it does not necessarily pay off.

In the same seminars, I heard it voiced, that this is a difficult concept to explain to Engineers.  
(Ironically this difficulty would be easily overcome by having a bit more empathy for Engineers :-P )

Is there an equivalent that might help Engineers understand how to gain empathy?  
Not the concept itself, a tool to get there.

Suppose your app connects to a Postgres DB to store data.  
At some point you notice that data isn't stored anymore.  
You have the state. What do you do to understand the nature of the issue and why it happened?

You enable **DEBUG** logging.

It may have been a Postgres patch update that auto-installed and is causing issues.  
It might have been something in your app's latest release.  
Or a network issue.  
You have the underlying causes.

By enabling DEBUG logging you've gotten a much deeper understanding of the causes, hopefully some idea about how to fix it and perhaps a few action items on how not to have this happen again. Also, same caveat as above applies here too. But if you spend the time...

You've got much greater **empathy for Postgres**.

**Now go do it for people too.**
