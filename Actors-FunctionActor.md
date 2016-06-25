---
layout: default
title: FunctionActor
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

# FunctionActor
The `FunctionActor` is an actor that evaluates an expression. It can use fields from the incoming JSON object and it can collect data from other actors. The function is evaluated to either a boolean, a string, or a numeric value, depending on the expression that was entered by the user.

### Creating a FunctionActor

The `FunctionActor` has `"type": "function"`. The `params` value is a JSON object with the following fields:

field  | type | required | description
:----- | :---- | :--- | :------------
`function` | string | yes | The expression that should be evaluated.
`outputfield` | string | yes | The name of the output field to where the result should be written.

<br>

#### Example
{% highlight json %}
{
  "type": "function",
  "params": {
    "function": "x > 10",
    "outputfield": "result"
  }
}
{% endhighlight %}

Now, when the following JSON comes in:

{% highlight json %}
{
  "x": 5
}
{% endhighlight %}

The FunctionActor will return 
{% highlight json %}
{
  "result": false
}
{% endhighlight %}

Because `x` is not larger than 10. 

<br>

#### Other examples of expressions

The FunctionActor can handle (simple and compounded) boolean expressions, string concatenations and numeric expressions. Below, a list of example expressions is provided:

Simple boolean expressions are supported:

{% highlight json %}
"x > 10"
{% endhighlight %}

Compounded boolean expressions are also supported:

{% highlight json %}
"x > 10 && y < 12"
"x > 10 || y > 12"
{% endhighlight %}

Parentheses are also supported:

{% highlight json %}
"(x > 10) || (y < 12 && z == 4)"
{% endhighlight %}

In these examples, `x`, `y` and `z` refer to field names in the incoming JSON object. If any of these fields cannot be found, nothing will be emitted. 

#### Nested fields

Nested fields and array fields can also be accessed, as follows:

{% highlight json %}
"x.y.z > 10 && y < 12"
{% endhighlight %}

if the incoming JSON object looks as follows:

{% highlight json %}
{
  "x": {
    "y": {
      "z": 12
    }
  }, "y": 8
}
{% endhighlight %}

This expression will evaluate to `true`.

#### Array access

It is possible to access an array element with the following notation:

{% highlight json %}
"x.y[2].z[0] > 10 && y < 12"
{% endhighlight %}

Which will access the value 42 in the following incoming JSON object:

{% highlight json %}
{
  "x": {
    "y": [{
      { "other": "notrelevant" },
      { "some": "notinteresting" },
      { "z": [ 42, 5, 9, 3 ] }
    }]
  }, "y": 8
}
{% endhighlight %}

Array indices are 0-based.

#### Numeric expressions

The examples above are all examples of boolean expressions which evaluate to either `true` or `false`. It is also possible to evaluate to a numeric value, like in the following expression:

{% highlight json %}
"(x + (12.4 * 7)) / 2"
{% endhighlight %}

The FunctionActor supports addition, subtraction, multiplication and division. 

#### String expressions

String concatenation is supported in the following way:

{% highlight json %}
"a.b.c + 'xyz'"
{% endhighlight %}

If the incoming JSON is

{% highlight json %}
{
  "a": {
    "b": {
      "c": "uvw"
    }
  }
}
{% endhighlight %}

The result will be 

{% highlight json %}
{
  "result": "uvwxyz"
}
{% endhighlight %}

In the future, string functions such as replace, substring and indexOf could also be supported.

#### Collect definition fields

Not only fields in the incoming JSON object can be looked up, but also fields from other actors through a collect procedure. A collect definition should be provided in the constructor of the FunctionActor as follows:

{% highlight json %}
{
   "collect": [
     { "alias": "myvariable", "from": "stats1", "field": "avg" },
     { "alias": "variable2", "from": "stats3", "field": "count" }
   ]
}
{% endhighlight %}

An expression that uses both collect variables and JSON variables looks as follows:

{% highlight json %}
"x > 10 && myvariable == 20 && variable2 != 4"
{% endhighlight %}

When the following JSON comes in:

{% highlight json %}
{
  "x": 4
}
{% endhighlight %}

and `stats` returns 20 for `myvariable` and `stats3` returns 5 for `variable2`, the result will be the following (assuming the same constructor as specified above):

{% highlight json %}
{
  "result": false
}
{% endhighlight %}

See [Expression Language Syntax](#syntax) for the full syntax of the expression language.

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

## <a name="syntax"></a> Expression language syntax

The syntax for the expression actor is as follows. A `:=` means "is a", the term before the `:=` is any one of the alternatives after the `:=`. A `|` means a parsing alternative, it can either match the term before the `|` or after the `|`. A `*` means that the term between parentheses can be repeated any number of times. A `~` means "and": the term consists of the part before the `~` and the part after the `~`. A term enclosed with `"` means a literal: it matches exactly the string between the two `"`s.

{% highlight text %}
expression := string_expression 
	| boolean_expression 
	| numeric_expression 
	| "(" ~ expression ~ ")"

string_expression := string_expression ~ ("+" ~ string_expression)* 
	| string_atom 
	| json_path 
	| "(" ~ string_expression ~ ")"
	
string_atom := "'" ~ string_literal ~ "'"
string_literal := <every string without ' in it>

boolean_expression := b_term ~ ("||" ~ b_term)*
b_term := b_factor ~ ("&&" ~ b_factor)*
b_factor := "!" ~ b_factor 
	| b_test 
	| b_atom 
	| "(" ~ boolean_expression ~ ")"
b_test := numeric_expression ~ (b_op ~ numeric_expression)*
b_atom := json_path 
	| "true" 
	| "false"
b_op := ">" | "<" | ">=" | "<=" | "==" | "!="

numeric_expression := n_term ~ ("+" ~ n_term | "-" ~ n_term)*
n_term := n_factor ~ ("*" ~ n_factor | "/" ~ n_factor)*
n_factor := "(" ~ numeric_expression ~ ")" 
	| n_atom
n_atom := numeric_literal 
	| json_path
numeric_literal := float_literal 
	| integer_literal

json_path := (json_ident ~ ".")*
json_ident := array_ident 
	| simple_ident
array_ident := simple_ident ~ "[" ~ integer_literal ~ "]"
simple_ident := <any valid Java identifier>
float_literal := integer_literal ~ "." ~ integer_literal
integer_literal := ('1' to '9') ~ ('0' to '9')*

{% endhighlight %}
