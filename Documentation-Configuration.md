---
title: Configuration Reference
layout: default
topic: Documentation
topic_order: 3
order: 7
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

## Configuration Reference

### Configuration options



### Default configuration file

The contents of the default configuration file present in `application.conf` in the `resources` directory in the Coral runtime JAR is shown below.
If these options are not overriden by manually specifying a configuration file on the command line or overriding options on the command line, these defaults are used.

{% highlight json %}
// This is the default configuration file. Settings can be
// overidden by specifying them on a .conf file that is provided
// on the command line, or with command line parameters. This
// file is automatically included with the JAR and contains
// fallback settings. It should not be modified since it contains
// default values.

akka {
  loggers = [akka.event.slf4j.Slf4jLogger]
  loglevel = info
  log-dead-letters-during-shutdown = false

  persistence {
    journal.plugin = "cassandra-journal"
    snapshot-store.plugin = "cassandra-snapshot-store"
  }

  // enables kryo settings customization
  extensions = ["com.romix.akka.serialization.kryo.KryoSerializationExtension$"]

  actor {
    provider = "akka.cluster.ClusterActorRefProvider"

    serializers {
      // kryo is much performant than default java serializer, it produces
      // smaller output stream in shorter time
      // since Akka 2.4.x, it deprecates the default Java serializer as not
      // optimal for the production use
      kryo = "com.romix.akka.serialization.kryo.KryoSerializer"
    }

    serialization-bindings {
      "java.io.Serializable" = kryo
      "scala.collection.immutable.List" = kryo
    }

    # configuration of the serialization library
    kryo {
      post-serialization-transformations = "off"
      idstrategy = "default"
      kryo-trace = false
      implicit-registration-logging = false
    }
  }

  remote {
    log-remote-lifecycle-events = off
    netty.tcp {
      hostname = ${coral.akka.hostname}
      port = ${coral.akka.port}
    }
  }

  cluster {
    seed-nodes = ${coral.cluster.seed-nodes}

    auto-down-unreachable-after = 10s
    metrics {
      enabled = off
    }
  }

  event-count-update {
    mode = "time"
    granularity = 1000
  }
}

spray.servlet.boot-class = "io.coral.api.Boot"

cassandra-journal {
  keyspace = ${coral.cassandra.persistence.journal-keyspace}
  table = "journal"
  contact-points = ${coral.cassandra.contact-points}
  port = ${coral.cassandra.port}
  keyspace-autocreate = ${coral.cassandra.persistence.keyspace-autocreate}

  authentication {
    username = ${coral.cassandra.persistence.user}
    password = ${coral.cassandra.persistence.password}
  }
}

cassandra-snapshot-store {
  keyspace = ${coral.cassandra.persistence.snapshot-store-keyspace}
  table = "snapshots"
  contact-points = ${coral.cassandra.contact-points}
  port = ${coral.cassandra.port}
  keyspace-autocreate = ${coral.cassandra.persistence.keyspace-autocreate}

  authentication {
    username = ${coral.cassandra.persistence.user}
    password = ${coral.cassandra.persistence.password}
  }
}

kafka {
  consumer {
    consumer.timeout.ms = 500
    auto.commit.enable = false
  }

  producer {
    producer.type = async
  }
}

// The actor factories to get actor definitions from.
// When supplying custom actors, an additional actor
// factory should be supplied here.
injections.actorPropFactories = [
  "io.coral.actors.DefaultActorPropFactory"
]

coral {
  // The log level of the application
  // Can be INFO, WARN or ERROR
  log-level = INFO

  // Whether runtimes should be read and started
  // from the runtime table on startup
  start-runtimes-in-table = false

  api {
    // The Spray.io HTTP interface
    // 0.0.0.0 means that Coral will listen to all interfaces
    interface = "0.0.0.0"
    // The HTTP port on which the RESTful API will communicate
    port = 8000
  }

  akka {
    // The interface to which akka actors will communicate
    hostname: "127.0.0.1",
    // The port on which akka actors will communicate
    port: 2551
  }

  // The default distribution mode for new runtimes.
  // Round-robin means that each machine is assigned
  // a new runtime in turn.
  distributor {
    mode = "round-robin"
  }

  authentication {
    // Can be "coral", "accept-all", "deny-all" or "ldap".
    // LDAP authentication currently not functional.
    mode = "coral"
    // Whether to cache runtimes, users and permissions in memory
    // for faster retrieval through the API. Advantage of this is
    // that Cassandra does not need to be queried after every REST
    // call. Disadvantage is that data can sometimes be stale in
    // the cached tables.
    cache = true
  }

  cassandra {
    persistence {
      journal-keyspace = ${coral.cassandra.keyspace}
      snapshot-store-keyspace = ${coral.cassandra.keyspace}
      journal-table = ${cassandra-journal.table}
      snapshot-table = ${cassandra-snapshot-store.table}
      user = ${coral.cassandra.user}
      password = ${coral.cassandra.password}
      keyspace-autocreate = ${coral.cassandra.keyspace-autocreate}
    }

    // The version of Cassandra that is connected to.
    // This impacts certain queries which depend on the
    // structure of system tables.
    version = 2
    contact-points = ["localhost"]
    port = 9042

    // The name of tables Coral uses internally
    // These tables cannot have conflicting names
    keyspace = "coral"
    user-table = "users"
    runtime-table = "runtimes"
    authorize-table = "auth"
    copies-table = "copies"

    // Whether the Coral keyspace will be autocreated
    // if it is not found
    keyspace-autocreate = true

    // The maximum number of results to
    // return from a select query
    max-result-size = 50000

    user = "coral"
    password = "coral"

    event-log {
      // Whether event logging is enabled in the application or not
      enabled = true

      // The base name of the eventlog table. Will be appended 
      // with year and month. For example, if it is february 2016 
      // today, events will be written to table "eventlogtest201602".
      table = "eventlogtest"

      integrated-index {
        // Whether or not the integrated index is enabled. If 
        // turned on, and DSE Max is used, it will fill the 
        // column "fsjon" in the eventlog tables with the 
        // transformed JSON (See EventLogHandler).
        enabled = true
      }

      elasticsearch-index {
        // Whether or not the external elasticsearch index is enabled.
        // This is independent of the integrated index since it can be useful
        // to turn them both on. See ElasticSearchIndexHandler.
        enabled = true
        seed-nodes = [ "192.168.100.101" ]
        port = 9200
      }
    }
  }

  spark {
    // Whether Spark is enabled for state recalculations.
    // This can be an integrated Spark (as in DSE Max), or it can be
    // a separate spark instance. We cannot assume that DSE Max is used
    // so it must work on a separate Spark instance/cluster as well.
    enabled = true
    seed-nodes = [ "192.168.100.101" ]
    port = 7077

    // Whether multiple Spark contexts should be allowed
    allow-multiple-contexts = true
  }

  cluster {
    // Whether the cluster capabilities of Coral are enabled.
    // Coral can also be run in standalone mode, with the "-nc" boot option.
    // In that case, it runs as a single node. This mainly impacts the
    // behavior of the ClusterDistributor, ClusterMonitor, RuntimeAdminActor
    // and MachineAdmin actors.
    enable = true
    seed-nodes = ["akka.tcp://coral@"${akka.remote.netty.tcp.hostname}":"${akka.remote.netty.tcp.port}]
  }
}
{% endhighlight %}