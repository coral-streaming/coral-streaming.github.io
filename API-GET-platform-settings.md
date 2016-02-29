---
title: GET platform settings
layout: default
topic: API Reference
category: Platform
verb: GET
endpoint: /api/platform/settings
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

### Get platform settings

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/settings</div>

Shows platform-wide settings.

### Example

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/settings</div>

### JSON output

{% highlight json %}
{
  "127.0.0.1:2551": {
    "akka": {
      "loggers": ["akka.event.slf4j.Slf4jLogger"],
      "loglevel": "info",
      "actor": {
        "provider": "akka.cluster.ClusterActorRefProvider"
      }
    }, "coral": {
      "log-level": "INFO",
      "api": {
        "interface": "0.0.0.0",
        "port": 8000
      }, "akka": {
        "hostname": "127.0.0.1",
        "port": 2551
      }, "distributor": {
        "mode": "local"
      }, "cassandra": {
        "contact-points": ["192.168.100.101"],
        "keyspace": "coral",
        "port": 9042,
        "user-table": "users",
        "runtime-table": "runtimes",
        "authorize-table": "auth",
        "journal-table": "journal",
        "snapshot-table": "snapshots"
      }, "cluster": {
        "enabled": false
      }
    }
  }
}
{% endhighlight %}

The example JSON result shows the settings per machine (IP address/port pair). In this setup, cluster mode is disabled (the JVM is started with the `nc` option). However, the actor provider is still ClusterActorRefProvider. This is the default behavior, even when cluster mode is disabled, actors are still internally referenced by their cluster identifiers. 

The HTTP API accepts connections on all interfaces on port 8000. Akka listens to port 2551 for actor traffic, and the IP address "192.168.100.101:9042" contains a Cassandra instance. Coral uses the Akka port and Akka IP address to identify nodes, which are different from the interface and ports of the HTTP API. 