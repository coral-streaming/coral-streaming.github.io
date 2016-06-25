---
title: State calculation
layout: default
topic: Documentation
topic_order: 3
order: 4
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

## State calculation

- [Actor types](#actortypes)
- [State: The Problem](#stateproblem)
- [Local state calculation](#localstate)
- [Local batch state calculation](#localbatchstate)
- [Spark state calculation](#sparkstate)
- [Log state calculation](#logstate)
- [Interplay between different modes](#interplay)
- [Finally](#finally)

<br>

In Coral (and in Akka in general), actors can contain *state*, which is any variable (or multiple variables) that can be updated when a new event comes in. In Akka, these variables are only visible and changeable by the actor in which the variable resides, and the variable can only be updated as a reaction on messages that were sent to the actor.

In Coral, there exists several methods to deal with state. To facilitate state calculations with Spark and the event log, several additional methods can be configured. These methods, and the relationships between them, are described on this page. 

--------------------------------

### <a name="actortypes"></a>Actor types

There are two types of actors in Coral:

- **Stateless actors**  
  These actors do not update any variables in reaction to incoming events, but only apply a function to the incoming JSON object, that determines its reaction. A typical example of a stateless actor is the [Threshold actor](Actors-ThresholdActor.html), which has two parameters: the field to check and the threshold amount. For example, the field to check might be "field1", and the threshold amount might be 100. This actor looks into "field1" of every incoming JSON object and determines if it is smaller or lager than 100. If it is smaller, it does not emit the event. If it is larger, it does emit the event.

  As is clear from the example, the state of the threshold actor is not influenced by incoming JSON events. Its state is constant; its threshold value will always be 100, regardless of the events that come through. These type of actors can be very easily scaled because the definition of the actor is constant and no effort has to be made to recalculate or synchronize state between actor copies.

- **Stateful actors**  
  This type of actors do somehow change internal variables when events come in. In a typical actor system, this often takes the form of a `var` field, which is then updated based on the incoming message. This approach is fine and is also available in Coral, but it has several drawbacks related to scalability and resiliency. 

  For instance, if the actor crashes, the state is lost. This can be mitigated by using a persistent actor with event sourcing and snapshots, but even in this case it is still hard to distribute the state across multiple actors to increase the throughput of the system. It is also unclear how the states of different copies of the same actor relate to each other if they have received different events. 

In Coral, state is handled through four different *state functions*, which are procedures that are executed depending on the settings of the specific actor. Each fo these state functions can be turned on or off independently. First, however, we continue to describe the problem of state in the Coral platform by describing several potential solutions to this problem.

--------------------------------

### <a name="stateproblem"></a>State: The Problem
To illustrate some of the potential problems that can arise when dealing with stateful actors in Coral, we first show a couple of possible solutions to the state problem.

<br>

####  <a name="solution1"></a>Solution 1: Ignore the problem (<a href="img/state1.png" target="_blank">show diagram</a>)

To start, let us pretend that there does not exist a problem, and we create a runtime on a machine, containing an actor that calculates the average of a certain JSON field. After a while, the load starts to increase, and more capacity is needed to handle the increased load. We can simply start up another runtime with the exact same configuration, and perform load balancing between the two runtimes. This means that the first machine sees only half of the messages, and the second machine sees the other half. 

Coral will not complain if you start two identical runtimes, but it might not give desirable results. In the picture below, machine 1 has processed events 1 and 2, and comes up with an average of 5. Machine 2, which runs the exact same runtime, has processed events 3 and 4 and comes up with an average of 10. 

There are several problems with this approach:

- **Which average is right?**  
There now exists inconsistent state between the two runtimes. You can rely on the statistics of large numbers here and assume that in the long run, the two averages will converge to the average of the original values, on average. If you don't want to sound like a statistician, this explanation is better avoided with another state method.

- **Reconciliation**  
How can we reconcile the state of the two actors? It is often not possible to aggregate two aggregates and come to the same results as when the calculation was performed directly on the original input events. For some types of state, such as a neural network for example, it might be completely unclear how to reconcile two states or it might simply be impossible.

- **Duplicate actions**  
For instance, if a JSON object is sent to Kafka if a certain threshold is reached, two JSON objects will be sent because we have two copies of the runtime. It is not clear how to prevent duplicate actions in the case of duplicate runtimes.

<br>

#### Solution 2: One runtime, one machine (<a href="img/state2.png" target="_blank">show diagram</a>)

A second solution is to always start a runtime on a single machine. This prevents inconsistencies in state between runtimes, but the effect is that the runtime cannot be scaled up beyond a single machine to handle larger loads. Also, if the runtime crashes, there is no live backup available to take over. This means that if a machine or a runtime crashes, it takes some time before a restarted copy is back online. In the meantime, no messages can be processed. This might or might not be a problem, depending on the specific use case and on the required recovery time.

<br>

#### Solution 3: Exact duplication (<a href="img/state3.png" target="_blank">show diagram</a>)

Instead of balancing messages to different instances of runtimes (*load balancing*), such as in Solution 1, each message could also be sent to every instance (*broadcasting*). This has the result that each instance sees all data and thus has identical state to other instances. This solution provides for a "live backup" of a runtime, but there is no effective scaling because the number of messages scales proportionally with the number of runtime copies on the platform. It is also unclear how duplicate actions should be handled, when, for instance, a JSON object is written to a text file when a certain threshold is reached.

<br>

#### Solution 4: State synchronization (<a href="img/state4.png" target="_blank">show diagram</a>)

Another solution is to synchronize the state of each actor with its copies. To achieve this, gossip messages could be sent to each copy after a number of received events. The problem with this approach is that possibly excessive synchronization traffic can occur because this synchronization happens point-to-point, between each pair of copies. It is also still unclear how two state variables, such as two averages, should be synchronized. In Coral, state synchronization with gossip messages is not available. 

<br>

#### Solution 5: Spark state calculation (<a href="img/state5.png" target="_blank">show diagram</a>)

Another option is to ask Spark for to recalculate state after a certain amount of time or a certain number of messages. At a certain point in time, the actor that needs to recalculate its state based on the event log asks Spark to start a new job, which has the following parameters:

- **Owner name**: The unique name of the owner;
- **Runtime name**: The name of the runtime;
- **Actor name**: The name of the actor in the runtime;
- **Begin timestamp**: The time from which events should be included in the calculation;
- **End timestamp**: The time until which events should be included in the calculation.
- **The piece of code to run**: The piece of code that returns a JSON object from the event log.

After a while, the code comes back with a new JSON object representing the new state of the actor. This approach is useful for large batches of events and heavy calculations that can be distributed with Spark. The advantage of this is that it becomes possible to calculate potentially complex new state on large batches of events in a distributed fashion across the Spark cluster. The disadvantage of this approach is that in the time between Spark state calculations, the state of the actors can not be updated with the incoming messages.

<br>

#### Solution 6: Spark state calculation + drift (<a href="img/state6.png" target="_blank">show diagram</a>)

If it is a requirement to incorporate the latest messages in the state of an actor, *state drift* could be accepted. In the context of Coral, state drift is the potential divergence of the state of different actor instances because of different incoming events. When a new Spark state calculation is performed, the state drift disappears and the states are fully synchronized again. State drift tends to increase the more time passes and/or the more messages are processed. Depending on the use case, this might or might not be a desirable method to calculate state.

--------------------------------

### The state functions

As the previous examples show, there are a couple of different ways to deal with state in Coral actors. Any of the behaviours above can be defined with the combination of 1+4 different "state modules", which are functions which define how to update the state of a certain actor. These functions are: 

1. (Trigger)
2. Local state calculation (*calc-local*)  
3. Local batch state calculation (*collect-local*)  
4. Spark state calculation (*spark-batch*)  
5. Log state calculation (*log-batch*)  

The trigger function is not strictly a state function, but is mentioned here because it is always executed after every JSON object is received and just before any one of the state functions. 

A Coral actor can implement a combination of the last four functions, or none of them, depending on the task of the actor. 

All four state functions ultimately achieve the same thing: they update the state of an actor (which can be a single variable or multiple variables) after processing either a single event or a batch of events. This can be done locally, on a locally stored batch, or it can be performed in a Spark job. The Cassandra event log can also be queried to recalculate state.

When an actor implements multiple ways to perform the same state update, it becomes possible to choose a method depending on the specific context of the actor or the specific runtime settings. For instance, if Spark is enabled, a Spark job can be started. If Spark is not available, the event log can still be queried directly to update state. If the event log is also not available, state updates can be performed on a local batch. If the local batch is turned off, ru-vars ([explained later](#ruvars)) can still be calculated.

#### Trigger

The first function, trigger, is not strictly a state function (hence the 1+4) because it does not modify any state. We mention it here because it is always executed after every JSON object, and the state functions are executed after it finishes, if they are turned on. The trigger function itself cannot be turned off and is always executed when a JSON object comes in. It contains the "scoring" part of the actor, or the code that is strictly functional/stateless. 

For instance, when the actor is a ThresholdActor, the trigger checks whether the specified field is lower than a certain threshold, and when it is, it passes on the message.

Coral actors never modify their internal state in the trigger function, but use the other four state functions to do this instead, if they implement these functions at all. It is thus safe to conclude that when no state functions are turned on in a runtime, the runtime can be considered fully functional/stateless and can be scaled up without any problems.

### <a name="localstate"></a>Local state calculation

The first state function, local state calculation/calc-local, makes all changes to local, internal variables that a Coral actor maintains. For instance, in an actor that counts the number of incoming events, its state could look as follows:

{% highlight scala %}
var count: Int = 0
{% endhighlight %}

In the trigger method of the actor, the following code could be found:

{% highlight scala %}
count += 1
{% endhighlight %}

This actor simply keeps a local counter variable that it increments on every trigger event. This approach works fine for runtimes which do not have to be scaled up and are perfectly capable to handle a reasonably low number of messages.

The drawback of this approach is that it is difficult to increase the throughput of a system which contains such an actor, since the state is located in a single actor. If there are too much messages coming in which the actor cannot handle (unlikely with such an easy calculation, but still), a large build-up of unprocessed messages will occur. 

When a copy of this actor is made (as seen in [Solution 1](#solution1)), and each actor copy is sent a JSON event in turn ("round-robin routing"), the end result will be that both actors keep a different count variable.

It is trivial to reconcile two actors that both keep a count: simply summing the counts will suffice.
However, the situation quickly becomes more complicated: when the state variables are two averages, or two medians, the action to reconcile them is not as obvious. When two actors keep more complicated state such as two neural networks, it might very well be impossible to define a function that merges the two states into a single one.

To reconcile the state between actors, a possibility is to send "gossip" messages between actor copies to reconcile the state after every event. The downside to this approach is that the resulting traffic can be so excessive that it completely eliminates any scaling benefits of using multiple actor copies. Because something like n(n - 1) messages need to be sent to fully synchronize state between all copies, it might just have been easier to send all messages to all copies instead! In this case, other state settings might be better suited. For single-instance actors and simple state, however, the local state calculation approach can work fine.

--------------------------------

### <a name="localbatchstate"></a>Local batch state calculation

Instead of updating the count variable in the previous example after every incoming event, a batch of events could also be saved, and the new state could be calculated after a certain amount of messages have been collected or a certain amount of time has passed. This mode is called local batch state calculation/collect-local.

For instance, a Coral actor runs for an hour, collects 100 messages, and after this hour, the new state is obtained by performing a calculation on the batch of events. After recalculation of the state, the batch is cleared and the process starts over again. 

In Coral, the function prototype that defines a local batch state calculation looks as follows:

{% highlight scala %}
type CollectLocal = BatchList => Unit
{% endhighlight %}

where BatchList is defined to be 

{% highlight scala %}
type BatchList = ListBuffer[(Long, JObject)]
{% endhighlight %}

meaning that it is a list that contains the timestamp when each object was received, and the object itself.
The CollectLocal definition returns Unit because the function has a side effect, i.e. updating the internal state.

Note that it is possible to update local state after every event *and* at the same time use local batch state calculation. If both options are enabled, both methods will simply be executed. There is no additional logic required of the user to make this happen; the Coral framework will make sure that state is updated properly according to the specified combination of platform and actor settings.

The local batch state calculation mode can still be used even if Spark or the event log are not available, since the actor simply collects events by itself. The disadvantage of collect-local is that it is again hard to scale it up to multiple actors, since each actor copy collects its own specific set of events if round-robin routing is enabled. When sending all events to all actor copies, each actor receives all events ("broadcasting"), so the problem of diverging state does not exist. However, there is no effective scaling either because the number of actors increases proportionally with the number of messages to process. 

### ru-vars and nru-vars
The reason that collect-local exists is that some calculations are impossible to perform if the original list of events is not available. A typical example of this is the median of a list of numbers. While it is possible to obtain a count by simply adding 1 to the old count, it is not possible to obtain a new median by altering the old median and somehow using the event that came in. In Coral, variables which can be updated without having the original list of input data are called "rolling update variables", or *ru-vars*. In contrast, a "non-rolling update variable", or *nru-var*, is a variable which cannot be calculated without the original input list. 

In practice, even though a variable could be expressed as a ru-var theoretically, it is sometimes infeasible to determine the algorithm to perform the rolling update. Therefore, it might be more practical to approach it as a nru-var instead. 

Another reason that the local batch state calculation method exists is that the Coral platform does not assume that Spark and the event log are available and enabled. Even if any of these is unavailable or turned off, the platform must still be able to function.

### Sliding windows

In Coral, it is also possible to use a *sliding window* in the case of collect-local, spark-batch and log-batch. When a sliding window is used in any of these modes, the oldest input events are dropped and the newest input events are added in front of the batch, with a speed that depends on the configuration of the sliding window.

A sliding window can be set up in two different modes:  

- **Count**: Move the sliding window every \<x\> items.  
- **Time**: Move the sliding window every \<x\> time units (months, days, hours, ...)

A sliding window has a certain size, and a certain speed with which items are dropped and added to the list.

For instance, when a sliding window with a size of 5 items and a count of 2 is defined, the actor will calculate its state based on an input list of 5 items, which moves after every 2 new items that come in:

<table class="toplinks">
	<tr>
		<td>a</td><td>b</td><td>c</td><td>d</td><td>e</td><td></td><td></td><td></td><td></td>
	</tr>
	<tr>
		<td></td><td></td><td>c</td><td>d</td><td>e</td><td>f</td><td>g</td><td></td><td></td>
	</tr>
	<tr>
		<td></td><td></td><td></td><td></td><td>e</td><td>f</td><td>g</td><td>h</td><td>i</td>
	</tr>
</table>

<br>

The time mode does not update the window after a number of events but after a period of time, regardless of the number of events in the list. Spark and log state calculation also allow the use of a sliding window over the selection of items. 

If a sliding window is used, all ru-vars must be treated as nru-vars. This is because it is not possible, for instance, to remove the oldest values from a sum of numbers if these old values are unknown.

A sliding window with a size or a time equal to the size of the slide parameter is equivalent to not using a sliding window at all.

--------------------------------

### <a name="sparkstate"></a>Spark state calculation

Instead of calculating new state inside the actor itself, it is also possible to calculate new state by running a Spark job that reads events from the event log and combines them somehow into a new state object. 
In the Coral platform, each actor itself knows how to combine multiple event log entries into a new state object. 

The function prototype that defines a spark state calculation looks as follows:

{% highlight scala %}
type StateFromSpark = RDD[LogEntry] => Future[JObject]
{% endhighlight %}

which means that a Coral actor that can run a state recalculation on Spark knows how to transform an RDD (a distributed array in Spark) of event log items into a single JSON object somewhere in the future.

If an actor implements this method, it tells the Coral framework that it knows how to use Spark for its state recalculation. The Coral framework will call this function when the combination of platform settings is correct, i.e. Spark is available and Spark recalculation is turned on for the actor, among other things. 

--------------------------------

### <a name="logstate"></a>Log state calculation

Since the Coral platform does not assume that Spark is available, it is also possible to calculate new state by simply fetching a list of historical events directly from the event log and performing the state recalculation function directly inside the actor. 

The function prototype that defines a log state calculation function looks as follows:

{% highlight scala %}
type StateFromLog = LogSelection => Future[JObject]
{% endhighlight %}

which means that a Coral actor that implements this function transforms a list of JSON objects selected from the event log -- annotated with the owner name, runtime name, actor name and timestamp -- into a single JSON state object somewhere in the future. When the future is finished, any variables that might need to be changed will be updated with the results in the JSON object. The logic of this function looks similar to the Spark state calculation above, except that the calculations are now performed locally, and not in a Spark job.

The downside of this method is that if there are multiple copies of a single actor running, each of these actors will calculate the same state if they use the same log selection. This is also true, however, for the State recalculation method with Spark.

--------------------------------

### <a name="interplay"></a>Interplay between different modes

In Coral, it is possible to set up the platform with the following settings:

- **Spark enabled**  
Is Spark enabled? If so, it can be used for Spark state calculation if there are actors in the runtime that need to use it. In order to use Spark for state calculation, the event log must also be enabled.
- **Event log enabled**  
Is the event log enabled? If so, it can be queried for historical events. It can be used by Spark in the *Spark batch calculation* function or it can be used directly by an actor in the *Log batch calculation* function.

Besides these platform settings, it is possible to start each actor with the following flags turned on or off:

- **Calculate local**  
Whether local variables should be updated after every event. This is the typical behavior of actors in Akka. 
- **Collect local**  
Whether batches of local events should be stored locally, and state should be recalculated based on these events. This is useful for calculations which cannot be performed without the list of original values (*nru-vars*).
- **Spark batch**  
Whether Spark should be used to recalculate state based on a set of event log items from Cassandra. This is useful for calculations which take too much time to calculate locally and can better be distributed across the cluster.
- **Log batch**  
Whether the Cassandra log should be used directly to recalculate state. This is useful if the event log is enabled but Spark is not available to recalculate state.

These settings are independent of each other, and they can be specified in the following combinations, not all of which are valid. Also, some of the combinations might not be suitable to use when an actor should be copied/instantiated multiple times in a runtime to scale out (column *distr*, *distributable*).

calc-local| collect-local | spark-batch | log-batch | &nbsp;&nbsp;&nbsp;valid&nbsp;&nbsp;&nbsp; | &nbsp;&nbsp;&nbsp;useful&nbsp;&nbsp;&nbsp; | &nbsp;&nbsp;distr.&nbsp;&nbsp; | &nbsp;&nbsp;remarks&nbsp;&nbsp;
:---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | 
 |  |  |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | 1
 |  |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | 2
 |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | 3
 |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> |  4
 |<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> |  |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | 5
 | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | ? | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | 6
 | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> |  | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | ? | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | 7
 | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span>| <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>  | 8
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | | | | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | 9
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | | | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| 10
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| 11
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| |<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>|<span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span>|<span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | 12
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>|<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | | | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>|<span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span> | 13
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>|<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span> | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| 14
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| | <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| 15
<span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-ok" aria-hidden="true" style="color: #007020" ></span>| <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span>| <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span>| <span class="glyphicon glyphicon-remove" aria-hidden="true" style="color: #8f4632"></span>| 16  

<br>

**Legend**:  

*calc-local*: Should the actor keep local variables and update them when new events come in?  
*collect-local*: Should the actor keep a local list of incoming JSON events and update its state based on these events in a local batch process?  
*spark-batch*: Should the actor ask Spark to recalculate its state from the event log?  
*log-batch*: Should the actor fetch events from the event log and calculate state by itself?  
*valid*: Is the combination of parameters valid?  
*useful*: Is the combination of parameters useful in practice?  
*distr* (distributable): does the combination of parameters make it possible to instantiate multiple copies of the same actor on different machines?

**Remarks**: 

1) Completely stateless, functional actor. Can be scaled to multiple copies without problems.  
2) State only directly calculated from event log, no local state changes. Can be scaled up.  
3) State only directly calculated from Spark, no local state changes. Can be scaled up.  
4) Spark state calculation will override event log calculation if Spark available. Can be scaled up.  
5) Only local batch recalculation. Cannot be scaled up without accepting state drift.*  
6) Local batch could be combined with log batch, log batch will override local.  
7) Local batches could be used for local state, can be combined with Spark. State drift.  
8) Spark overrides log batch, otherwise identical to 7).  
9) Local actors with simple variables, normal actor behavior.  
10) Keep local state and sychronize through event log. State drift between synchronization points.  
11) Local actor state with Spark recalculation. State drift between synchronization points.  
12) Spark overrides log batch, otherwise identical to 11).  
13) Local state with local batch calculation. Useful for combination of ru-vars and nru-vars.   
14) Local combination of ru-vars and nru-vars, synchronization with log. State drift.  
15) Local combination of ru-vars and nru-vars, synchronization with Spark. State drift.  
16) Spark overrides log batch, otherwise identical to 15).    

\* *State drift* is the phenomenom that two actors accrue different state over time because of differences in input events, while update calculations are identical.

--------------------------------

### <a name="finally"></a>Finally

There is no need to worry about the interplay between these different calculation methods, since the Coral platform automatically determines the optimal way to perform state calculation, based on platform settings and settings of individual actors. Where applicable, each actor knows how to calculate state in any of the ways mentioned above, so in the case some method is not available, another method is simply used. In the case an invalid combination of settings is used, Coral will return an error.

Coral prefers to use Spark for its state calculation, but the platform will run fine without Spark, or even the event log. In the future, more intelligent methods of determining the optimal state calculation function might be implemented, such as looking at the number of records to process, and then determining if it might not be worth the trouble to ask Spark to start up a new job.

--------------------------------

### Example state definitions

Below, a couple of examples of state function definitions are given. Note that the update settings can be applied to local, Spark and log batch calculation.

<br>

#### Default setting

{% highlight json %}
{
  ...,
  "state": {
    "calc-local": true
  }
}
{% endhighlight %}

This is the default setting for all stateful Coral actors, so it is unnecessary to specify it in the runtime configuration of a stateful actor. The rest of the state functions (collect-local, spark-batch and log-batch) are turned off by default. Nonetheless, in this example, the shortcut notation is used, and only calc-local is enabled after every event. 

<br>

#### Purely functional actor

{% highlight json %}
{
  ...,
  "state": {
    "calc-local": false,
    "collect-local": false,
    "spark-batch": false,
    "log-batch": false
  }
}
{% endhighlight %}

In this example, a purely functional actor is defined. This is the default setting of all stateless Coral actors, so it is unnecessary to specify it in the runtime configuration of a stateless actor. If the actor is a stateful actor, this setting would not be sensible since the actor would not update its internal state with these settings. In this case, Coral will return an error.

<br>

#### Local state and local batch state calculation

{% highlight json %}
{
  ...,
  "state": {
    "calc-local": true,
    "collect-local": true
  }
}
{% endhighlight %}

In this example, shortcut notation is used again, both calc-local and collect-local are enabled after every event. For the [StatsActor](Actors-StatsActor.html), for example, this would mean that both ru-vars and nru-vars are updated after every event. The rest of the state functions is turned off by default.

<br>

#### Local state and local batch state calculation after every 10 events

{% highlight json %}
{
  ...,
  "state": {
    "calc-local": true, 
    "collect-local": {
      "enabled": true,
      "update": {
        "event-count": 10
      }
    }
  }
}
{% endhighlight %}

In this example, the extended notation is used, and calc-local and collect-local is enabled. For collect-local, an update interval of 10 events is used, meaning that after every 10 events that come in, the internal collect batch is cleared and the collection process starts over from the beginning. 

<br>

#### Spark batch calculation with time interval

{% highlight json %}
{
  ...,
  "state": {
    "calc-local": false,
    "spark-batch": {
      "enabled": true,
      "update": {
        "duration": {
          "hours": 1,
          "minutes": 10,
          "seconds": 20
        }
      }
    }
  }
}
{% endhighlight %}

In this example, a purely distributed Spark batch calculated actor is created which updates its state after every 1 hours, 10 minutes and 20 seconds.

<br>

#### Log batch calculation with local updates

{% highlight json %}
{
  ...,
  "state": {
    "calc-local": true,
    "log-batch": {
      "enabled": true,
      "update": {
        "sliding-window": {
          "size": 10,
          "slide": 2
        }
      }
    }
  }
}
{% endhighlight %}

In this example, local updates are enabled and calculations are performed directly from the event log. Updates are performed on batches of 10 events, with a sliding window of 2. This means that after every two new events, these two events are added in front of the batch and the two oldest events are dropped. On this new batch, the new state is calculated. Local state is still updated after every event.

<br>

#### Combinations of the above

Note that the event-count, duration and sliding-window settings are mutually exclusive; it is not possible to define any combination of them together for an actor. Coral will return an error if you define more than one of them for the same actor. A sliding window with a size of 10, a slide of 0 and units set to events is the same as an event-count of 10.

The calc-local function does not accept an update setting; it can only update its local state after every event. 

The update settings can be applied to any of the local batch, Spark batch and log batch definitions.