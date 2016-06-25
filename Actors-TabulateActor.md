---
layout: default
title: TabulateActor
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

# TabulateActor
The `TabulateActor` is an actor that can manage a table based on a number of dimension fields in the incoming JSON and execute an aggregation function for each unique combination of dimension values. The state of the table can then be queried as a whole, or individual cells can be queried by other actors through a collect definition. 

### Creating a TabulateActor

The `TabulateActor` has `"type": "stats"`. The `params` value is a JSON object with the following fields:

field  | type | required | description
:----- | :---- | :--- | :------------
`dimensions` | array[string] | yes | The names of all dimension fields on which grouping should be performed
`value` | string | yes | The value that should be aggregated.
`functions` | array[string] | yes | The aggregation functions to calculate. Can be `average`, `count`, `sum`, `min` or `max`. See [Aggregation Functions](#functions) description below.
`normalize` | string | no | How to normalize data when data is collected. Can be `row`, `column` or `total`. Only used for "count" and "sum" aggregation functions. Defaults to `total`.
`passthrough` | boolean | no | If the original input event should be emitted after processing it. Defaults to `true`.

#### Example
{% highlight json %}
{
  "type": "tabulate",
  "params": {
    "dimensions": [ "x", "y" ],
    "value": "field1",
    "functions": [ "average", "count", "sum", "max" ],
    "normalize": "row",
    "passthrough": true
  }
}
{% endhighlight %}

<br>

In this example, data in the field `field1` is tabulated by the dimensions `x` and `y`, and the functions average, count and max are calculated separately for `field1` for every unique combination of the `x` and `y` dimensions. The number of dimensions is not bounded by the actor.

Using one dimension is functionally comparable to using a [GroupByActor](Actors-GroupByActor.html) combined with a [StatsActor](Actors-StatsActor.html), using two dimensions is similar to chaining two GroupByActors together and putting a StatsActor behind it.

For the functions `count` and `sum`, data is normalized by rows, i.e. divided by the total of all cells with the same value for `field1`. For the functions `average`, `min` and `max`, the normalize option is ignored.

As an example of an incoming JSON object that this actor could receive, see the following JSON object:

{% highlight json %}
{
  "x": "somevalue",
  "y": "othervalue",
  "field1": 140
}
{% endhighlight %}

<br>

When the TabulateActor receives this JSON object, it will check if the cell with coordinates `("somevalue", "othervalue")` already exists, and if it does, it executes the aggregation functions by combining the old value currently in the cell with the value 140. If it does not exist yet, it will create a new cell and it will then execute the aggregation functions.

## <a name="functions"></a>Aggregation functions

The aggregation functions that can be calculated for each unique combination of dimension values are the following:

function  | required | description
:----- | :---- | :--- | :------------
`average` | no | The average of a numeric field
`count` | no | The number of received events
`sum` | no | The sum of a numeric field
`min` | no | The minimum of a certain numeric field until now
`max` | no | The maximum of a certain numeric field until now

<br>

At least one of these functions must be provided in the constructor of the actor. The `count` function works on all types of JSON fields, while the other aggregation functions only work on numeric JSON fields. For the count and sum aggregation functions, data in a cell can be normalized by row, column or the total of the entire table.

### Trigger
The TabulateActor reads the value of the field with the name specified in `value` ("field1" in the example), if it exists. If it does not exist, the actor does nothing. If the value can be found, all specified aggregation functions are executed for each unique value of all dimension fields. 

There is no limitation on the data type of the dimension fields, even numeric values can be used. However, numeric values are rounded to 4 decimals before being used as dimension values.

The normalization function is only executed when the state is collected by another actor and when the aggregation function is `count` or `sum`. Internally, the TabulateActor will not keep the normalized values but it will only calculate them on request.

### Emit
The TabulateActor can be configured to pass through the original input data with the `passthrough` parameter. If `passthrough` is true, the actor emits the original input event. If it is set to false, it will not emit anything.

### State
The TabulateActor keeps the table as state.

### Collect 

On a collect request of the field `table`, the actor will return the following:

{% highlight json %}
{
  "table": [{
    "x": "value1", 
    "y": "value2",
    "field1": {
      "average": 152.2,
      "count": 35,
      "sum": 529.8,
      "max": 1031.5
  }]
}
{% endhighlight %}

<br>

### State functions

The TabulateActor implements the following state functions:

- **calc-local**  
  When *calc-local* is turned on, the actor will keep the table in memory and will update it after every event.
- **spark-batch**  
  When *spark-batch* is turned on, the actor will persist all events it receives to the Cassandra cluster and will periodically receive a new snapshot of the aggregation table from Spark according to its *spark-batch* update settings. In between state updates from Spark, the table will not be updated unless *calc-local* is also turned on.
- **log-batch**  
  Automatically supported because *spark-batch* is supported.

Since the TabulateActor is a stateful actor, the actor does nothing when all of the state functions are turned off. The TabulateActor does not implement the *collect-local* procedure. 

#### Memory usage

The amount of memory that this actor consumes is proportional to the size of the table. If this is undesirable behavior, an alternative is to connect a [FunctionActor](Actors-FunctionActor.html), a [GroupByActor](Actors-GroupByActor.html) and a [StatsActor](Actors-StatsActor.html) in sequence. The FunctionActor should then concatenate all dimension values in a single string, which will be the field that the GroupByActor groups on. The GroupByActor then creates a StatsActor for each unique combination of dimension fields. The resulting behavior is the same as the `TabulateActor` except that state is now split over multiple actors.

The advantage of a `TabulateActor` over this alternative setup is that it is faster to look up data in a single actor than it is to look up data in multiple actors. Also, it is easier to handle (manually submit, save, and restore) a persisted snapshot from Cassandra for a TabulateActor than to handle snapshots for multiple actors.

### Collect
The `TabulateActor` does not collect state from other actors.