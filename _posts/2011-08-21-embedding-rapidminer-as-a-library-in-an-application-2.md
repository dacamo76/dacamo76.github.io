---
layout: post
tags: [data mining, open source , rapidminer, java]
category: blog
title: Embedding RapidMiner as a library in an application (Part II)
date: 2011-08-21
comments: true
share: true
---

My last post described [how to create attributes in Java][last post], the first step in the process of creating a RapidMiner example set. This entry will show the steps necessary to get from attributes to a full-fledged example set.

First, let’s review the workflow for creating an example set:


![Flow for creating an example set](/images/2011/07/rapidminer_flow2.png "Flow for creating an example set")

We already know how to create attributes, the next step is to create an example table. Then we’ll populate the example table with data before finally creating the example set.

<!-- more -->

What is an ExampleTable?
--------------------------

Before we create an example table, let’s review why we need one in the first place. We know we need an ExampleSet to apply a model. So why do we need the intermediary step of an example table? According to the RapidMiner docs, an [ExampleTable](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/ExampleTable.html), is the core supplier for example sets. An ExampleTable contains all data like in a database management systems and all ExampleSets are only views on the data. This description tells us that creating the ExampleTable is the all important step. The ExampleSet is secondary, as will be confirmed later in the post. So that’s the why an ExampleTable?. Now let’s get to the how?. For our purposes, all we need to know is that an example table stores attributes and rows, with the attributes acting as column headings. Basically, you can think of an example table as your typical spreadsheet with columns and rows. (It actually contains more information than that, as always, refer to the [data core](http://rapid-i.com/wiki/index.php?title=Data_core) wiki entry for more information.) As for the internals, an example table stores the rows as [DataRow](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DataRow.html) objects. After creating an ExampleTable, we need to populate it with DataRows. This seems like a perfect time to take a closer look at the DataRow interface. Let’s leave example tables behind for a moment and take a closer look at data rows.

What is a Data Row?
-------------------

As mentioned above, ExampleTables store each row as a [DataRow](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DataRow.html). A DataRow is essentially an array. The recommended implementation is the [DoubleArrayDataRow], which as its name implies, is an internal array of double values.  Other DataRow types can be found in the DataRowFactory documentation. For the remainder of this post, unless otherwise stated, I will use DoubleArrayDataRow and DataRow interchangeably. Rarely have I used an implementation other than DoubleArrayDataRow. In a few circumstances I’ve used the [SparseArrayDataRow](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/SparseDataRow.html) where my data was **extremely**, and I mean extremely, sparse, but only after tests showed it performed much faster. You can safely stick to a DoubleArrayDataRow and not lose any sleep. In a DoubleArrayDataRow, each array index corresponds to a column heading. Recall that column headings are represented by attributes in the example table. If a column contains nominal values, or non numeric values for that matter, each value must be converted into a double before it can be stored in the data row. Nominal attributes contain an internal mapping strictly for this purpose.

Attribute Mappings
------------------

To better understand attribute mappings, we’ll construct an example using the following training data that corresponds to the [attributes we created in the last post.][last post]

![RapidMiner example set values](/images/2011/08/rapidminer-example-values.png "RapidMiner example set values")

As you can see, “teamID”, “leader”, “leader changed”, and “structure” are all nominal attributes. We ignore the attribute “label”, since it’s the value we’re trying to predict, and we obviously don’t have access to it when creating our example set. For illustration purposes, the image below represents a possible mapping of the “leader” attribute.

![Attribute mapping for Leader](/images/2011/08/attribute_mappings.png "Attribute mapping for Leader")

The value “Mr. Miller” maps to 0. The value “Mrs. Green” maps to 1, and so on. We’ll assume the index of the “leader” attribute in our ExampleTable is 3. This means index 3 of each row in the table contains a 0 for instances of “Mr. Miller” and a 1 for instances of “Mrs. Green”. We use the mapping to get the double value from the nominal value and vice versa. Now that we know about mappings, let’s get back to creating a data row.

Creating Data Rows and Example Tables
-------------------------------------

So far we’ve learned how to create attributes and that these attributes will act as column headings in an example table. We know that nominal attributes contain a mapping that allows us to convert a value between its nominal representation and a double.* We also know that in order to create a data row we must first convert all nominal attribute values to doubles. We then use the data rows to populate an example table. Having said all that, we still don’t know how to create a DataRow or an ExampleTable. So let’s get to it.

_* The method getMapping() is part of the Attribute interface, so all implementations must support the method, even numeric attributes. Attributes with no concept of a mapping return an UnsupportedOperationException when invoking getMapping(). Avoid this exception by calling isNominal() beforehand, thus guaranteeing an attribute is nominal._

Let’s begin by creating the aforementioned DoubleArrayDataRow. First we map each nominal value to a double and create an array of those double values. Then we pass this array to the [DoubleArrayDataRow constructor][2]. An example on how to [manually populate an example table](http://rapid-i.com/wiki/index.php?title=Integrating_RapidMiner_into_your_application#Transform_data_for_RapidMiner) this way can be found in the RapidMiner wiki. Here is the relevant code copied from the wiki. It shows how to create an example table and populate it with data rows. In this specific example, the variable label is a nominal attribute and attributes is a list of Attributes with label as the last element. Every other attribute in the list is a double.

~~~ java
// create table
MemoryExampleTable table = new MemoryExampleTable(attributes);
// fill table (here: only real values)
for (int d = 0; d < getMyNumOfDataRows(); d++) {
  double[] data = new double[attributes.size()];
  for (int a = 0; a < getMyNumOfAttributes(); a++) {
    // fill with proper data here
    data[a] = getMyValue(d, a);
  }
  // maps the nominal classification to a double value
  data[data.length - 1] = label.getMapping().mapString(getMyClassification(d));
  // add data row
  table.addDataRow(new DoubleArrayDataRow(data));
}
~~~

Let’s see if we can follow what’s going on. In line 2 we create the `ExampleTable`.
The for loop in line 4 loops over all the rows in the input data set.
Line 5 creates a new array of type double with length equal to the number of columns/attributes in the row. The internal for loop in line 6 iterates over each column of the row. Remember, the first n-1 columns contain double values, so we can just add them to the array without worries. However, the last attribute, label, is nominal. We need to convert it to a double before we can add it to the array. Let’s break down this step, shown in line 12. First we get the mapping for the attribute by calling [label.getMapping()] which returns a [NominalMapping]. We then call [mapString()] on this mapping, passing the nominal value as a parameter. The method `getMyClassification(d)` is just a helper method used in the example that returns the value at index d of the incoming row. Quoting the javadocs for mapString, _mapString Returns the internal double representation (actually an integer index) for the given nominal value. This method creates a mapping if it did not exist before._ So now that we have a mapping for our nominal value, all that’s left is to add it as the last element of the array. The assignment in line 11 does just that. Finally, in line 13 we create the [DoubleArrayDataRow] and add it to the example table. This is repeated for every row in the incoming data.

Got it? I didn’t the first time. Or the second. There’s a lot going on for something so simple.
You have to really think about what the code is doing in order to understand each line.
I had to look at the code line by line and inspect the source code of the corresponding classes to see what was really going on. There are too many levels of abstraction. Nested for-loops. Array manipulation by indices.
Manually checking the data type for each Attribute. Manually getting nominal mappings if necessary.
One mistake and hello nasty bugs.
Not to mention, all this in a trivial example where all attributes are numeric except the last one.
Now, I know this code is just an example in the wiki.
It’s not a complaint against the example, it’s mostly a warning to all the copy-paste programmers.
Don’t do it. Please. Refactor. That way you’ll understand exactly what’s going on.

Refactoring for Simplicity
--------------------------

Let’s take our own advice and see how we can refactor this code to make it easier to understand.
At first glance, our goals are:

- Use lists instead of arrays
- Use For-each loops instead of for loops
- Abstract away the attribute types and mappings

We can eliminate the outer for loop by using a list instead of an array for our incoming data.
We can eliminate the inner for loop by representing a row as a list of Strings.
Doing this, our input data becomes a `List<List<String>>`. That’s still not ideal.
It’s difficult looking at all those angle brackets. Welcome to Java!
Not even the famous so-called [diamond] will clean up the syntax.
But, we can define a Row interface that represents a `List<String>` and clean up the syntax.

~~~ java
public interface Row extends List<String> {}
~~~

So now we can represent our incoming data as a list of rows, `List<Row>`, cleaner code with the added advantage of clearly stating our intent. Assuming the variable inputData contains our incoming list of rows, the code now looks like this.

~~~ java
// create table
MemoryExampleTable table = new MemoryExampleTable(attributes);
// fill table (here: only real values)
for (Row row : inputData) {
  double[] data = new double[attributes.size()];
  int index = 0;
  for (String s : row) {
    // fill with proper data here
    data[index] = Double.valueOf(s);
    index++;
  }
  // maps the nominal classification to a double value
  data[data.length - 1] = label.getMapping().mapString(row);
  // add data row
  table.addDataRow(new DoubleArrayDataRow(data));
}
~~~

Ok. We got rid of the for loops, but at the expense of adding an index variable whose sole purpose is to keep track of the column index.
Not good. Let’s eliminate the index. We need to get rid of the array named data and convert it to a List.
We should also abstract out the creation of a DataRow, we shouldn’t have to loop through each column of an input row. We also didn’t address the issue of the abstraction level of the attributes, we’re still retrieving the mappings manually. If we abstract out the data row creation, we get the added bonus of also abstracting out the manual mapping manipulation. But how should we abstract out the data row creation? Let’s follow the [Single Responsibility Principle] and create a new class whose sole purpose is creating DataRows. This new class should create a DataRow given a list of Attributes and a list of Strings containing the data values. Lucky for us, such a class exists, with a few caveats. The class is the [DataRowFactory]. Let’s get more acquainted with the DataRowFactory, then I’ll explain the caveats.

Enter the DataRowFactory
-------------

Before diving into the details of the DataRowFactory, let’s show our updated example code with the DataRowFactory.

~~~ java
// create table
MemoryExampleTable table = new MemoryExampleTable(attributes);
DataRowFactory factory = new DataRowFactory(DataRowFactory.TYPE_DOUBLE_ARRAY, '.');
for (Row row : inputData) {
  String[] data = toArray(row);
  Attribute[] atts = toArray(attributes);
  DataRow dataRow = factory.create(data, atts);
  table.addDataRow(dataRow);
}

public <T> T[] toArray(List<T> list) {
  return (T[]) list.toArray();
}
~~~

That looks much better. We don’t need to know or care about nominal value mappings. We create the factory in line 3. The DataRowFactory [constructor][3] takes two arguments. The first is the type of data row. As always, we are using the DoubleArrayDataRow. The second constructor parameter is the decimal point character used in numeric fields. In our example it’s the period, or full stop. Once we create the factory, all that’s left it to iterate over our input data, create the data rows, and add them to the example table. This is shown in lines 4 – 9. The [create()][4] method takes an array of Strings and an array of Attributes as parameters. Line 5 converts a Row into an array and line 6 converts the Attribute list into an array. These arrays are then used in line 7 to create the DataRow which is added to the table in line 8.

Now life is good, we have a working exampleTable. But, what we need in order to use the RapidMiner libraries is an ExampleSet. Remember the figure at the beginning of the post, the one that describes the workflow? We are finally at the end of the workflow. Having our example table, all that’s left to do is create the example set.

Finally, An ExampleSet Sighting
-------------------------------

This is the easiest step. In my last post I mentioned [attribute roles][last post] and said we didn’t need to worry about them until we created an example set. Well, now it’s time to use the roles. Let’s rehash the code used to create the roles.

~~~ java
Map<Attribute, String> roles = new HashMap<Attribute, String>();
roles.put(teamID, Attributes.ID_NAME);
~~~

That’s it. Simple, right? Creating the ExampleSet is even easier.

~~~ java
ExampleTable table = createExampleTable();
table.createExampleSet(roles);
~~~

That’s it. We now have an ExampleSet with teamID as the id attribute. The full listing now looks like this.

~~~ java
public interface Row extends List<String> {}
~~~

~~~ java
public <T> T[] toArray(List<T> list) {
  return (T[]) list.toArray();
}
~~~

~~~ java
Map<Attribute, String> roles = new HashMap<Attribute, String>();
roles.put(teamID, Attributes.ID_NAME);

MemoryExampleTable table = new MemoryExampleTable(attributes);
DataRowFactory factory = new DataRowFactory(DataRowFactory.TYPE_DOUBLE_ARRAY, '.');
for (Row row : inputData) {
  String[] data = toArray(row);
  Attribute[] atts = toArray(attributes);
  DataRow dataRow = factory.create(data, atts);
  table.addDataRow(dataRow);
}
table.createExampleSet(roles);
~~~

You can use this code to create your example sets and continue on with applying your models and all will be good.
But, if you want to simplify the ExampleTable creation, have more manageable code, and don’t mind getting your hands dirty, read on.

Can We Do Better?
-----------------

Lines 7 and 8 in the code above convert our lists to arrays to pass into the [create(String\[\], Attribute\[\])][4] method.
The available DataRowFactory.create() methods all take arrays as an argument.
Our goal of [preferring lists to arrays][prefer lists] was not met.
Given that RapidMiner is [open source][agpl], we can modify the source and [add a method to DataRowFactory that takes Lists as its arguments][bug 683].
In my projects I do just that.
The diff has not been applied in the main RapidMiner core, so if you want to go this route you have to patch the source and compile your own RapidMiner jar.
After applying the [diff][bug 683 diff], the resulting sample code looks like this:

~~~ java
// create table
MemoryExampleTable table = new MemoryExampleTable(attributes);
DataRowFactory factory = new DataRowFactory(DataRowFactory.TYPE_DOUBLE_ARRAY, '.');
for (Row row : inputData) {
  DataRow dataRow = factory.create(row, attributes);
  table.addDataRow(dataRow);
}
~~~

This code is more compact and easier to reason about.
You can understand what it’s doing by just looking at it. Try reading it aloud.
“Create a new example table. Create a new data row factory.
For each row in the input data, use the factory to create a new data row and add it to the example table.”

If you don’t feel like compiling your own RapidMiner jar, I uploaded a simple example of a class that [encapsulates the DataRowFactory interface][gist 1] and exposes only a few methods. The gist is a complete working copy with a [usage example][gist 2] and the [expected output][gist 3]. You can [download all the files][gist full] and use it as a replacement for the DataRowFactory. Lines 31 - 35 in the usage example file show how to create an ExampleTable. It is reproduced here.

~~~ java
MemoryExampleTable table = new MemoryExampleTable(attributes);
DataRowFactory2 factory =
        DataRowFactory2.withFullStopDecimalSeparator(attributes);
for (Row row : inputData) {
  DataRow dataRow = factory.createRow(row);
  table.addDataRow(dataRow);
}
~~~

The main difference is on lines 2 and 5. In line 2 we create the factory with a more readable, if slightly more verbose, static factory method `DataRowFactory2.withFullStopDecimalSeparator(attributes)`. The method name tells us exactly what kind of factory we are creating, a factory which treats numeric attributes with a ‘.’ as the decimal separator. Besides the [inherent advantages of static factory methods][prefer static factory methods], we get the added bonus of IDE support, the method `.withFullStopDecimalSeparator` will be found during auto-completion, so we can decide right then what type of factory we need, no need to go digging in source files to find `static final int`s to pass as parameters. Finally, line 5 shows how the factory creates a new DataRow from an `Iterable<String>`, we no longer pass in the attributes on every invocation.

Let me expand on the importance of the last two points. I mentioned there were some caveats when using the DataRowFactory. The first caveat is the lack of type safety when using static final ints to represent the different DataRow types. The example [DataRowFactory2][gist 4] adds type safety by using static factory methods for each DataRow type. The second caveat is that you must pass a list of Attributes to the DataRowFactory.create() method on each invocation. The Attribute list (not to mention the attributes themselves) must not be modified between calls to create(). The DataRowFactory2 tries to solve this by making a private copy of the attributes. The createRow() method then uses this copy of the attributes to create the DataRow. This assures us the actual list does not change, but does not guard against changes in individual elements (Attribute) of the list. Thus, extra care must still be exercised in multi-threaded environments.

All Roads Lead to ExampleSets
-----------------------------

Whichever way you decide to go with creating the DataRows. The bottom line is we have a fully operational ExampleTable which we can use to create an ExampleSet and be on our way toward applying RapidMiner models directly in our Java programs.

[last post]: {% post_url 2011-07-22-embedding-rapidminer-as-a-library-in-an-application %}
[DoubleArrayDataRow]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DoubleArrayDataRow.html
[2]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DoubleArrayDataRow.html#DoubleArrayDataRow(double[])
[label.getMapping()]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/Attribute.html#getMapping()
[NominalMapping]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/NominalMapping.html
[mapString()]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/NominalMapping.html#mapString(java.lang.String)
[diamond]: http://download.oracle.com/javase/tutorial/java/generics/gentypeinference.html#type-inference-instantiation
[Single Responsibility Principle]: http://en.wikipedia.org/wiki/Single_responsibility_principle
[DataRowFactory]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DataRowFactory.html
[3]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DataRowFactory.html#DataRowFactory(int,%20char)
[4]: http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DataRowFactory.html#create(java.lang.String[],%20com.rapidminer.example.Attribute[])
[prefer lists]: http://books.google.com/books?id=ka2VUBqHiWkC&lpg=PA119&ots=yYDhRfvZU0&dq=effective%20java%20prefer%20lists&pg=PA119#v=onepage&q&f=false
[bug 683]: http://bugs.rapid-i.com/show_bug.cgi?id=683
[agpl]: http://rapid-i.com/content/view/29/215/lang,en/
[bug 683 diff]: http://bugs.rapid-i.com/attachment.cgi?id=113&action=diff
[gist 1]: https://gist.github.com/1138546#file_data_row_factory2.java
[gist 2]: https://gist.github.com/1138546#file_usage_example.java
[gist 3]: https://gist.github.com/1138546#file_output.txt
[gist full]: https://gist.github.com/1138546
[prefer static factory methods]: http://books.google.com/books?id=ka2VUBqHiWkC&lpg=PR9&dq=effective%20java%20item%201&pg=PA5#v=onepage&q&f=false
[gist 4]: https://gist.github.com/1138546#file_data_row_factory2.java
