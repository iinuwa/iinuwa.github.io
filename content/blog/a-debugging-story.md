+++
title = "A debugging story"
description = "Learning debugging principles from a production outage"
date = "2025-07-30"
tags = [["debugging"]]
+++

Recently, I sat down to play a round of Clue with my family. It was one of
those exciting rounds where everyone thought they knew enough for an
accusation, so we all rushed to the center of the game board at the same time.
It turns out that it took 4 of us guessing, one after another, to finally get
the right answer. When we found out the real answer, I reviewed my notes (I
like to keep extensive notes) and realized I could have known the answer about
5 turns earlier.

I often read debugging stories and postmortems because, let's be honest, it's
fun to watch things break in interesting ways up when it's not your problem to
solve. But more importantly it gives an intuition on how things can go wrong
and how to avoid and recover from faults. But sometimes I feel similarly about
the software system postmortems as the Clue, ... err, postmortem, above. I
often read some of them wondering, "Could I have figured that out?"

Now that I have a few more years of experience debugging, I know two things:
- Reading a debugging story is like reading a mystery novel backwards. Clues
  become obvious once you know the answer. Actually experiencing a debugging
  session is totally different.
- Debugging is a skill you can learn with guiding principles to help you.

Here is a story of debugging issue that I encountered earlier in my career.

# The Problem

During a time when my role focused more on system administration than
development, the developer team approached us with a problem. We had two
real-time applications that supported our live radio campaigns: a call center
monitoring application and a dashboard application. Our call-center application
used WebSocket connections to record operator status in real-time and enabled
other features like in-app messaging to contact call-center supervisors or
initiate call transfers. The dashboard application displayed incoming donations
and statistics for our radio announcers to see live during the campaign.

During our live radio campaigns (but never during our normal call-center
operations), all of the call center operator and dashboard connections would
drop. The actual phone audio was sent over a separate, unaffected channel, so
calls were uninterrupted. But it required a refresh for the operators and
announcers, so it was pretty disruptive. The issue would occur intermittently:
one day it would be fine, and then it would “crash” the next day, then fine one
day, then crash sometime later. Never the same time of day though. This issue
was tricky to debug because we also couldn’t reproduce it: load testing the
server didn’t result in the same issue.

# The Environment

We had recently replaced our hardware load balancer with NGINX, and so our
developers assumed that there was something wrong with NGINX because they
hadn’t changed anything fundamental in their systems, and there were no errors
in the application logs. It also affected two separate services, so we
assumed that the reverse proxy was the common point between them. I verified on
the proxy that the TCP connection closes were initiated by the application
server-side, not the proxy, so it was something in the application.

I didn't know it yet, but there were also three key facts needed for
understanding the cause:

- The previous load balancer was both relatively slow and did not support
  WebSockets (so the application used Server-Sent Events), but NGINX was fast
  and _did_ support WebSockets.
- The developer team reached a milestone of stability and did not need to do
  any mid-day updates to the dashboard application.
- During live campaigns, the dashboard application runs constantly as users
  load the application to display on giant TV screens.

You might be able to guess already what the problem was, but it took us a bit
of time to correlate these facts.

# Discovering the solution

For a while we were content to shrug our shoulders about it, but I really
wanted to resolve the problem once and for all. So before another campaign, I
took out the big guns: I added Prometheus metrics to the application to show
its internal state, and I wired up the application server to ship application
and OS-level logs to our new central observability platform that I had been
working on. I also added some OS-level CPU and network metrics (thanks to the
community building [Prometheus Windows
Exporter](https://github.com/prometheus-community/windows_exporter)).

Not all of these were necessary; in hindsight, we could have figured out the
problem without some, but they helped to eliminate solutions. After observing
the failure again, I helped to look into the OS logs on the application server
and discovered the problem: IIS was restarting the dashboard application.
That'll do it.

Because the dashboard application was stable, it stayed alive for the first
day, we never did any manual restarts. Previously, we would plan to restart the
application to push updates during campaign downtimes, but we didn't need to
anymore. What we didn't realize is that was resetting the default 29 hour IIS
application pool recycle timeout. Without our manual updates, the application
would restart itself sometime during the live segment of the campaign. The
application recovered just fine, and then the 29-hour cycle would resume.
Because it was not a regular time interval divisible by 2 or 3, the pattern
wasn’t obvious.

This issue also only occurred during live campaigns because users would only
keep the dashboards open for that long during these campaigns. Because the
issue was fundamentally related to the application runtime and not performance,
load testing was ineffective to reproduce the issue. Like I said, it's easy to
see in hindsight, but we hadn't enabled the logs we needed to see it happening
before, and there were other distracting correlations that muddled the issue.

So that explained the issue with the dashboard application, but not the call
center application. That one shut down after hours when the call center was
closed, so it wouldn't trigger the recycle timeout.

However, because the application was now using WebSockets, the connections
remained open longer than a normal SSE + HTTP keep-alive that was being used
before migrating to NGINX. There was an OS-level limit on how many TCP
connections could be established at once. When the dashboard application
restarted, that caused all the clients to renegotiate connections, which
overwhelmed the application server’s pool of TCP connections, which caused the
call-center application WebSocket connections to close, and then to renegotiate
again. So the dashboard application restarting cascaded into the call center
applications.

# The lessons

Ultimately, the solution we went with was to change the dashboard application
to always restart overnight so that it never hit its recycle timeout during the
day. If either change had occurred by itself, then the system would have been
fine, but the interaction between the two changes caused the error, but we
focused on the more visible change that was temporally correlated. Solving the
problem still required looking at the whole environment.

This has become one of my most often repeated quotes when teaching junior devs
about debugging:
> [Temporal] correlation is not [always] causation.

Using "when did this start happening?" is a very useful starting point for
debugging, and often resolves simple issues. But complex problems cannot be
discovered by temporal correlation alone, and sometimes, it becomes a red
herring. It’s important to observe contemporary events, but to accept that they
may only be a partial cause of the problem, or not the cause at all.

Another thing that would have helped resolve the issue faster was to already
have observability tools set up before we experienced the issue. After that
incident, I went on to expand our observability platform's capabilities in our
environment, like adding end-to-end tracing that were automatically correlated
with logs.

Later on, I read [David Agans’ book on debugging](https://debuggingrules.com/),
and found that many of the techniques I had built around debugging are ones he
codified; in this case, "understand the system" and "quit thinking and look"
were important for finding the problem. I recommend that book for any one
wanting to level up their debugging skills.
