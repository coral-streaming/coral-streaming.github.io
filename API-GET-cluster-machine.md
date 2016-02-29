---
title: GET cluster machine
layout: default
topic: API Reference
category: Platform
verb: GET
endpoint: /api/platform/cluster/<id>
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

### Get cluster machine information

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/cluster/&lt;id&gt;</div>

Obtains information for a specific "machine", or more specifically, a specific Coral platform instance that is part of the cluster. The `<id>` can be formatted as an IP address, a colon and a port ("127.0.0.1:2551") or as a machine alias ("machine1"). In the case of a machine alias, the machine should have been added to the cluster through the API with that alias. The alias should be unique across the cluster.

Because it is possible to start two instances of Coral on the same machine using different ports, the machine needs to be identified using the IP address and the port combined. Nonetheless, it is questionable how useful it is to divide a fixed amount of resources (a physical computer) across multiple Coral instances.

### Example

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/cluster/127.0.0.1:2551</div>
<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/cluster/machine1</div>

### JSON output

When successful:

{% highlight json %}
{
    "alias": "machine1",
    "ip": "127.0.0.1",
    "port": 2551,
    "roles": [],
    "status": "Up"
}
{% endhighlight %}

The platform returns the alias of the machine (if known, else null), the IP address, the port, possible [Akka roles](http://doc.akka.io/docs/akka/snapshot/java/cluster-usage.html#Node_Roles) of the machine and its [current status](http://doc.akka.io/docs/akka/2.4.1/common/cluster.html#Member_States).

On failure:

{% highlight json %}
{
    "action": "Get machine",
    "success": false,
    "reason": "Machine with given name not found."
}
{% endhighlight %}

### Errors

status code | description | reason
---: | :--- | :---
`404` | Not found | When the combination of IP address and port or the given alias does not match a known machine in the cluster, a 404 error is returned.
