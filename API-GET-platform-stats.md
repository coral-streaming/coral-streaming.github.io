---
title: GET platform statistics
layout: default
topic: API Reference
category: Platform
verb: GET
endpoint: /api/platform/stats
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

### Get platform statistics

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/stats</div>

Obtains platform statistics for the entire platform. This contains statistics about the number of runtimes and the number of actors living on the platform, among other things.

### Example

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/stats</div>

### JSON output

{% highlight json %}
{
  "totalMachines": 2,
  "totalRuntimes": 23,
  "totalActors": 58,
  "totalMessages": 1324,
  "totalExceptions": 12,
  "counters": {
    "total": {
      "stat1": 10,
      "stat2": 20
    }, "actor1": {
      "stat1": 10,
      "stat2": 20
    } 
  }, "cluster": {
    "leader": "akka.tcp://coral@127.0.0.1:2551",
    "roles": [],
    "members": [{
      "address": {
        "fullAddress": "akka.tcp://coral@127.0.0.1:2551",
        "protocol": "akka.tcp",
        "system": "coral",
        "ip": "127.0.0.1",
        "port": 2551
      }, "status": "Up",
      "roles": []
    }]
  }
}
{% endhighlight %}

This output shows that there are currently two machines (or JVM instances) in the cluster, and that there are 23 runtimes running on the platform, with 58 actors in total. Together, they have processed 1324 messages and have encountered 12 exceptions. 

Each actor can keep counters, which are shown in the counters object. The total for each statistic, summed over all actors in the platform, are also shown.

Cluster information is also shown, which shows the leader of the cluster and members currently in the cluster.