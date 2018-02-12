---
layout: post
title:  "Too many signals - Resque on heroku"
date:   2015-04-01 13:05:01 -0500
categories: resque ruby rails heroku
---

At [Grouper](https://joingrouper.com), like a lot of the Rails community, we use [Resque](https://github.com/resque/resque) to manage our asynchronous background jobs. It lets us do everything from sending emails to processing charges and most of the time everything works perfectly.

## The problem

But sometimes things don’t work quite as well as we’d like. Recently we ran into a common problem with Resque deployments on Heroku - our jobs were being sent [TERM](http://unixhelp.ed.ac.uk/CGI/man-cgi?signal+7) signals in the middle of processing.

A quick review of Heroku signal handling can be found [here](https://devcenter.heroku.com/articles/dynos#graceful-shutdown-with-sigterm).

TL;DR - Heroku reserves the right to send TERM signals to any dyno whenever it wants. Note that the TERM signal is sent to EVERY process in the dyno, not just the process type. Anything running is given 10 seconds to clean itself up and shut down before Heroku sends it a KILL signal which causes it to die immediately.

At first this didn’t seem like too big of an issue, just catch the TERM signal and reenqueue the job (we use [Resque Retry](https://github.com/lantins/resque-retry)) - no harm done.

Except that not all of our jobs are fully idempotent. Running them multiple times could cause users to be charged more than once or to get emails more than once which is definitely not desirable.

## An example

Here’s an example short-running job similar to one we run all the time:

{% highlight ruby %}
class TrackEventJob
  def self.perfom(tracking_id, event_name)
    EventTrackingService.track(tracking_id, event_name)
  end
end
{% endhighlight %}

And a snippet from our Procfile:

{% highlight ruby %}
worker: env TERM_CHILD=1 bundle exec rake resque:work QUEUES=*
{% endhighlight %}

What happens if this job receives a TERM signal in the middle of processing? It immediately raises `Resque::TermException` which aborts execution. If we had set up Resque Retry to reenqueue the job then that would happen at that time. However, there’s no easy way to tell if the event tracking service received data from us before the TERM signal ends the job. Hence by retrying we could end up sending the event more than once and thus confuse our analytics.

Similar reasoning applies to charging credit cards or anything that you can’t make atomic.

In particular we found that our very frequent short running jobs were failing the most. On the order of several hundred per day as a result of using [Hirefire](http://hirefire.io/) to dynamically scale our Resque workers, and deploying continuously throughout the day. Frankly, we were pretty surprised by the sheer number of jobs that were failing and it was obvious we needed to do something about it.

Since retrying the jobs was not an option, we settled on giving them a few seconds between when Heroku sent a TERM signal and when they actually raised `Resque::TermException`. At the very least this would let our short running jobs complete execution in most cases and return normally as opposed to being sent KILL signals and ending with `Resque::DirtyExit`.

## The solution

### A possible solution

First some terminology:

- Parent: The Process that pulls down jobs but processes them in a fork.
- Child: The fork of the parent process that does the actual processing.

We wanted to give our short running jobs (which generally take < 1 second to process) an opportunity to finish executing rather than having them die immediately. Unfortunately this didn’t seem to be easily possible given how Resque was dealing with signals.

So we extended Resque with an optional `PRE_TERM_TIMEOUT` length that caused the parent to wait before sending the TERM signal to the child (A quick overview of the [changes](https://github.com/ejlangev/resque/commit/296edf23d9c09e81c1bb465e9633cd461c248a3b)).

This was a good start, but didn’t actually resolve the problem. Since Heroku sends TERM signals to EVERY process running on a dyno (and a Resque worker is just a dyno), the parent and child processes would receive the signal at the same time despite our modification to delay the parent from sending it to the child.

Unfortunately, a few more changes were needed to get this working.

### Our actual solution

After evaluating all our options, including swapping out Resque in favor of [Sidekiq](http://sidekiq.org/), we determined that the best solution was to add another small modification to Resque.

We needed to have the child process be able to ignore `TERM` signals from Heroku but let the parent process command it to die in keeping with normal Resque signal handling.

Our solution was to add another configuration option to Resque. This time called `TERM_CHILD_SIGNAL` which controls what signal is sent from the parent to the child when the parent is telling the child to die gracefully. To complete the configuration we set our `TERM_CHILD_SIGNAL` to be `QUIT` (rather than the default `TERM`).

Our updated procfile looked like this:

{% highlight ruby %}
worker: env PRE_TERM_TIMEOUT=5 TERM_CHILD_SIGNAL=QUIT TERM_CHILD=1 bundle exec rake resque:work QUEUES=*
{% endhighlight %}

The effect of doing this is to have the child ignore `TERM` signals which are sent by Heroku but not ignore `QUIT` signals sent by the parent process. Now receiving `QUIT` causes `Resque::TermException` (somewhat confusingly named unfortunately) to be raised in the child which plugs directly into any retry logic we may have implemented. That set of changes is visible [here](https://github.com/ejlangev/resque/commit/e285130bedca1305e33eaf5a0677fc435ca859dd).

The best part of this approach is that no modifications need to be made to the job itself.

## Conclusion

While it certainly isn’t desirable to fork your own version of a gem (we submitted [this PR](https://github.com/resque/resque/pull/1205) to resque but it hasn’t been merged, update it was merged [here](https://github.com/resque/resque/pull/1514)), in the short term it can serve as a passable solution to a tough problem like this.

While this isn’t the only possible way to solve this problem, it is a way that worked fairly well for us (though not perfectly). We still get sporadic errors from jobs taking longer than the `PRE_TERM_TIMEOUT` to finish and therefore being killed. However, the volume of errors is significantly reduced from where it was.

Despite a lot of searching around, we couldn’t find much discussion about people facing a similar issue, though it is inevitable that this problem would crop up given how Heroku works.
