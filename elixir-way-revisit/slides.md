---
# try also 'default' to start simple
# theme: apple-basic
theme: seriph

# layout: intro
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
---

# Elixir Way by revisiting a project


@gzzengwei

---

# Starting

- ๐งโ๐ป **About Me**
  * long time rubist
  * like engineering/ops stuffs
  * getting into FP/elixir

<v-click>

- ๐คน **About this talk**
  * an ealry elixir project
  * revisiting / improvement

</v-click>

<v-click>

- ๐ฅ **Disclaimer**
  * learning in progress
  * spark/poc, not all verified in production

</v-click>

<!--
- Did my first few elixir projects as kind of experiments
- Do full time elixir for around a year
- Reviewing the projects I done, Wondering what approach I would do if I do it again
- The project is relativly simple yet have real world requuirements
- During the spark, I think the journey might be a good topic for beginner to get to know elixir


- entry level, in production the config of the params will be more complex and could take long time to config
-->

---

# Project backgrounds

- ๐ An importer for CSV files from external source
- ๐ Data source is updated daily (cronjob)
- ๐ There are up to 50+ csv types (different job types)
- ๐ Data is to dump to db directly for BI purpose (downstream target)
- ๐ CSV schema changes occasionally (exceptions could happen)
- ๐บ A simple dashboard is required to integrate in main app (web interface)


---

# Choosing the stack

<br />

<p style="text-align: center;">
  <h2> Ruby / Elixir </h2>
</p>

<!--

- multi csv types: job workers

- A UI dashboard: web service

ruby
  - sidekiq/activeJob
  - rails/sinatra
  - Redis

-->

---

# Demo

- ๐ป Simple job process service
- ๐ท Dummy code to demo some of elixir basic modules

<br />

<v-click>

#### So, we are going to start with something very simple ...

</v-click>

<!--

Terminal:

git checkout v01

- explain job/job_manager/supervisor
- run happy/bad path
- enable Scheduler, try both paths

Issues:
- sequential takes time, waste infra resouces
- exeption will break rest of the loop, also the caller process

-->

---

# Task

- ๐ conveniences for spawning and awaiting tasks
- ๐บ execute one particular action throughout their lifetime
- ๐ฆ little or no communication with other processes
- ๐๐พโโ๏ธ convert sequential code into concurrent code by computing a value asynchronously
- Simple usage: `start` and `async`

<!--

Terminal:

git checkout v02

- changes: `Task.start`
- run happy/bad path
- enable Scheduler, try both path
- sometimes will need task results, also a place to store the state

-->

---

# GenServer

- โน๐ฟโโ๏ธ A behaviour module for implementing the server of a client-server relation.
- ๐ฆ A process to keep state, execute code asynchronously and so on
- ๐ซ Standard set of interface functions, includes tracing and error reporting

<!--

My understanding
- Implemetation of Actor Model
- elixir is FP languange, and GenServer is goto place store and due with state
- Communicate with other process by exchange of sending messages

Terminal:

git checkout v03

- changes:
  - Application -> enable JobManager in supervisor children
  - JobManager -> GenServer
  - Task.start -> Task.async
  - Save task results to state
  - Job -> has a return value

- run happy path, then check state

:sys.get_state(JobDemo.JobManager)

- run bad path, show: jobs not finished/JobManger restarted
- uncomment job.ex s rescue block to show error handling

-->

---

# Issues

- Often we are running many processes, like many Job types in the example
- When single process dies, and it should not effects others, and its parent process
- process should be isolated when crashed, and restart

---

# Supervisor

- ๐ Supervises other processes
- ๐ฒ supervision tree: hierarchical process structure
- ๐ฅ provide fault-tolerance/encapsulate how app start/shutdown

---

# Structure Diagram

<div grid="~ cols-2 gap-4" m="-t-2">
<div>

**Current supervision tree**

```mermaid {theme: 'forest', scale: 0.7}
graph TD
A([Application]) --> B(Scheduler)
A([Application]) --> C(JobManager)
C -->|run task| D[JobA]
C -->|run task| E[JobB]
C -->|run task| F[JobC]
```

</div>

<v-click>

<div>

**With Supervisor**

```mermaid {theme: 'forest', scale: 0.7}
graph TD
A([Application]) --> B(Scheduler)
A([Application]) --> C(JobManager)

C --> D{{Supervisor}}
C --> E{{Supervisor}}
C --> F{{Supervisor}}
D --> G(JobA)
E --> H(JobB)
F --> I(JobC)
```
</div>

</v-click>

</div>

<!--
Terminal:

git checkout v04

- changes:
  - Application -> DynamicSupervisor
  - creates JobSuperviosr, restart polices
  - Job -> GenServer
  - JobManager init -> start JobSuperviosr via DynamicSupervisor

:sys.get_state(JobDemo.JobManager)

- run happy/bad path, to show JobManager not effected
:sys.get_state(JobDemo.JobA) # show last_done
:sys.get_state(JobDemo.JobB) # show nil last_done
-->


---

# New Issue

- ๐ฅ Everything works fine, until ..
- ๐ฆ A big tenant has large csv file(s) in both size and qunatity
- ๐ฅ Hit by DB error

```
(DBConnection.ConnectionError socket closed (the connection was closed by the pool,
possibly due to a timeout or because the pool has been terminated))
```
- ๐ฟ Data reach peak once cronjob triggers, and flood the DB with all the concurrent tasks
- ๐ฆ DB simple cannot handle the load

---

# Revisit the workload

- ๐ sequential: taking too long and waste of infra resouces
- ๐ concurrent without control: too much pressures on downstream service in short time

<v-click>

- ๐ธ The job actually split into 2 parts
  - **Upstream**: Data fetching(read rows from csv file(s))
  - **Downstream**: Import the rows to db

</v-click>

<v-click>

- ๐ซ Split the workflow to (multi) stages that we have control of

</v-click>

---

# GenStage

- ๐ธ data-exchange steps that send and/or receive data from other stages
- ๐ฅฝ producer: stage sends data; consumer: it receives data;
- ๐ฆท some stages can be both producer and consumer
- ๐ธ back-pressure mechanism: consumer is sending demand upstream, the producer will emit items


<v-click>

**Data flow:**

[Producer] -- data --> [ProducerConsumer] -- data --> [Consumer]

</v-click>

<v-click>

**Demand flow:**

[Producer] <-- ask for data -- [ProducerConsumer] <-- ask for data -- [Consumer]

</v-click>

---

# Let's do some spark

- ๐? **producer** : sending rows from csv file(s)
- ๐ข **consumer** : Importing data to db
- ๐ฏ issue: data source is passive(trigger by cronjob)
- ๐ we need something in between reading data in files and emitting data


---

# First version: Pubsub

- ๐ Why Pubsub
- ๐ฃ A buffer between Job/Producer

```mermaid {theme: 'forest', scale: 0.5}
sequenceDiagram
  participant ExternalSource
  participant JobManager
  participant Supervisor
  participant Job
  participant PubSub
  participant Producer
  participant Consumer
  participant Database

  Note over Producer,Consumer: GenStage

  JobManager ->> Supervisor: spin up
  Supervisor ->> Job: supervise
  Consumer -->> Producer: Subscribe
  Producer -->> PubSub: Subscribe
  ExternalSource ->> Job: read rows from source
  Job ->> PubSub: broadcast data in batch
  PubSub ->> Producer: push data
  Producer ->> Consumer: emit data
  Consumer ->> Database: import

```

<!--

Why? I havn't read the docs throuthoroughly enough

Terminal:

git checkout v05

- changes:
  - Application -> PubSub/Producer/Consumer
  - Job ->
    - run: use stream
    - pubsub: broadcast
  - JobType: simulate range/data
  - Producer: subscribe to PubSub
  - Consumer: subscribe to Producer/ use sleep simulate run

- run happy path to show consumer log
- run bad path to show JobA/C works
- update job range to 1000s to show discards events
- update application consumer number and rerun

-->

---

# Buffering demand

- ๐ฆ Handle cases that:
  - events arrive and there are no consumers
  - consumers send demand and there are not enough events
- ๐ด link: https://hexdocs.pm/gen_stage/GenStage.html#module-buffering-demand

## BroadcastDispatcher

- ๐ Accumulates demand from all consumers before broadcasting events to all of them.
- ๐ฆข Guarantees that events are dispatched to all consumers within demand range
- Advanced features like `:selector` consumer only subscribe to certain events


<!--

Terminal:

git checkout v06

- changes:
  - remove PubSub
  - Producer: GenStage.BroadcastDispatcher
  - Consumer: subscribe to Producer/ use sleep simulate run


- run happy path to show consumer log
- run bad path to show JobA/C works
- update job range to 1000s to show discards events
- update application consumer number and rerun
- update producer

  def init(state) do
    {:producer, state, dispatcher: GenStage.BroadcastDispatcher, buffer_size: :infinity}
  end

-->

---

# Buffering demand (cont.)

## BroadcastDispatcher with queue

- ๐ Use erlang `:queue` module (FIFO)
- ๐ฅ Control over the events and demand by tracking this data locally

<v-click>

```mermaid {theme: 'forest', scale: 0.5}
sequenceDiagram
  participant ExternalSource
  participant JobManager
  participant Supervisor
  participant Job
  participant Producer
  participant Consumer
  participant Database

  Note over Producer,Consumer: GenStage

  JobManager ->> Supervisor: spin up
  Supervisor ->> Job: supervise
  Consumer -->> Producer: Subscribe
  ExternalSource ->> Job: read rows from source
  Job ->> Producer: notify events
  Producer ->> Producer: queue up events
  Consumer ->>+ Producer: demand
  Producer -->>- Consumer: emit events
  Consumer ->> Database: import

```

</v-click>

<!--

Terminal:

git checkout v07

- changes:
  - Producer:
    - init: new erlang :queue instance
    - handle_call(:notify):
      - add incoming event to queue
      - dispatch pending events in state
    - handle_demand: dispatch incoming and pending events


- run happy path to show consumer log
- run bad path to show JobA/C works
- update job range to 1000s to show discards events
- update application consumer number and rerun
- update producer

  def init(state) do
    {:producer, state, dispatcher: GenStage.BroadcastDispatcher, buffer_size: :infinity}
  end

-->

---

# Summary

- ๐ฅฏ More understanding Elixir tools
- ๐ Tools are powerful but they might require learning curve
- ๐ Confidence in its performance

<v-click>

- ๐? More to explorer
  - Broadway: Concurrent and multi-stage data ingestion and data processing
  - Commanded: CQRS/EventSourcing

</v-click>

<br />

<v-click>

### Reference

Book by *Svilen Gospodinov*, **Concurrent Data Processing in Elixir**

</v-click>

---
layout: center
class: text-center
---

# Thank You

### Questions ?
