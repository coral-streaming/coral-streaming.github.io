---
title: GET cluster
layout: default
topic: API Reference
category: Platform
verb: GET
endpoint: /api/platform/cluster
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

### Get cluster information

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/cluster</div>

Shows cluster information and machines currently in the cluster.

### Example

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/platform/cluster</div>

### JSON output

{% highlight json %}
{
    "cluster": {
        "enabled": true
    }, "machines": [{
        "alias": null,
        "ip": "127.0.0.1",
        "port": 2551,
        "roles": [],
        "status": "Up"
    }]
}
{% endhighlight %}

The platform returns whether cluster mode is enabled ([see the `nc` option](Documentation-CommandLineReference.html) in the command line reference) and a list of all machines in the cluster.  If not part of a cluster, only the current machine is returned. For each machine, the alias, if known, is returned as well. The alias can be specified when adding a machine to a cluster. For each machine, the Akka roles, if any, are shown as well. For more information about information on individual machines, look [here](API-GET-cluster-machine.html).