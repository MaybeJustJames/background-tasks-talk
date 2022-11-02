# Which to ski?
#### A software architecture dilemma
![Mountains](Nature.jpg)

Note:
Today I want to take you on a tour of these slopes. And by the end you will have no idea
which one to ski.

---

![Brace yourselves](brace-yourself.jpg)

Note:
First I'm going to tell you how to ski, then I'll describe what you might
want from a good ski slope, then I'll tell you about the available slopes,
and finally put you to task and push you off.

---

In an _event-driven_ application, there is generally a _main loop_ that listens for events and calls
a handler function when an event is detected.

```rust
loop {
    let event = listen();
    match (event.name) {
        "Lunch time" => wake_up(),
        "Team meeting" => be_late(),
        "Call from Alex" => pretend_to_work(),
        _ => relax(),
    }
}
```

Note:
Here is a simplified program encoding me. Or my main event loop. All my time is spent relaxing
and waiting for events. When I "hear" an event I react to it (or "handle" it) then I go back
to listening and relaxing.

---

# Examples

* A web server / service

* Any GUI application (e.g. Android apps)

* Device drivers

* Browser applications

Note:
Many real programs that we might be interested in are event driven.

---

## "Background" tasks?

Everything I've talked about so far are "foreground" tasks.

A "background" task is any computation that happens independent of
the _main (event) loop_.

Note:
I'm sure you're already aware of what a background task is, I just want to draw a complete picture.
From the perspective of the _main loop_ any task not handling an event is a background task.
Background tasks necessarily run concurrently with the main loop.

---

# Examples

* Processing large CSV files in a web browser

* Checking if the contents of a form field are valid PDBIDs in an Android application

* Computing protein tertiary structure from sequence in a web service

---

# Why?

Executing long-running computations in the "foreground" can cause problems...

```python
def mouse_driver():
    event = listen()
    if event == "move_cursor":
        mine_cryptocurrency()
        send_move_event_to_os()
    elif: ...
```

Note:
To take a visceral example. Imagine a mouse driver that does a little crypto mining on every mouse
event it receives from the hardware. This might result in a jittery mouse using experience followed
rapidly by a broken mouse, tears, screaming,...

---

# When?

The problem is that many interesting computations we might want to perform take time.

![Foreground tasks](https://devcenter1.assets.heroku.com/article-images/310-imported-1443570180-Screen-20shot-202012-04-12-20at-202.49.38-20PM.png)

Often more time than we can allow.

---

So we need a way to push these computations to the
"background" while we continue to get on with the latency sensitive work of the main
loop in the "foreground".

![Background tasks](https://devcenter0.assets.heroku.com/article-images/310-imported-1443570182-Screen-20shot-202012-04-12-20at-203.59.12-20PM.png)

Note:
A "background" task processing service is responsible for carrying out work scheduled
by the web service. Once it has finished a work item it can persist the results and
move onto the next one (or wait until another work item is available).

Depending on requirements and design, this idea can scale from device drivers to compute grids.

---

## Types of background work

Note:
It's useful to distinguish 3 schedules for background work.

---

* Stuff that has to happen _**now**_!

![Do it](https://cdn-images-1.medium.com/max/1600/1*2rK6gBMkTSyTEYZ6YHRrlQ.jpeg)

---

* Stuff that has to happen at a exact specific later time

![Tick tock](ticktock.png)

---

* Stuff that just has to happen sometime later

![Do it later](https://devblogs.microsoft.com/startups/wp-content/uploads/sites/66/2020/10/image1.jpg)

---

## What about features/requirements?

- Monitoring
- Error handling
- Accuracy/latency
- Ordering
- Complexity
- Cost
- Configuration for a specific use-case
- Code sharing / application organisation
- Scaling

Notes:
What requirements might we have?

- Monitoring:
    - How many jobs are in the queue?
    - Is the system crashing?
    - How long is it taking for background tasks to complete?
    - What happens to records of work items once they've been processed?

- Error handling:
    - What happens if there are too many jobs?
    - What happens if the system crashes while processing a job?
    - What if the network goes down or a HD dies?

- Accuracy/latency:
    - If I schedule a job to run at 10AM, does it run at exactly 10AM?
    - If I request a job to run immediately, how long does it take to start running?

- Ordering:
    - Do I care about job execution ordering?

- Complexity:
    - How complex is it to implement/use the system?
    - Can the system adapt to my needs?

- Cost:
    - Is there an up front cost? (time, money)
    - Is there an operational cost? (time, money)
    - What's the trade-off between these am I willing to accept?

---

### Common pattern: Message queues

![Message queue](https://stiller.blog/wp-content/uploads/2020/02/RabbitMQ-vs-Kafka-Message-Queuing.svg)

Notes:
Producers place work items onto a shared queue then they can go back to whatever they need to do.
Eventually, a worker ("Consumer") will remove the work item from the queue and process it.

---

## Common pattern: Pub/Sub

![PubSub](https://stiller.blog/wp-content/uploads/2020/02/RabbitMQ-vs-Kafka-PubSub.svg)

Note:
Similar to a message queue but a single message can be processed by multiple workers ("Consumers").

---
<!-- .slide: data-background="https://www.allwhitebackground.com/wp-content/uploads/5/Nature.jpg" -->
# A tour of the slopes

---

# DIY

![DIY](https://www.theonering.com/wp-content/uploads/2020/11/pippinmerry011128a.jpg)

Note:
- Example:
    1. Create a data structure to represent a queue, maybe protected by a mutex
    2. Spawn N threads to read from the queue
    3. Application exclusively write into the queue
    4. Worker threads vie to work on tasks in the queue

- Good:
    - Full control over semantics and features

- Bad:
    - Developer responsible for quality and features of a whole system

---

# CRON

![Cron](https://cdn-images-1.medium.com/max/1200/1*ct0QEPfiUl5PG7_Q_gpe_w.png)

Note:
- Example:
    - Application puts tasks in a database with a timestamp
    - CRON job reads tasks since last run and executes each one
    - possibly inserts results back into another database table

- Good:
    - Runs stuff at a set time or on a schedule
    - Run stuff later
    - Well understood and supported

- Bad:
    - 1 minute resulution
    - Error handling
    - Monitoring
    - Configuration and execution out of the application
    - Blocking (if your work runs for longer than the schedule you have a problem)

---

# Postgres LISTEN/NOTIFY

![Postgres](http://sql.sh/wp-content/uploads/2012/12/logo-postgresql-elephant.png)

Note:
- Example:
    - Application inserts tasks into a datbase table
    - Task executor listens for changes to the table
      and executes tasks as they're inserted

- Good:
    - Executes tasks as they're inserted into the task table (immediate execution)
    - Directly tied to data
    
- Bad:
    - Only Postgres
    - Have to write seperate task executor
    - No monitoring (or ad-hoc monitoring)
    - Error handling

---

# Redis Messaging

![Redis](http://thenewstack.io/wp-content/uploads/2015/03/redis-logo.png)

Notes:
Same as for Postgres

---

# Message queues
#### or
# Event streaming

![Celery](http://www.pngmart.com/files/5/Celery-PNG-Clipart.png)

Note:
- Good:
    - Purpose designed to run background tasks
    - Monitoring and error handling built-in
    - Task executor is a seperate process
    - Control over task semantics: schedule, ordering, complexity, etc.

- Bad:
    - Have to run/manage the message queue system
    - Configuration
    - Some more appropriate for certain tasks than others (no 1 size fits all)
    - Executor is a seperate application (though can share source which requires careful project organisation)

- Examples:
    - RabbitMQ
    - ZeroMQ | NanoMsg
    - Celery
    - ActiveMQ
    - Apache Kafka
    - Beanstalk (demo)

* https://kafka.apache.org/intro
* https://stiller.blog/2020/02/rabbitmq-vs-kafka-an-architects-dilemma-part-1/

---

# Framework provisions

![Elixir](https://www.alwaysdata.com/static/img/technologies/languages/elixir.png)

Note:
- Good:
    - Built into your language or application framework
    - Often give you the semantics you need from a message queue
    - Share application code
    - Monitoring and error handling built-in
    
- Bad:
    - Sometimes require an extra dependency (needs to be kept up-to-date; may not play well with chosen framework)
    - May just be a convenient API to one of the previous discussed systems (so comes with the baggage too)

- Examples:
    - Elixir `GenServer`
    - Java/Spring `@Scheduled`
    - Python/FastAPI `BackgroundTasks`
    - Ruby/Rails `SideKiq`, `delayed_job`, ...
    - Haskell `OddJobs`, `hworker`, `jobqueue`, ...
    - PHP/Laravel `Queues`
    - NodeJS/Ember `Ember.run.schedule`, `Ember.run.later`
    - Rust/Tokio `Channel`

- Links:
    - https://www.haskelltutorials.com/odd-jobs/haskell-job-queues-ultimate-guide.html
    - https://fastapi.tiangolo.com/tutorial/background-tasks/

---

# SaaS / MaaS

<iframe src="https://giphy.com/embed/cPSeb8U6cjALDo0GGL" width="480" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

Note:
- Good:
    - No need to manage any of the task management or queing ourselves
    - Built-in monitoring and error handling as well as scaling
    - All useful semantics from a message queue

- Bad:
    - Expensive up front
    - Requires internet access
    - Internet latency and unreliability
    - Give commercial access to our data
    - Set up workers on SaaS is seperate from a single deploy
    - Seperate monitoring and error handling
    - Requires authentication (seperate concern from scheduling a background task)
    - Tied to a SaaS task queue provider
    - May not be a good fit for the specific use-case

- Examples:
    - Amazon Simple Queue Service
    - Azure Service Bus
    - Ralley

* https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing

---

# PaaS

<iframe src="https://giphy.com/embed/1hMjFZY16VxGWrfDfq" width="480" height="270" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

Note:
- Good:
    - Dont have to run/manage the task queue system
    - Built-in monitoring, error handling, scaling
    - All useful semantics from a message queue
    - Minimal latency
    - Access to all the compute power anyone might need (for a price)

- Bad:
    - Expensive up front
    - Tied to a cloud provider
    - May not be a good fit for the specific use-case

- Examples:
    - AWS Kinesis Data / AWS Elastic Beanstalk
    - Heroku Worker Dynos
    - Azure Event Hubs / Azure Batch
    - Google Cloud Tasks
    - ...

* https://devcenter.heroku.com/articles/background-jobs-queueing

---

## "The Cloud" has a problem

#### Let a politician explain...

<iframe width="560" height="315" src="https://www.youtube.com/embed/PBaZAcBWwik?start=28" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

# Sun Grid Engine

<iframe src="https://giphy.com/embed/l0Iympgkss7fnFvDa" width="480" height="240" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

Note:
Now called Oracle Grid Engine, Son of Grid Engine, Open Grid Scheduler, Univa Grid Engine.

You've probably noticed that I havn't mentioned SGE yet. That's because it's purpose
is related but tangiential to this topic. The key is the word "grid": SGE schedules and
dispatches jobs onto a distributed computer. We _may_ want that behaviour from our
background tasks but it should be abstracted from such a system.

---

<iframe src="https://giphy.com/embed/d2eNXDl6ZkpOIqfhG6" width="480" height="264" frameBorder="0" class="giphy-embed" allowFullScreen></iframe>

---

![This mountain](Here.png)
