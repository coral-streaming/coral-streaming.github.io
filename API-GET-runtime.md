---
title: GET runtime definition
layout: default
topic: API Reference
category: Runtimes
verb: GET
endpoint: /api/runtimes/<id>
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

### Get runtime definition

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/runtimes/&lt;id&gt;</div>

Shows the definition of a runtime

### Example

<div class="alert alert-info" role="alert"><strong>GET</strong> /api/runtimes/runtime1</div>

### JSON output

{% highlight json %}
{
  "id": "166ec30a-e1ab-4805-86a9-d8e269d68ede",
  "owner": "fb5ffab3-3759-4dd7-9d56-0ce2ee7b369f",
  "name": "runtime1",
  "uniqueName": "coral-runtime1",
  "status": "created",
  "project": "None",
  "json": {
    "name": "runtime1",
    "owner": "fb5ffab3-3759-4dd7-9d56-0ce2ee7b369f",
    "actors": [
      {
        "name": "generator1",
        "type": "generator",
        "params": {
          "format": {
            "field1": "N(100, 10)"
          }, "timer": {
            "rate": 100000000000
          }
        }
      }, {
        "name": "threshold1",
        "type": "threshold",
        "params": {
          "key": "field1",
          "threshold": 100
        }
      }, {
        "name": "sample1",
        "type": "sample",
        "params": {
          "fraction": 0.3
        }
      }, {
        "name": "log1",
        "type": "log",
        "params": {
          "file": "/tmp/ex1.log"
        }
      }
    ], "links": [
      { "from": "generator1", "to": "threshold1" },
      { "from": "threshold1", "to": "sample1" },
      { "from": "sample1", "to": "log1" }
    ]
  }, "starttime": "2016-01-28T13:57:36.355"
}
{% endhighlight %}

The example JSON result shows the definition of the given runtime, which includes:

- The UUID that was assigned to the runtime
- The owner UUID of the runtime by the platform when the runtime was created
- The unique name of the runtime. Coral combines the name of the owner with the name of the runtime to make 
each runtime unique across the entire platform. This way, "user1" and "user2" can both have a "runtime1".
- The start time of the runtime
- The JSON definition of the runtime, containing:
  - All actor definitions, as they were posted when the runtime was created
  - All links between actors, as they were posted when the runtime was created
  - The distribution mode of the runtime, if the default is overriden in its definition.
