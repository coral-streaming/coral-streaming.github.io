---
title: The event log
layout: default
topic: Documentation
topic_order: 2
order: 3
---
<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

## The event log

  - [Introduction](#introduction)
  - [Costs](#costs)
  - [Structure](#structure)
  - [Example](#example)
  - [Persistence](#persistence)
  
<br>

### <a name="introduction"></a>Introduction

The event log is a Coral feature that makes it possible to log incoming and outgoing events for every actor in a runtime. To enable the event log in your runtime, you first need to make sure that the event log is enabled on the platform. 

In your Coral configuration file, set the following value:

{% highlight json %}
coral.cassandra.event-log.enabled = true
{% endhighlight %}

Additionally, `coral.cassandra.event-log.table` should be set to a valid table name. For the event log to work properly, a valid connection to Cassandra must be provided.

If this is all set up properly, it is now possible to enable the event log in your runtime. 
The event log is configured per actor, and can be enabled for the following types of events:

- **Incoming messages**  
Enable incoming messages for an actor if you want to use the event log for state recalculation through *spark-batch* or *log-batch*. You cannot enable *spark-batch* or *log-batch* for an actor which does not have incoming event logging enabled. 

- **Outgoing messages**  
Enable outgoing message logging for an actor if you want to use the event log for the following reasons: 
  - *Manual analysis*  
    You can perform offline analysis by logging outgoing events. For instance, if you have an actor that calculates a boolean field `suspicious` and marks all suspicious transactions with `true` and the others with `false`, you can perform an offline analysis to investigate properties of suspicious transactions in a Spark job.
  - *Logging and debugging*  
     The event log can be used to find out how fast events are processed, which actors generate exactly which events and what decisions were made by the runtime.
  - *Lineage and audit requirements*  
    If there are requirements to log exactly what decision was made for what event, this can later be reconstructed with the event log.
  
Both incoming and outgoing messages are written to the same event log table in Cassandra. By default, both incoming and outgoing event logging are *disabled* for all actors.

### <a name="cost"></a>Costs

<div class="alert alert-warning" role="alert">
  <span class="glyphicon glyphicon-exclamation-sign" aria-hidden="true"></span>
  <span class="sr-only">Warning</span>
  <strong>NOTE</strong>: Event logging is expensive, turn it on only if you really need it, for the actors that really need it.
</div>

Please keep in mind that event logging can create a large load on your Cassandra instance and on Coral in general. If an actor receives 100.000 events per second, this means that 100.000 events per second will be written to the event log table if incoming messages are logged for an actor. If outgoing messages are also logged, there can be a maximum of 200.000 messages written to Cassandra per second, and this is just for a single actor. If we assume that a single event is 1 KB, a single actor alone will write 97 MB to the event log per second. To put this into perspective, this is almost 3 petabyte per year (3.079.687.500 MB)!

The number of messages becomes even bigger if you have multiple actors in your runtime and you enable event logging for incoming and outgoing events for all of them. It is therefore recommended to only turn on event logging for certain actors if you really need it, and to be very reluctant to enable incoming and outgoing event logging for all of your actors in your runtime. 

It is generally a good idea to think really hard about what you want to log. There are no hard rules, but the following guidelines could provide helpful:

- Do you need to calculate event-based and count-based snapshots for actors with *spark-batch* and *log-batch* enabled? Then turn on incoming event logging only for the actors which have these state functions enabled.
- Do you need to have a complete overview of all incoming messages in your runtime? Then turn on incoming event logging for all Kafka consumers in the runtime.
- Stateless actors generally do not need incoming or outgoing event logging. A stateless actor can always be re-executed to get its output given a certain input; for stateful actors this is more difficult since the output also depends on its state at that moment in time. 
- Do you need to satisfy lineage and/or audit requirements? Keep in mind that it is enough to simply log incoming messages and re-run the incoming messages with the same runtime definition to find out which decision was finally made by the runtime, if all actors in the runtime are stateless.
- Do you want to use the output that a certain actor generated for offline analysis? Then turn on outgoing event logging for that actor.
- Do you need to know which messages were sent to external systems (either Kafka or other API's)? Then turn on outgoing message logging for all Kafka producers and all HTTP client actors in your system.
- It makes no sense to enable outgoing event logging on the first actor and incoming event logging on the second actor if there is just a single connection between the two; the outgoing messages of the first actor will be identical to the incoming messages of the second actor.

Overall, try to reduce the number of logged messages to an absolute minimum.

### <a name="structure"></a>Structure

The log file table is structured as follows:

**Name** | **Type** | **Description**
logtype | int | 1 for incoming and 2 for outgoing messages
owner | text | the unique name of the owner of the runtime
runtime | text | the name of the runtime
actor | text | the name of the actor
day | int | the day, ranging from 1 to 31 in which the event was logged
hour | int | the hour, ranging from 0 to 23 in which the event was logged
timestamp | bigint | the timestamp on which the event was logged
json | text | the JSON object in textual representation
fjson | text | the flattened JSON in textual representation

<br>

The primary key for the event log table is `(logtype, owner, runtime, actor, day, hour)`. This means that if you want to query for events spanning multiple hours, days, actors, runtimes or owners, you have to break the ranges up in separate queries.

### <a name="example"></a>Example

Below, an example of a fraction of an actor definition is shown, wich uses incoming event logging to calculate  spark batch every hour:

{% highlight json %}
{
  "log": {
    "incoming": true,
    "outgoing": false
  }, "state": {
    "spark-batch": {
      "enabled": true,
      "update": {
        "duration": {
          "hours": 1
        }
      }
    }
  }
}
{% endhighlight %}

Note that it would have been an invalid runtime definition if `incoming` was set to `false` and `spark-batch` was enabled for this actor.

### <a name="persistence"></a>Event log and persistence

As described in the section about [state calculation](Documentation-StateCalculation.html), the event log can be turned on or off for individual actors, and for incoming and outgoing messages separately. The event log is meant as a mechanism to store events to perform long-running state calculations on and to make offline analysis possible. The event log, however, is not meant as a mechanism for fast recovery after an actor, runtime or machine crashes.

For this reason, Coral implements persistency for resiliency purposes separately from the event log. In contrast to the event log, Akka persistence cannot be turned on or off for individual actors, and is always enabled for all actors. It can be turned on or off for the entire platform, however. 

The number of events and the number of snapshots stored in the persistency tables can be configured, but it is recommended not to extend the storage period for the persistency tables beyond the minimum amount of time needed to store a single snapshot and all events after that. For instance, if an actor is configured to calculate a new snapshot every hour, it is sufficient to configure persistency to keep data for a little longer than an hour. Older snapshots can safely be discarded.

Keep in mind that persistency is turned on for all actors by default, meaning that the size of the storage can grow very fast. It is therefore recommended to keep the time to store persistency snapshots and events as short as possible.
