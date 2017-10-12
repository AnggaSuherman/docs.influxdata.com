---
title: Syntax

menu:
  kapacitor_1_3:
    name: Syntax
    identifier: syntax
    weight: 3
    parent: tick
---

# Table of Contents

   * [Concepts](#concepts)
   * [TICKscript syntax](#tickscript-syntax)
      * [Code representation](#code-representation)
      * [Variables and literals](#variables-and-literals)
      * [Statements](#statements)
   * [Taxonomy of node types](#taxonomy-of-node-types)
   * [InfluxQL in TICKscript](#influxql-in-tickscript)
   * [Lamdba expressions](#lambda-expressions)
   * [Summary of variable use between syntactic domains](#summary-of-variable-use-between-syntactic-domains)
   * [Gotchas](#gotchas)

# Concepts

The sections [Introduction](/tbd) and [Getting Started](/tbd) already presented the key concepts of **nodes** and **pipelines**.  Nodes represent process invocation units, that either take data as a batch, or in a point by point stream, and then alter that data, store that data, or based on changes in that data trigger some other activity  such as an alert.  Pipelines are simply logically organized chains of nodes.    

**Built on GO**

One important thing to keep in mind is that the TICKscript parser is built on GO.  Some arguments get passed from TICKscript into underlying GO API's, and some methods accept arguments that can be directly used to instantiate underlying GO structures, especially using the Duration type of the GO Time library.  

**Three Syntax domains**  

When working with TICKscript, three syntax domains will be encountered.  Overarching is the TICKscript syntax of the TICKscript file.  This is primarily composed of nodes chained together in pipelines.  Some nodes on instantiation use strings representing InfluxQL statements.  So, InlfuxQL represents the second syntax domain that can be found.  Other nodes and methods use Lambda expressions, which represents the third syntax domain that will be met. The syntax between these domains, such as when accessing variable values, can differ in important ways, and this can sometimes lead to confusion.

To summarize the three syntax domains found in TICKscript are:

   * TICKscript proper
   * InfluxQL
   * Lambda Expressions

**Directed Acyclic Graphs**  

As mentioned in Getting Started, a pipeline is a Directed Acylic Graph (DAG). (For more information see [Wolfram](http://mathworld.wolfram.com/AcyclicDigraph.html) or [Wikipedia](https://en.wikipedia.org/wiki/Directed_acyclic_graph)). It contains a finite number of nodes (a.k.a. vertices) and edges.  Each edge is directed from one node to another.  No edge path can lead back to an earlier node in the path, which would result in a cycle or loop.  TICKscript paths (a.k.a pipelines and chains) typically begin with a processing mode definition node with an edge to a data set definition node and then pass their results down to filtering and processing nodes.

<!-- img src="/img/kapacitor/dag.png" width="120" height="75"  style="height: 200px; width: 320px"></img -->

# TICKscript syntax

TICKscript is case sensitive and uses Unicode. The TICKscript parser scans TICKscript code from top to bottom and left to right instantiating variables and nodes and then chaining or linking them together into pipelines as they are encountered.  When loading a TICKscript the parser checks that a chaining method called on a node is valid.  If an invalid node is encountered the parser will throw an error with the message "no method or property &lt;identifier&gt; on &lt;node type&gt;".

## Code representation

As the TICKScript parser is built on GO, source files should be encoded using **UTF-8**.  A script is broken into **declarations** and **expressions** that terminate with a newline character.  

**Whitespace** is used in declarations to separate variable names from operators and literal values.  It is also used before expressions to create indentations, which indicate the hierarchy of method calls.  This also helps to make the script more readable.  Whitespace can be used in method calls in much the same way it is used in GO.  That is, it can be used to separate and to name method call arguments.  Argument naming is especially common in passing lambda expressions.  

**Comments** can be created on a single line by using a pair of forward slashes "//" before the text.  Comment forward slashes can be preceded by whitespace and need not be the first characters of a newline. However comments should not follow after a declaration or expression.

### Keywords

TICKscript is compact and contains only a small set of keywords compared to less specific languages.

| **Word** | **Usage** |
| :----|:------|
| **AND**  | Standard boolean conjunction operator. |
| **FALSE** | The literal boolean value "false". |
| **lambda** | Flags that what follows is to be interpreted as a lambda expression. |
| **OR** | Standard boolean disjunction operator. |
| **TRUE** | The literal boolean value "true". |
| **var** | Starts a variable declaration. |

Since the set of native node types available in TICKscript is limited, each node type, such as `batch` or `stream`, could be considered key.  Node types and their taxonomy are discussed in detail below.  

### Operators

TICKscript borrows many operators from GO and adds a few which make sense in its data processing domain.

**Standard Operators**

| **Operator** | **Type** | **Usage** | **Examples** |
|:-------------|:---------|:----------|:-------------|
| **+** | Binary | Addition and string concatenation | `3 + 6`, `total + count` and `'foo' + 'bar'` |
| **-** | Binary | Subtraction | `10 - 1`, `total - errs` |
| **\*** | Binary | Multiplication | `3 * 6`, `ratio * 100.0` |
| **/** | Binary |  Division | `36 / 4`, `errs / total` |
| **==** | Binary | Comparison of equality |  `1 == 1`, `date == today` |
| **!=** | Binary | Comparison of inequality | `result != 0`, `id != "testbed"` |
| **<** | Binary | Comparison less than | `4 < 5`, `timestamp < today` |
| **<=** | Binary | Comparison less than or equal to | `3 <= 6`, `flow <= mean` |
| **>** | Binary | Comparison greater than | `6 > 3.0`, `delta > sigma` |
| **>=** | Binary | Comparison greater than or equal to | `9.0 >= 8.1`, `quantity >= threshold` |
| **=~** | Binary | Regular expression equality.  Right value must be a regular expression <br/>or a variable holding such an expression. | `tag =~ /^cz\d+/` |
| **!~** | Binary | Negative regular expression equality. Right value must be a regular expression <br/>or a variable holding such an expression. | `tag !~ /^sn\d+/` |
| **!** | Unary | Logical not | `!TRUE`, `!cpu_idle > 70` |
| **AND** | Binary | Logical conjunction |  `rate < 20.0 AND rate >= 10` |
| **OR** | Binary | Logical disjunction | `status > warn OR delta > signma` |

Standard operators are used in TICKscript and in Lambda expressions.

**Chaining Operators**

| **Operator** | **Usage** | **Examples** |
|:-------------|:----------|:------------|
| **\|** |  Declares an instance of a new node and chains it to the node above it | `stream`<br/>&nbsp;&nbsp;&nbsp;\|`from()` |
| **.** | Invokes a property method, setting or changing an internal property in the node above | `from()`<br/>&nbsp;&nbsp;&nbsp;`.database(mydb)` |
| **@** | Declares a user defined function (UDF) | `from()`<br/>`...`<br/>`@MyFunc()` |

Chaining operators are used to create pipeline statements.

## Variables and literals

No scripting language would be of much use without the ability to declare variables to hold values and to initialize them with literals.  Variables in TICKscript are useful for storing and reusing values and for providing a friendly mnemonic for quickly understanding what a variable represents.

### Variables

#### Naming variables

Variables are declared using the keyword `var` at the start of the declaration and then assigning them a literal value.  Variable identifiers must begin with a standard ASCII letter and can be followed by any number of letters, digits and underscores.  Both upper and lower case can be used.  The type the variable will hold depends upon the literal value it is assigned when it is declared.  

**Example 1 &ndash; variable declarations**
```javascript
var my_var = 'foo'
var MY_VAR = 'BAR'
var my_float = 2.71
var my_int = 1
var my_node = stream
```

Variables are immutable and cannot be reassigned new values later on in the script, though they can be used in other declarations and can be passed into methods.

### Literal values

Literal values are parsed into type instances.  They can be declared directly in method arguments or can be assigned to variables.  The parser interprets types based on context and creates instances of the following: boolean, string, float, integer, regular expression.  Lists, lambda expressions, GO Duration structures and nodes are also recognized.  The rules the parser uses to recognize a type are discussed in the following Types section.

#### Types

##### Booleans
Boolean values are generated using the boolean keywords: `TRUE` and `FALSE`.  Note that these keywords use all upper case letters.  The parser will throw an error when values using lower case are used, e.g. `True` or `true`.

**Example 2 &ndash; Boolean literals**
```javascript
var true_bool = TRUE
...
   |flatten()
       .on('host','port')
       .dropOriginalFieldName(FALSE)
```

In Example 2 above the first line shows a simple assignment using a boolean literal.  The second example shows using the boolean literal `FALSE` in a method call.

##### Numerical types
Any literal token beginning with a digit will lead to the generation of a numerical type instance.  TICKscript understands two numerical types based on GO: `int64` and `float64`.  Any numerical token containing a decimal point will result in the creation of a `float64` value.  Any numerical token that ends without containing a decimal point will result in the creation of an `int64` value.  If an integer is prefixed with the zero character, `0`, it is interpreted as an octal.

**Example 3 &ndash; Numerical literals**
```javascript
var my_int = 6
var my_float = 2.71828
var my_octal = 0400
...
```
In Example 3 above `my_int` is of type `int64`, `my_float` is of type `float64` and `my_octal` is of type `int64` octal.

##### Strings
Strings begin with either one or three single single quotation marks: `'` or `'''`.  Strings can be concatenated using the addition `+` operator.  To escape quotation marks within a string delimited by a single quotation mark use the backslash character.  If it is to be anticipated that many quotation marks will be encountered inside the string, delimit it using triple single quotation marks instead.  The double quotation mark, which is used to access field and tag values, can be used without an escape.   

**Example 4 &ndash; Basic strings**

```javascript
var region1 = 'EMEA'
var old_standby = 'foo' + 'bar'
var query1 =   'SELECT mean("sequestr") FROM "co2accumulator"."autogen".co2 WHERE "location" = \'50.0675, 14.471667\' '
var query2 = '''SELECT mean("sequestr") FROM "co2accumulator"."autogen".co2 WHERE "location" = '50.0675, 14.471667' '''
...
batch
   |query('SELECT count(sequestr) FROM "co2accumulator"."autogen".co2 WHERE sequestr < 0.25 GROUP BY location')
...   
```
In Example 4 above the first line shows a simple string assignment using a string literal.  The second line uses the concatenation operator.  Lines three and four show two different approaches to declaring complex string literals with and without internally escaped single quotation marks.  The final example shows using a string literal directly in a method call.

To make long complex strings more readable newlines are permitted within the string.

**Example 5 &ndash; Multiline string**
```javascript
batch
   |query('
      SELECT mean("sequestr") AS stat
      FROM "co2accumulator"."autogen".co2
      WHERE "location" = \'50.0675, 14.471667\'
      ')
```
In Example 5 above the string is broken up to make the query more easily understood.

##### String templates

String templates allow node properties, tags and fields to be added to a string. This is useful when writing alert messages.  To add a property, tag or field value to a string template, it needs to be wrapped inside of double curly braces: "{{}}".

**Example 6 &ndash; Variables inside of string templates**
```javascript
|alert()
  .id('{{ index .Tags "host"}}/mem_used')
  .message('{{ .ID }}:{{ index .Fields "stat" }}')
```    
In Example 6 three values are added to two string templates.  In the call to the setter `id()` the value of the tag `"host"` is added to the start of the string.  The call to the setter `message()` then adds the `id` and then the value of the field `"stat"`.  This is currently applicable with the [Alert](/tbd) node and is discussed further in the section [Accessing values in string templates](#accessing-values-in-string-templates) below.

String templates can also include flow statements such as `if...else` as well as calls to internal formating methods.

```
.message('{{ .ID }} is {{ if eq .Level "OK" }}alive{{ else }}dead{{ end }}: {{ index .Fields "emitted" | printf "%0.3f" }} points/10s.')
```

##### String lists

A string list is a collection of strings declared between two brackets.

**Example 6 &ndash; String list**
```javascript
var my_str_list = [ 'cz', 'sk', 'pl', 'hu', 'at' ]
```
<p style="color: red">FIXME COMMENT - I see that I can create this in a script, but cannot derefence the internal elements.  I also see in the code base, that there is a String List node (tick/ast/parser.go:366) which creates a list of String nodes.  However, I cannot seem to find an example, even in tests, where this is used.  Is there a use case for this?</p>

##### Regular expressions

As in Javascript regular expressions begin and end with a forward slash: `/`.  The expression content should be compatible with the GO [regular expression library](https://golang.org/pkg/regexp/syntax/) (`regexp`).

**Example 7 &ndash; Regular expressions**
```javascript
var cz_turbines = /^cz\d+/
var adr_senegal = /\.sn$/
var local_ips = /^192\.168\..*/
...
var locals = stream
   |from()
      .measurement('responses')
      .where(lambda: "node" =~ local_ips )

var south_afr = stream
   |from()
      .measurement('responses')
      .where(lambda: "dns_node" =~ /\.za$/ )       
```
In Example 7 the first three examples show the assignment of regular expressions to variables.  The `locals` stream reuses the regular expression assigned to the variable `local_ips`. The `south_afr` stream uses a regular expression comparison with the regular expression declared literally as a part of the lambda expression.

##### Lambda expressions as literals

A lambda expression is a parameter representing a short easily understood function to be passed into a method call or held in a variable. It can wrap a boolean expression, a mathematical expression, a call to an internal function or a combination of these three.  As of release 1.3 the three stateful functions are provided.

   * `sigma` - counts the number of standard deviations a give value is from the running mean.
   * `count` - counts the number of values processed.
   * `spread`- computes the running range of all values.

Additionally it is possible to use stateless type conversion and mathematical functions.  These are discussed in the sections [Type converversion](#type-conversion) and [Lambda Expressions](#lambda-expressions) below.  Lambda expressions are presented in detail in the topic [Lambda Expressions](/kapacitor/v1.3/tick/expr/)   

Lambda expressions begin with the token `lambda` followed by a colon, ':' &ndash; `lambda:`.  

**Example 8 &ndash; Lambda expressions**
```javascript
var my_lambda = lambda: 1 > 0
var lazy_lambda = lambda: "usage_idle" < 95
...
var data = stream
  |from()
...
var alert = data
  |eval(lambda: sigma("stat"))
    .as('sigma')
    .keep()
  |alert()
    .id('{{ index .Tags "host"}}/cpu_used')
    .message('{{ .ID }}:{{ index .Fields "stat" }}')
    .info(lambda: "stat" > info OR "sigma" > infoSig)
    .warn(lambda: "stat" > warn OR "sigma" > warnSig)
    .crit(lambda: "stat" > crit OR "sigma" > critSig)

```
Example 8 above shows that a lambda expression can be directly assigned to a variable.  In the eval node a lambda statement is used which calls the sigma function. The alert node uses lambda expressions to define the log levels of given events.  

##### GO Duration structures

GO duration structures are generated internally whenever a duration literal is encountered in the script in a context to which a time duration can be applied.  This syntax follows the same syntax present in [InfluxQL](https://docs.influxdata.com/influxdb/v1.3/query_language/spec/#literals).  A duration literal is comprised of two parts: an integer and a duration unit.  It is essentially an integer terminated by one or a pair of reserved characters, which represents a unit of time.

**Unit**  | **Meaning**
-------|-----------------------------------------
u or µ | microseconds (1 millionth of a second)
ms     | milliseconds (1 thousandth of a second)
s      | second
m      | minute
h      | hour
d      | day
w      | week

**Example 9 &ndash; Duration expressions**
```javascript
var period = 10s
var every = 10s
...
var views = batch
    |query('SELECT sum(value) FROM "pages"."default".views')
        .period(1h)
        .every(1h)
        .groupBy(time(1m), *)
        .fill(0)
```

In Example 9 above the first two lines show the declaration of Duration types.  The first represents a period of 10 seconds and the second a time frame of 10 seconds.  The final example shows declaring duration literals directly in method calls.

##### Nodes

Like the simpler types Node types are declared and can be assigned to variables.  More often they are added to pipelines using chaining methods.

**Example 10 &ndash; Node expressions**
```javascript
var data = stream
  |from()
    .database('telegraf')
    .retentionPolicy('autogen')
    .measurement('cpu')
    .groupBy('host')
    .where(lambda: "cpu" == 'cpu-total')
  |eval(lambda: 100.0 - "usage_idle")
    .as('used')
  |window()
    .period(period)
    .every(every)
  |mean('used')
    .as('stat')
...
var alert = data
  |eval(lambda: sigma("stat"))
    .as('sigma')
    .keep()
...    
```
In Example 10 above, in the first section, five nodes are created.  The top level node `stream` is assigned to the variable `data`.  `stream` is then used as the root of the pipeline to which the nodes `from`, `eval`, `window` and `mean` are chained in order. In the second section the pipeline is then extended using assignment to the variable `alert`, so that a second `eval` node can be applied to the data.  

#### Working with arguments and variables

While it is possible to declare and use variables in TICKscript, it is also possible to work with tags and fields drawn from InfluxDB data series. This is most evident in the examples presented so far. The following section explores working not only with variables but also with tag and field values, that should be found in the data.    

##### Accessing values

As was pointed out in the [Getting started guide](http://localhost:1414/kapacitor/v1.3/introduction/getting_started/#gotcha-single-versus-double-quotes) accessing data tags and fields, using string literals and accessing TICKscript variables each involves different syntax.  Additionally it is possible to access the results of lambda expressions used with certain nodes.  

   * **Variables** &ndash; To access a _TICKscript variable_ simply use its identifier.  

   **Example 11 &ndash; Variable access**
   ```javascript
   var db = 'website'
   ...
   var data = stream
    |from()
        .database(db)
   ...
   ```
   In Example 11 the variable `db` is assigned the literal value `'website'`.  This is then used in the setter `.database()` under the chaining method `from()`.

   * **String literals** &ndash; To declare a _string literal_ use single quotation marks as discussed in the section [Strings](#strings) above.

   * **Tag and Field values** &ndash; To access a _tag value_ or a _field value_ from the data set in a Lambda expression use double quotes.  To refer to them in method calls use single quotes.  

   **Example 12 &ndash; Field access**
   ```javascript
   // Data frame
  var data = stream
     |from()
        .database('telegraf')
        .retentionPolicy('autogen')
        .measurement('cpu')
        .groupBy('host')
        .where(lambda: "cpu" == 'cpu-total')
     |eval(lambda: 100.0 - "usage_idle")
        .as('used')
   ...        
   ```
   In Example 12 two values from the data frame are accessed.  In the `where()` method call the lambda expression uses the tag `"cpu"` to filter the data frame down to only datapoints whose "cpu" tag equals the literal value of `'cpu-total'`.  The chaining method `eval()` also takes a lambda expression that accesses the field `"usage-idle"` to calculate cpu processing power 'used'.  Note that the `groupBy()` method uses a string literal to name the tags by which the data will be grouped.  

   * **Named lambda expression results** &ndash; Lambda expression results get named using an `as()` method.  Think of the `as()` method functioning just like the 'AS' keyword in InfluxQL.  See the `eval()` method in Example 12 above.  The results of lambda expressions can be accessed with double quotation marks, just like data fields.  

  **Example 13 &ndash; Name lambda expression access**

  ```javascript
  ...
      |window()
        .period(period)
        .every(every)
      |mean('used')
        .as('stat')

    // Thresholds
    var alert = data
      |eval(lambda: sigma("stat"))
        .as('sigma')
        .keep()
      |alert()
        .id('{{ index .Tags "host"}}/cpu_used')
        .message('{{ .ID }}:{{ index .Fields "stat" }}')
        .info(lambda: "stat" > info OR "sigma" > infoSig)
        .warn(lambda: "stat" > warn OR "sigma" > warnSig)
        .crit(lambda: "stat" > crit OR "sigma" > critSig)
  ```
  Example 13 above continues the pipeline from Example 12.  In Example 12, the results of the lambda expression named as `'used'` under the `eval()` method are then accessed in Example 13 as an argument to the method `'mean()'`, which then names its result _as_ `'stat'`.  A new statement then begins.  This contains a new call to the method `'eval()'`, which has a lambda expression that accesses `"stat"` and sets its result _as_ `'sigma'`.  `"stat"` is also accessed in the `message()` method and the threshold methods (`info()`,`warn()`,`crit()`) under the `alert()` chaining method.  The named result `"sigma"` is also used in the lambda expressions of these methods.

  **Note &ndash; InfluxQL nodes and tag or field access** &ndash; [InfluxQL nodes](kapacitor/v1.3/nodes/influx_q_l_node/) are special nodes that wrap InfluxQL functions. See the section [Taxonomy of node types](#taxonomy-of-node-types) below.  When accessing field values, tag values or named results with this node type single quotes are used.

  **Example 14 &ndash; Field access with an InfluxQL node**
  ```javascript
  // Dataframe
var data = stream
 |from()
   .database('telegraf')
   .retentionPolicy('autogen')
   .measurement('cpu')
   .groupBy('host')
   .where(lambda: "cpu" == 'cpu-total')
 |eval(lambda: 100.0 - "usage_idle")
   .as('used')
 |window()
   .period(period)
   .every(every)
 |mean('used')
   .as('stat')
  ```
  In Example 14 above the `eval` result gets named as `used`.  The chaining method `mean` is an alias of the node type InfluxQL.  It wraps the InfluxQL `mean` function.  In the call to mean the value `used` is accessed using only single quotes.


##### Accessing values in string templates

As mentioned in the section [String templates](#string-templates) it is possible to add values from node specific properties, and from tags and fields to output strings.  This can be seen under the alert node in Example 13.  The accessor expression is wrapped in two curly braces.  To access a property a period, `.`, is used before th identifier.  To access a value from tags or fields the token 'index' is used, followed by as space and a period and then the part of the data series to be accessed (e.g. `.Tag` or `.Field`), the actual name is then specified in double quotes.      

```javascript
|alert()
  .id('{{ index .Tags "host"}}/mem_used')
  .message('{{ .ID }}:{{ index .Fields "stat" }}')
```    

For more specific information see the [Alert](/tbd) node documentation.

##### Type conversion

Within lambda expressions it is possible to use stateless conversion functions to convert values between types.

   * `bool()` - converts a string, int64 or float64 to boolean.  
   * `int()` - converts a string, float64, boolean or duration type to an int64.
   * `float()` - converts a string, int64 or boolean to float64.
   * `string()` - converts an int64, float64, boolean or duration value to a string.
   * `duration()` - converts an int64, float64 or string to a duration type.  

**Example 15 &ndash; Type conversion**

```javascript
   |eval(lambda: float("total_error_responses")/float("total_responses") * 100.0)
```

In Example 15 above the `float` conversion function is used to ensure that the calculated percentage uses floating point precision when the field values in the InfluxDB database may have been stored as integers.

##### Numerical precision

<!-- issue 1244 -->

When writing floating point values in messages, or to InfluxDB it might be helpful to specify the decimal precision in order to make the values more readable or better comparable.  For example in the `messsage()` method of an `alert` node it is possible to "pipe" a value to a `printf` statement.  

```javascript
|alert()
  .id('{{ index .Tags "host"}}/mem_used')
  .message('{{ .ID }}:{{ index .Fields "stat" | printf "%0.2f" }}')
```   

When working with floating point values in lambda expressions, it is possible to use the floor function and powers of ten to round to a less precise value.  Note that since values are stored as 64bits, this has no effect on storage.  If this were to be used with the `InfluxDBOUt` node, for example when downsizing data, it could lead to a needless loss of information.   

**Example 16 &ndash; Rendering floating points less precise**
```javascript
stream
 // Select just the cpu measurement from our example database.
 |from()
    .measurement('cpu')
 |eval(lambda: floor("usage_idle" * 1000.0)/1000.0)
    .as('thousandths')
    .keep('usage_user','usage_idle','thousandths')
 |alert()
    .crit(lambda: "thousandths" <  95.000)
    .message('{{ index .Fields "thousandths" }}')
       // Whenever we get an alert write it to a file.
    .log('/tmp/alerts.log')   
```
Example 16 accomplishes something similar to using `printf`.  The `usage_idle` value is rounded down to thousandths of a percent and then used for comparison in the threshold method of the alert node.  It is then written into the alert message.

##### Time precision

<!-- Issue 1244 -->

As Kapacitor and TICKscripts can be used to write values into an InfluxDB database, it will be necessary, in some cases, to specify the time precision to be used.  This occurs when working with the `InfluxDBOut` node, whose precision property can be set.  It is important not to confuse _mathematical_ precision, which is used most commonly with field values, and _time_ precision which is specified for timestamps.  

**Example 17 &ndash; Setting time precision with InfluxDBOut**
```javascript
...
   |influxDBOut()
      .database('co2accumulator')
      .retentionPolicy('autogen')
      .measurement('testvals')
      .precision('m')
      .tag('test','true')
      .tag('version','0.1')
...      
```
In Example 17 the time precision of the series to be written to the database "co2accumulator" as measurement "testvals" is set to the unit minutes.  

Valid values for precision are the same as those used in InfluxDB.

|String|Unit|
|:-----|----|
| "ns" | nanoseconds |
| "ms" | milliseconds |
| "s" | seconds |
| "m" | minutes |
| "h" | hours |

<!--
##### Regular expressions
TODO:  Is there more to be said about regular expression?
-->
## Statements

There are two types of statements in TICKscript: Declarations and Expressions.  Declarations declare variables.  Expressions express a pipeline (a.k.a chain) of method calls, which instantiate and set the properties of processing nodes.   

### Declarations

Declarations begin with the "var" keyword followed by an identifier for the variable being declared.  An assignment operator follows with a literal right side value, which will set the type and value for the new variable.

**Example 18 &ndash; Typical declarations**
```javascript
...
var db = 'website'
var rp = 'autogen'
var measurement = 'responses'
var whereFilter = lambda: ("lb" == '17.99.99.71')
var name = 'test rule'
var idVar = name + ':{{.Group}}'
...
```
Example 18 shows six declaration statements. Five of them create variables holding strings and one a lambda expression.

A declaration can also be used to assign an expression to a variable.  

**Example 19 &ndash; Declaring an expression to a variable**
```javascript
var data = stream
    |from()
        .database(db)
        .retentionPolicy(rp)
```
In Example 19 the `data` variable holds the stream pipeline declared in the expression beginning with the node `stream`.

### Expressions

An expression begins with a node identifier or a variable identifier holding another expression.  It then chains together additional node instantiation methods (chaining methods), property setters (property methods) or user defined functions (UDF).  The pipe operator "|" indicates the start of a chaining method call, returning a new node into the chain.  The dot operator "." adds a property setter.  The ampersand operator "@" introduces a user defined function.

Expressions can be written all on a single line, but this can lead to readability issues.  The command `kapacitor show <taskname>` will pretty print scripts using indentation.  Adding a new line and indenting new method calls is the recommended practice for writing TICKscript expressions.  Typically, when a new chaining method is introduced in an expression, a newline is created and the new link in the chain gets indented three or more spaces.  Likewise, when a new property setter is called, it is set out on a new line and indented an additional number of spaces.  For readability user defined functions should be indented the same as chaining methods.  

An expression ends with the last setter of the last node in the pipeline.

**Example 20 &ndash; Single line expressions**
```javascript
stream|from().measurement('co2')|window().period(1m).every(30s).align()
   |eval(lambda: sqrt("mofcap") ).as('sqrt')
   |influxDBOut().database('co2accumulator').retentionPolicy('autogen').measurement('testvals')
       .tag('test','true').tag('version','0.1')

...
```
Example 20 shows an expression with a number of nodes and setters declared all on the same line.  While this is possible, it is not the recommended style.    

**Example 21 &ndash; Recommended expression syntax**
```javascript
// Dataframe
var data = batch
  |query('''SELECT mean(used_percent) AS stat FROM "telegraf"."autogen"."mem" ''')
    .period(period)
    .every(every)
    .groupBy('host')

// Thresholds
var alert = data
  |eval(lambda: sigma("stat"))
    .as('sigma')
    .keep()
  |alert()
    .id('{{ index .Tags "host"}}/mem_used')
    .message('{{ .ID }}:{{ index .Fields "stat" }}')
    .info(lambda: "stat" > info OR "sigma" > infoSig)
    .warn(lambda: "stat" > warn OR "sigma" > warnSig)
    .crit(lambda: "stat" > crit OR "sigma" > critSig)

// Alert
alert
  .log('/tmp/mem_alert_log.txt')
```
Example 21, taken from the example [mem_alert_batch.tick](https://github.com/influxdata/kapacitor/blob/master/examples/telegraf/mem/mem_alert_batch.tick) in the code base, shows the recommended style for writing expressions.  This example contains three expression statements.  The first begins with the declaration of the batch node for the data frame.  This gets assigned to the variable data.  The second expression takes the data variable and defines thresholds for warning messages.  This gets assigned to the alert variable.  The third expression sets the log property of the alert node.  

### Node instantiation

With two exceptions (`stream` and `batch`) nodes always occur in pipeline expressions (chains), where they are instantiated through chaining methods.  Chaining methods are generally identified using the node type name.  One notable exception to this is the InfluxQL node, which uses aliases.  See the section [Taxonomy of node types](#taxonomy-of-node-types) below.   

For each node type, the method that creates an instance of that type uses the same signature.  So if a `query` node instantiates an `eval` node and adds it to the chain, and if a `from` node can also create an `eval` node and add it to the chain, the chaining method creating a new `eval` node will accept the same arguments (e.g. one or more lamdba expressions) regardless of which node created it.  

**Example 22 &ndash; Instantiate eval node in stream**
```javascript
...
var data = stream
  |from()
    .database('telegraf')
    .retentionPolicy('autogen')
    .measurement('cpu')
    .groupBy('host')
    .where(lambda: "cpu" == 'cpu-total')
  |eval(lambda: 100.0 - "usage_idle")
    .as('used')
    .keep()
    ...
```  
Example 22 instantiates three nodes: `stream`, `from` and `eval`.

**Example 23 &ndash; Instantiate eval node in batch**
```javascript
...
var data = batch
  |query('''SELECT 100 - mean(usage_idle) AS stat FROM "telegraf"."autogen"."cpu" WHERE cpu = 'cpu-total' ''')
    .period(period)
    .every(every)
    .groupBy('host')
  |eval(lambda: sigma("stat"))
    .as('sigma')
    .keep()
    ...
```
Example 23 also instantiates three nodes: `batch`,`query` and `eval`.

Both Examples 22 and 23 instantiate an `eval` node.  Despite that `eval` is chained below a `from` node in Example 21 and below a `query` node in Example 22, the signature of the chaining method remains the same.  

A short taxonomy of nodes is presented in the section [Taxonomy of node types](#taxonomy-of-node-types) below.  The catalog of node types is available under the topic [TICKscript nodes](/kapacitor/v1.3/nodes/).

### Pipelines

To reiterate, a pipeline is a logically ordered chain of nodes defined by one or more expressions.  "Logically ordered" means that nodes cannot be chained in any random sequence, but occur in the pipeline according to their role in processing the data.  A pipeline can begin with with one of two mode definition nodes: `batch` or `stream`.  The data frame for a `batch` pipeline is defined in a `query` definition node.  The data stream for a `stream` pipeline is defined in a `from` definition node.  After the definition nodes any other types of nodes may follow.  

Standard node types get added to the pipeline with a chaining method indicated by the pipe "|" character.  User defined functions can be added to the pipeline using the ampersand "@" character.

Each node in the pipeline has internal properties that can be set using property methods delineated using a period ".".  These methods get called before the node processes the data.  

Each node in the pipeline can alter the data passed along to the nodes that follow.  In some nodes, setting a property can significantly alter the data received by downstream siblings.  For example, with an `eval` node, setting the names of lambda functions with the `as` property effectively blocks field and tag names from being passed downstream.  For this reason it is important to set the `keep` property, to keep them in the pipeline.  

It is important to become familiar with the [reference documentation](/kapacitor/v1.3/nodes/) for each node type before using it in a TICKscript.


**Example 24 &ndash; a typical pipeline**
```javascript
// Dataframe
var data = batch
  |query('''SELECT 100 - mean(usage_idle) AS stat FROM "telegraf"."autogen"."cpu" WHERE cpu = 'cpu-total' ''')
    .period(period)
    .every(every)
    .groupBy('host')

// Thresholds
var alert = data
  |eval(lambda: sigma("stat"))
    .as('sigma')
    .keep()
  |alert()
    .id('{{ index .Tags "host"}}/cpu_used')
    .message('{{ .ID }}:{{ index .Fields "stat" }}')
    .info(lambda: "stat" > info OR "sigma" > infoSig)
    .warn(lambda: "stat" > warn OR "sigma" > warnSig)
    .crit(lambda: "stat" > crit OR "sigma" > critSig)

// Alert
alert
  .log('/tmp/cpu_alert_log.txt')
```
Example 24 shows a `batch`&rarr;`query` pipeline broken into three expressions using two variables.  The first expression declares the data frame, the second expression the alert thresholds and the final expression sets the `log` property of the `alert` node.  The entire pipeline begins with the declaration of the `batch` node and ends with the call to the property method `log()`.    

# Taxonomy of node types

To aid in understanding the roles that different nodes play in a pipeline, a short taxonomy has been defined.  For complete documentation on each node type see the topic [TICKscript Nodes](/kapacitor/v1.3/nodes/).

**Special nodes**

These nodes are special because they can be instantiated and returned using identifiers other than their type names.  An alias representing an aspect of their functionality can be used.  This may apply in all instances, as with the InfluxQL node, or only in one, as with the Alert node.

   * [`alert`](/kapacitor/v1.3/nodes/alert_node/) - can be returned as a Deadman switch
   * [`influxQL`](kapacitor/v1.3/nodes/influx_q_l_node/) - directly calls functions in InfluxQL, so can be returned when a TICKScript chaining method using the name of the InfluxQL method is called.
      * example 1 `From()|Mean()` - calls the mean function on dataset defined in the from node and returns an InfluxQL node.
      * example 2 `Query()|Mode()` - calls the mode function on the dataset defined in the Query node and returns an InfluxQL node.

**Mode definition nodes**

First node in a TICKScript graph is either `batch` or `stream`. They define the _mode_ used in processing the data.

   * [`batch`](/kapacitor/v1.3/nodes/batch_node/) - chaining method call syntax is not used in the declaration.
   * [`stream`](kapacitor/v1.3/nodes/stream_node/) - chaining method call syntax is not used in the declaration.

**Data definition nodes**

Mode definition nodes are typically followed by nodes whose purpose is to define a _frame_ or _stream_ of data to be processed by other nodes.

   * [`from`](/kapacitor/v1.3/nodes/from_node/) - has an empty chaining method.  Can follow only a `stream` node.  Configure using property methods.
   * [`query`](/kapacitor/v1.3/nodes/query_node/) - chaining method takes a query string. Can follow only a `batch` node.

**Filtering nodes**

Values within the data set can be altered or generated using filtering nodes.

   * [`default`](/kapacitor/v1.3/nodes/default_node/) - has an empty chaining method. Its field and tag properties can be used to set default values for fields and tags in the data series.
   * [`sample`](/kapacitor/v1.3/nodes/sample_node/) - chaining method takes an int64 or a duration string.  It extracts a sample of data based on count or the time period.
   * [`shift`](/kapacitor/v1.3/nodes/shift_node/) - chaining method takes a duration string. It shifts datapoint time stamps.  The duration string can be proceeded by a minus sign to shift the stamps backward in time.
   * [`where`](/kapacitor/v1.3/nodes/where_node/) - chaining method takes a lambda node. It works with a `stream` pipeline like the `WHERE` statement in InfluxQL.
   * [`window`](/kapacitor/v1.3/nodes/window_node/) - has an empty chaining method.  It is configured using property methods. It works in a `stream` pipeline usually after the `from` node to cache data within a moving time range.

**Processing nodes**

Once the data set has been defined and filtered it can be passed to other nodes to process it or to transform it or to trigger a process based on changes within.

* Nodes for changing the structure of the data or for mixing together data sets
   * [`Combine`](/kapacitor/v1.3/nodes/combine_node/) - chaining method takes a list of one or more lambda expression. It can combine the data from a single node with itself.
   * [`Eval`](/kapacitor/v1.3/nodes/eval_node/) - chaining method takes a list of one or more lambda expressions. It evaluates expressions on each datapoint it receives and makes the results available to nodes that follow in the pipeline.  
   * [`GroupBy`](/kapacitor/v1.3/nodes/group_by_node/) - chaining method takes a list of one or more strings representing the tags of the series. It groups incoming data by tags
   * [`Join`](/kapacitor/v1.3/nodes/join_node/) - chaining method takes a list of one or more node variables. It joins data from any number of nodes based on matching time stamps
   * [`Union`](/kapacitor/v1.3/nodes/union_node/) -  chaining method takes a list of one or more node variables. It creates a union of any number of nodes

* Nodes for transforming or processing the datapoints within the data set.
   * [`Delete`](/kapacitor/v1.3/nodes/delete_node/) - empty chaining method. It relies on properties (field, tag) to delete fields and tags from datapoints.
   * [`Derivative`](/kapacitor/v1.3/nodes/derivative_node/) - chaining method takes a string representing a field for which a derivative will be calculated.
   * [`Flatten`](/kapacitor/v1.3/nodes/flatten_node/) - empty chaining method.  It relies on properties to flatten a set of points on specific dimensions.
   * [`InfluxQL`](/kapacitor/v1.3/nodes/influx_q_l_node/) - special node (see above). It provides access to InfluxQL functions. It should not be instantiated directly.
   * [`StateCount`](/kapacitor/v1.3/nodes/state_count_node/) - chaining method takes a lambda expression. It computes the number of consecutive points that are in a given state.
   * [`StateDuration`](/kapacitor/v1.3/nodes/state_duration_node/) - chaining method takes a lambda expression. It computes the duration of time that a given state lasts.
   * [`Stats`](/kapacitor/v1.3/nodes/stats_node/) - chaining method takes a duration expression. It emits internal stats about another node at the given interval.

* Nodes for triggering events, processes
   * [`Alert`](/kapacitor/v1.3/nodes/alert_node/) - empty chaining method. It relies on a number of properties for configuring the emission of alerts.  
   * [`Deadman`](kapacitor/v1.3//nodes/stream_node/#deadman) - actually a helper function on the stream node.  It is an alias for an `alert` that gets triggered when data flow falls below a specified threshould.
   * [`HTTPOut`](/kapacitor/v1.3/nodes/http_out_node/) - chaining method takes a string. It caches the most recent data for each group it receives, making it available over the Kapicator http server using the string argument as the final locator context
   * [`HTTPPost`](/kapacitor/v1.3/nodes/http_post_node/) - chaining method takes an array of strings.  It can also be empty. It posts data to http endpoints specified in the string array
   * [`InfluxDBOut`](/kapacitor/v1.3/nodes/influx_d_b_out_node/) - empty chaining method &ndash; configured through property setters.  It  writes data to InfluxDB as it is received
   * [`K8sAutoscale`](/kapacitor/v1.3/nodes/k8s_autoscale_node/) - empty chaining method. It relies on a number of properties for configuration. It triggers autoscale on Kubernetes&trade; resources.
   * [`KapacitorLoopback`](/kapacitor/v1.3/nodes/kapacitor_loopback_node/) - empty chaining method &ndash;  configured through property setters. It writes data back into the Kapacitor stream.  
   * [`Log`](/kapacitor/v1.3/nodes/log_node/) - empty chaining method. It relies on level and prefix properties for configuration. It logs all data that passes through it.

**User Defined Functions**

User defined functions are nodes that implement functionality defined by user programs or scripts that run as separate processes and that communicate with Kapacitor over sockets or standard system data streams.

   * [`UDF`](/kapacitor/v1.3/nodes/u_d_f_node/) - signature, properties and functionality defined by the user.

**Internally used nodes - Do Not use**

   * [`NoOp`](/kapacitor/v1.3/nodes/no_op_node/) - a helper node.  Do not use it!


# InfluxQL in TICKscript

InfluxQL occurs in a TICKscript primarily in a `query` node, whose chaining method takes an InfluxQL query string.  This will nearly always be a `SELECT` statement.

InfluxQL is very similar in its syntax to SQL.  When writing a query string for a TICKscript `query` node, generally only three clauses will be required: `SELECT`, `FROM` and `WHERE`.  The general pattern is as follows:

```SQL
SELECT {<FIELD_KEY> | <TAG_KEY> | <FUNCTION>([<FIELD_KEY>|<TAG_KEY])} FROM <DATABASE>.<RETENTION_POLICY>.<MEASUREMENT> WHERE {<CONDITIONAL_EXPRESSION>}  
```  
   * The base `SELECT` clause can take one or more field or tag keys, or functions.  These can be combined with mathematical operations and literal values.  Their values or results will be added to the data frame and can be aliased with an `AS` clause.  The star, `*`, wild card can also be used to retrieve all tags and fields.  
       * When using the `AS` clause the alias identifier can be accessed later on in the TICKscript as a named result by using double quotes.
   * The `FROM` clause requires the database, retention policy and the measurement name from which the values will be selected.  Each of these tokens is separated by a dot.
   * The `WHERE` clause requires a conditional expression.  This may include `AND` and `OR` boolean operators as well as mathematical operations.

**Example 25 &ndash; A simple InfluxQL query statement**
```javascript
batch
    |query('SELECT cpu, usage_idle FROM "telegraf"."autogen".cpu WHERE time > now() - 10s')
        .period(10s)
        .every(10s)
    |httpOut('dump')
```

Example 25 shows a simple `SELECT` statement that takes the `cpu` tag and the `usage_idle` field from the cpu measurement as recorded over the last ten seconds.

**Example 26 &ndash; A simple InfluxQL query statement with variables**
```javascript
var my_field = 'usage_idle'
var my_tag = 'cpu'

batch
    |query('SELECT ' + my_tag + ', ' + my_field + ' FROM "telegraf"."autogen".cpu WHERE time > now() - 10s')
        .period(10s)
        .every(10s)
    |httpOut('dump')
```
Example 26 reiterates the same query from Example 24, but shows how to add variables to the query string.

**Example 27 &ndash; An InfluxQL query statement with a function call**
```javascript
...
var data = batch
  |query('''SELECT 100 - mean(usage_idle) AS stat FROM "telegraf"."autogen"."cpu" WHERE cpu = 'cpu-total' ''')
    .period(period)
    .every(every)
    .groupBy('host')
...
```
Example 27 shows a `SELECT` statement that includes a function and mathematical operation in the `SELECT` clause, as well as the `AS` alias clause.

Note that the select statement gets passed directly to the InfluxDB API.  Within the InfluxQL query string field and tag names do not need to be accessed using double quotes, as is the case elsewhere in TICKscript.  However, the database name, and retention policy do get wrapped in double quotes. String literals, such as `'cpu-total'` are expressed inside the query string with single quotation marks.

See the [InfluxQL](/influxdb/v1.3/query_language/) documentation for a complete introduction.  

# Lambda expressions

Lambda expressions occur in a number of chaining and property methods.  Two of the most common usages are in the creation of an `eval` node or in defining threshold properties on an alert node.  They are declared with the keyword "lambda" followed by a colon: `lambda:`.  They can contain mathematical and boolean operations as well as calls to a large library of internal functions.  With many nodes, their results can be captured by setting an `as` property on the node.

Within lambda expressions TICKscript variables can be accessed using their plain identifiers.  Tag and field values from data series's can be accessed by surrounding them in double quotes.  Literals can also be used directly.  

**Example 28 &ndash; Lambda expressions**
```javascript
...
// Parameters
var info = 70
var warn = 85
var crit = 92
var infoSig = 2.5
var warnSig = 3
var critSig = 3.5
var period = 10s
var every = 10s

// Dataframe
var data = batch
  |query('''SELECT mean(used_percent) AS stat FROM "telegraf"."autogen"."mem" ''')
    .period(period)
    .every(every)
    .groupBy('host')

// Thresholds
var alert = data
  |eval(lambda: sigma("stat"))
    .as('sigma')
    .keep()
  |alert()
    .id('{{ index .Tags "host"}}/mem_used')
    .message('{{ .ID }}:{{ index .Fields "stat" }}')
    .info(lambda: "stat" > info OR "sigma" > infoSig)
    .warn(lambda: "stat" > warn OR "sigma" > warnSig)
    .crit(lambda: "stat" > crit OR "sigma" > critSig)

// Alert
alert
  .log('/tmp/mem_alert_log.txt')
```
Example 28 contains four lambda expressions.  The first expression is passed to the `eval` node.  It calls the internal stateful function `sigma`, into which it passes the named result `stat`, which is set using the `AS` clause in the query string of the `query` node.  Through the `.as()` setter of the `eval` node its result is named `sigma`.  Three other lambda expressions occur inside the threshold determining property methods of the `alert` node.  These lamda expressions also access the named results `stat` and `sigma` as well as variables declared at the start of the script.  They each define a series of boolean operations, which set the level of the alert message.  

# Summary of variable use between syntactic domains

<!-- see defect 1238 -->
| Occurrence | TICKscript | Query String | Lambda | InfluxQL Node|
|:-----------|:-----------|:-------------|:-------|:-------------|
| TICKscript variable <br/><br/>declaration examples: <br/><br/> `var my_var = 'foo'` <br/> `var my_num = 2.71` | Simply use the identifier<br/><br/> examples: <br/><br/> `var my_other_num = my_num + 3.14` <br/>`...`<br/>&nbsp;&nbsp;&nbsp;\|`default()`</br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`.tag('bar', my_var)` | Simply use the identifier  with string concatentation.  Note that strings will be interpreted as field and tag names.<br/><br/>example:<br/><br/>`query('SELECT ' + my_var + ' FROM "telegraf"."autogen".cpu)` | Simply use the identifier<br/><br/>example:<br/><br/> `.info(lambda: "stat" > my_num )` | identifier - Note that strings will be interpreted as field names<br/><br/>example:</br></br>\|`mean(my_var)` |
| Field, Tag name or Named result<br/><br/>example:<br/><br/> \|`eval(lambda: sigma("stat"))`<br/>&nbsp;&nbsp;&nbsp;`.as('sigma')`<br/><br/> \|`query('SELECT mean(usage_idle) AS mean ...')` |In method calls use single quotes <br/><br/>example:<br/><br/>\|`derivative('malloc')`| Use the identifier directly in the string<br/><br/>example:<br/><br/>\|`query('SELECT cpu, usage_idle FROM "telegraf"."autogen".cpu')`|Use double quotes<br/><br/>examples:<br/><br/>\|` eval(lambda: 100.0 - "usage_idle")`<br/><br/>\|`.info(lambda: "sigma" > 2 )`|Use single quotes <br/><br/>example:<br/><br/>\|`mean('used')`<br/><br/>|

# Gotchas

## Literals versus field values

Please keep in mind that literal string values are declared using single quotes.  Double quotes are reserved for accessing the values of tags, of fields and of named results.  In most instances using double quotes in place of single quotes will be caught as an error: `unsupported literal type`.  On the other hand, using single quotes when double quotes were intended, i.e. accessing a field value, will not be caught and, if this occurs in a lambda expression, may use the literal value instead of the desired value of a tag, of a field or of a named result.

As of Kapacitor 1.3 it is possible to declare a variable using double quotes and the parser will not flag it as an error.  For example `var my_var = "foo"` will pass so long as it is not used.  However, when this variable is used in a Lambda expression or other method call, it will trigger a compilation error.

<!--
## Incompatible node types
FIXME: not sure what was intended now by this heading.
-->
## Circular rewrites
<!-- defect 589 -->

## Alerts and ids
<!--  see email TICKscript - .id() - 2017-10-10 -->

# Where next?

See the [examples](https://github.com/influxdata/kapacitor/tree/master/examples) in the code base on Github.  See also the detailed use case solutions in the section [Guides](/kapacitor/v1.3/guides).  






**===================================================================================**
<br/>Previous documentation below<br/>
**===================================================================================**

Literals
--------

### Booleans

Boolean literals are the keywords `TRUE` and `FALSE`.
They are case sensitive.

### Numbers

Numbers are typed and are either a `float64` or an `int64`.
If the number contains a decimal it is considered to be a `float64` otherwise it is an `int64`.
All `float64` numbers are considered to be in base 10.
If an integer is prefixed with a `0` then it is considered a base 8 (octal) number, otherwise it is considered base 10.

Valid number literals:

* 1 -- int64
* 1.2 -- float64
* 5 -- int64
* 5.0 -- float64
* 0.42 -- float64
* 0400 -- octal int64


### Strings

There are two ways to write string literals:

1. Single quoted strings with backslash escaped single quotes.

    This string `'single \' quoted'` becomes the literal `single ' quoted`.

2. Triple single quoted strings with no escaping.

    This string `'''triple \' quoted'''` becomes the literal `triple \' quoted`.

### Durations

TICKscript supports durations literals.
They are of the form of InfluxQL duration literals.
See https://docs.influxdata.com/influxdb/v1.3/query_language/spec/#literals

Duration literals specify a length of time.
An integer literal followed immediately (with no spaces) by a duration unit listed below is interpreted as a duration literal.

#### Duration unit definitions

 Units  | Meaning
--------|-----------------------------------------
 u or µ | microseconds (1 millionth of a second)
 ms     | milliseconds (1 thousandth of a second)
 s      | second
 m      | minute
 h      | hour
 d      | day
 w      | week

Statements
----------

A statement begins with an identifier and any number of chaining function calls.
The result of a statement can be assigned to a variable using the `var` keyword and assignment operator `=`.

Example:

```javascript
var errors = stream
    |from()
        .measurement('errors')
var requests = stream
    |from()
    .measurement('requests')
// Join the errors and requests stream
errors
    |join(requests)
        .as('errors', 'requests')
    |eval(lambda: "errors.value" / "requests.value")
```

Format
------

### Whitespace

Whitespace is ignored and can be used to format the code as you like.

Typically property methods are indented in from their calling node.
This way methods along the left edge are chaining methods.

For example:

```javascript
stream
    |eval(lambda: "views" + "errors")
        .as('total_views') // Increase indent for property method.
    |httpOut('example') // Decrease indent for chaining method.
```

### Comments

 Basic `//` style single line comments are supported.
