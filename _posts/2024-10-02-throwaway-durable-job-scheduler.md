---
title: Rolling My Own Throwaway Durable Job Scheduler in 50 Lines of Code
tags: [product development, software development, python, gevent]
style: 
color: warning
description: Don't try this at home.
---

For [Bite Ballot](https://www.biteballot.app), I wanted to add a feature where ballots could be set to end at a set time instead of when everyone has submitted votes. It sounds simple enough, but the catch here is that it has to be **durable.** Ballots may end hours in the future. What if I need to restart the backend to deploy an update during that time? When it starts back up, the ballot should still be scheduled to end at the right time.

## Weighing the options

![rabbit_and_celery]({{ site.baseurl }}/assets/image/rabbit_and_celery.jpg)

My first instinct was to leverage an existing solution like Celery with RabbitMQ, but this would require managing at least one more process just to support one small feature. My backend was currently monolithic with all the advantages that brings in terms of simplicity. And as for the overall product, it was still in the MVP / product-market-fit stage.

> Do things that don't scale.

The advice from Y Combinator echoed in my head. Scalability is great, but it takes planning, development time, and an increase in code complexity to get right. If you don't even know if your product is worth anything yet, just getting something *done* and in the hands of users for feedback is way more valuable. You can always make things scalable if and when you actually need it.

## Making a choice

So, *could I get away with a simpler solution that doesn't scale?*

Well, my backend already had a dependency on [gevent](https://www.gevent.org/), which I was using to implement [long-polling](/projects/1-bite-ballot#long-polling). It, in turn, has a dependency on [greenlet](https://pypi.org/project/greenlet/), a lightweight coroutine library, and it also has a function called `spawn_later`, which I could use to schedule a greenlet to run at a set time. Excellent.

The last piece of the durability puzzle was to store ballot end times on disk. That's easy enough. I could just add a column to the `ballot` table in my Postgres database.

With all my ducks in a row, it was now only a matter of implementation.

## Implementation

This solution doesn't scale because it's tightly coupled -- it's made specifically for scheduling ballot end times and can't be reused for anything else. It also wasn't designed to handle a huge load. But as long as I maintained a clean interface to it, I could easily swap it out later if necessary.

First, let's write a function called `schedule_end` that does what it sounds like it does. Looking at [spawn_later](https://www.gevent.org/api/gevent.html#gevent.spawn_later), we see it takes a delay in seconds. Given a `Ballot` that has an `end_time` in UTC, we can calculate the delay with:

```python
ballot.end_time - datetime.datetime.now(datetime.timezone.utc)
```

But there are a couple of wrinkles. A ballot can end early if two voters move to end voting. A ballot can also be deleted if all its voters leave. If we don't do something, the greenlet will run and an exception will be thrown due to being in one of these unexpected states.

We should probably just kill the greenlet when either of these things happen, but that means we need to keep a reference to it. Since there's always only one per ballot, and we'll always have a ballot ID when we need to kill it, a dictionary of ballot ID to greenlet is a natural fit:

```python
_greenlets = {}
```

We'll add greenlets to it when a ballot end time is scheduled, then we can kill one later with:

```python
def unschedule_end(ballot_id):
    if ballot_id in _greenlets:
        _greenlets[ballot_id].kill()
        del _greenlets[ballot_id]
```

When scheduling a greenlet, we give it a function to call. In my case, I'll call that function `_end_ballot()` and have it call `unschedule_end()` at the end to clean it out of the greenlet dictionary:

```python
def _end_ballot(ballot_id):
    with _app.app_context():
        ballot = db.session.execute(db.select(Ballot).where(Ballot.id == ballot_id)).scalar()
        result = ballot_algorithm.calculate_result(ballot)
        db.session.add(result)
        db.session.commit()
        unschedule_end(ballot_id)
```

Again, this is domain-specific code that ideally shouldn't be tightly coupled to the durable job scheduler. But this is throwaway code, and the development speedup at this point in the product lifecycle is worth the tradeoff. I'll just add a note about this at the top of the file.

#### Make it so

Now that the durability pieces are in place, let's actually make it durable. The ballot end times are being stored in the database on disk, but if the backend goes down, the greenlets won't execute. And when it starts up again, the greenlets will need to be rescheduled. To know when to schedule them, we'll need to query the database for ballots that have end times but no results yet.

But there's another wrinkle we have to think about first. It's possible that a ballot could have ended while the backend was down. In that case, we'll just want to end it immediately. Okay, *now* let's write our code:

```python
def init(app):
    global _app
    _app = app
    ending_ballots = db.session.execute(db.select(Ballot)
                                        .where(Ballot.result == None)
                                        .where(Ballot.end_time != None)).scalars()
    now = datetime.datetime.now(datetime.timezone.utc)
    for ballot in ending_ballots:
        delta: datetime.timedelta = ballot.end_time - now
        delta_seconds = delta.total_seconds()
        if delta_seconds <= 0:
            _end_ballot(ballot.id)
        else:
            _schedule_end(ballot, delta_seconds)
```

And that's it! We're done. Here's the entire file:

```python
"""
A durable job scheduler for ending ballots that have a set end time.
"""
# This is throwaway code. If this durable job scheduler needs to be extended or reused for other purposes,
# we should probably just switch to using something like RabbitMQ + Celery.

import gevent
import datetime

from flask import Flask

from . import ballot_algorithm
from .model import *

_app: Flask
_greenlets = {}


def init(app):
    global _app
    _app = app
    ending_ballots = db.session.execute(db.select(Ballot)
                                        .where(Ballot.result == None)
                                        .where(Ballot.end_time != None)).scalars()
    now = datetime.datetime.now(datetime.timezone.utc)
    for ballot in ending_ballots:
        delta: datetime.timedelta = ballot.end_time - now
        delta_seconds = delta.total_seconds()
        if delta_seconds <= 0:
            _end_ballot(ballot.id)
        else:
            _schedule_end(ballot, delta_seconds)


def schedule_end(ballot: Ballot):
    now = datetime.datetime.now(datetime.timezone.utc)
    delta: datetime.timedelta = ballot.end_time - now
    _schedule_end(ballot, delta.total_seconds())


def _schedule_end(ballot: Ballot, delta_seconds):
    greenlet = gevent.spawn_later(delta_seconds, _end_ballot, ballot.id)
    _greenlets[ballot.id] = greenlet


def unschedule_end(ballot_id):
    if ballot_id in _greenlets:
        _greenlets[ballot_id].kill()
        del _greenlets[ballot_id]


def _end_ballot(ballot_id):
    with _app.app_context():
        ballot = db.session.execute(db.select(Ballot).where(Ballot.id == ballot_id)).scalar()
        result = ballot_algorithm.calculate_result(ballot)
        db.session.add(result)
        db.session.commit()
        unschedule_end(ballot_id)
```

It is:
- Readable
- Tightly-coupled
- Short
- Fast to develop
- Simple

Once again, do **NOT** write code like this into a mature product. (Hopefully, it would not pass code review anyway.) This code does what it needs to do for as long as it is useful. Then, it can and should be thrown away!

In a more mature product, the code should generally be modular and follow the **Single-Responsibility Principle (SRP)**. That has the cost of increasing the file count and overall line count, but the benefit is that each individual module will be short and easy to understand and change in isolation. This is **critically important** when working on a team on a huge, complex codebase.

## Usage

`init()` is called during Flask startup. `schedule_end()` is called whenever an end time is picked for a ballot. `unschedule_end()` is called when a ballot has been moved to end or when the last voter has left a ballot.

## Testing

I tested all the possible scenarios outlined above:

1. Ballot ends normally.
2. Ballot ends while backend is down.
3. Ballot ends after backend is restarted.
4. Ballot ends by moving to end, then the original end time is passed.
5. Ballot is deleted because all voters left, then the original end time is passed.

They all functioned correctly.

## Conclusion

Sometimes you need something quick and dirty. With this code, I was able to implement the needed functionality to test the usefulness of the feature very quickly without significantly increasing the complexity of my codebase nor the complexity of the administration of the backend. But critically, its addition also didn't hurt maintainability of the overall codebase because it was isolated to a single file and has a clean interface.

Anyway, don't try this at home.

Or rather, try it at home, but not at work.
