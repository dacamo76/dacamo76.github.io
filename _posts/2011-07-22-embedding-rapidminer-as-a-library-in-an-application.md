---
layout: post
tags : [data mining, open source, rapidminer, java]
category: blog
date: 2011-07-22
comments: true
share: true
---

I was involved recently in a project deploying real-time predictive models created using RapidMiner. As the documentation is sparse, I struggled at first grasping the RapidMiner internals. But once I got the basics of the RapidMiner data model, the process was pretty straightforward. So I decided to write the basic steps for integrating RapidMiner in a Java application along with a basic understanding of the data model and a few tips and gotchas I encountered along the way.

<!-- more -->

Let me preface this post by stating that RapidMiner was not developed as a library, but rather as a data mining platform, so one should expect a few unexpected behaviors (does that make it actually expected behavior?) when embedding it into a project as a library. Also, I only describe how to deploy a model or process, not how to create one. (Since both RapidMiner processes and models accept an example set and return a modified example set, for simplicity, I will refer to both as models from now on.)

Basic API
---------

The following interfaces are the ones you will spend the most time with when deploying an embedded RapidMiner application:

- [Attribute](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/Attribute.html)
- [DataRow](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DataRow.html) (Actually an abstract class)
- [ExampleTable](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/ExampleTable.html)
- [ExampleSet](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/ExampleSet.html)

It’s usually unnecessary to know the implementations, but if you want a better understanding of how the RapidMiner data model works under the hood, you can look at the source for the following concrete classes:

* [DoubleArrayDataRow](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/DoubleArrayDataRow.html)
* [MemoryExampleTable](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/table/MemoryExampleTable.html)

As always, refer to the RapidMiner [data core](http://rapid-i.com/wiki/index.php?title=Data_core) wiki entry for a more in-depth look.

Enough with the introduction, let’s get to the point of this post, how to apply a model to your data. In RapidMiner, this means converting your data into an ExampleSet and passing it into your model. Below is the workflow for creating an example set.

![Flow for creating an example set](/images/2011/07/rapidminer_flow2.png "Flow for creating an example set")


First create the attributes, then create an example table with the attributes as the column headings. The next step is to fill the example table with data before finally creating the example set. Now that we’re familiar with the steps involved, let’s continue by exploring the attributes further.

Introducing attributes
-----------------------

There are two required properties for each attribute:

1. Name
2. Data type

The name is self-explanatory, it is just the name of the attribute. The data type tells us what type the data represents. The major types being nominal and numeric. Other types such as Date and more specific sub-types of numeric and nominal are also available. You can find other attribute data types in the [Ontology](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/tools/Ontology.html) class.

A third attribute property that deserves mention is Role. The role lets us know if that attribute has a special meaning. Is it a label? Is it an id?. Does it represent a weight? The most common roles are:

- Regular
- Id
- Prediction
- Label
- Cluster
- Weight

Classification and clustering models append a prediction or cluster role, respectively, when applied to an example set.  Supervised learning models use the label role. The rest of the roles are often used in input data sets, which is what we’re interested in. You can find all the roles as static fields in the [Attributes](http://rapid-i.com/api/rapidminer-5.1/index.html?com/rapidminer/doc/package-summary.html) interface.

I should note that a role isn’t a direct property of an attribute, rather it’s assigned when the example set is created. You can think of an example set as a class that contains an example table, attributes, and a mapping of attributes to roles.

Creating Data Set In RapidMiner GUI
-----------------------------------

I hate explanations that don’t show you how to get a working example with non trivial data. So let’s create a RapidMiner data set that we can use to create attributes and eventually apply a model to. We’ll use one of the data generating operators in the RapidMiner GUI to accomplish this. Go to *Utility -> Data Generation -> Generate Team Data Profit Data* and drag it onto the main canvas. Then go to *Data Transformation -> Name and Role Modification -> Set Role* and drag it onto the canvas. Connect the Generate Team Data Profit Data output port to the Set Role input port and connect the first Set Role output port to the result port. Your canvas should look similar this image:

![Example RapidMiner canvas](/images/2011/07/rapidminer-canvas.png "Creating a sample example set")

Now in the *Parameters* tab, set the *Set Role* parameters like so:


![Screenshot of Setting teamID attribute as the id](/images/2011/07/rapidminer-set-role.png "Setting teamID attribute as the id")


This will convert the teamID attribute into an id for the example set.

Pressing the run button will create an example set with random data with six regular attributes and two attributes with special roles. These are the attributes we are going to reference throughout this example.

![Attributes in RapidMiner Meta Data View](/images/2011/07/tts.png "Attributes in RapidMiner Meta Data View")

There are eight attributes in this data set, but we can ignore the label attribute, since that is the value we are trying to predict. The teamID attribute is the id of the row and the rest of the attributes are regular attributes.

We now have a data set filled with random data that we can use in our examples. Let’s see how we can create these attributes in our code.

Creating Attributes
-------------------

When creating a model in the RapidMiner GUI, take note of the attribute names, types, and roles. Save the information in a format that is easy to parse as you will need this information to create the attributes in your program. In my implementations, I tend to encode the attribute representations in a CSV string or JSON, both of which are easily created and parsed.
 Let’s say we settled on using JSON, this is a snippet of what a representation might look like:

{% highlight json %}
    [
      {
         "teamID":
              {
               "type" : "nominal",
               "role" : "id"
              }
       },
       {
         "size":
              {
                "type" : "integer"
              }
       },
       ...
       {
         "structure":
              {
                "type" : "nominal"
              }
       }
    ]
{% endhighlight %}

We parse this in our program and knowing the name, type, and role of each attribute, we can go ahead and create the attributes. In the interest of space in a blog post that is quickly becoming longer than expected, I will try to only post the RapidMiner specific source code. So, I will assume there is an interface

{% highlight java %}
interface AttributeRepresentation {
    List<Attribute> newAttributeList();
}
{% endhighlight %}

with an implementation that takes the JSON string as a constructor parameter. Whenever we need a new instance of the attributes, we call newAttributeList(). Internally, the class uses an [AttributeFactory](http://rapid-i.com/api/rapidminer-4.4/com/rapidminer/example/table/AttributeFactory.html) to create the attributes, the snippet below shows how you would create the teamID and size attributes.

{% highlight java %}
Attribute teamID = AttributeFactory.createAttribute("teamID", Ontology.NOMINAL);
Attribute size = AttributeFactory.createAttribute("size", Ontology.NUMERICAL);
{% endhighlight %}

The rest of the attributes are created in the same way.


Creating Attribute Roles
------------------------
Now that we have a list of attributes, let’s see how to define each attribute’s role. As we noted earlier, the attribute role is not part of the actual attribute and we don’t need to worry about the roles until we create an example set.

Even if we don’t need the attribute roles just yet, let’s use the JSON representation to select the attributes with special roles to define a map which we’ll later use when creating the example set. An example set expects a map with an Attribute as the key type and String as the value type. The value must correspond to one of the predefined static strings in the [Attributes](http://rapid-i.com/api/rapidminer-5.1/com/rapidminer/example/Attributes.html) interface. (Aside: Wow, how awful were APIs pre Java 1.5?! Seriously, how did programmers get along without Enums. Worse, why are there still new API’s that don’t use enums instead of static final fields ?! I understand legacy API’s, like RapidMiner, but newer API’s ?! Is asking for type safety and a little readability too much? ) All right, cooling off.... ok.. where were we? Thats right, I was talking about the attribute role map. Anyway, the following code shows how to set the role of id to the teamID attribute.

{% highlight java %}
Map<Attribute, String> roles = new HashMap<Attribute, String>();
roles.put(teamID, Attributes.ID_NAME);
{% endhighlight %}

Once we’ve created the attributes and their corresponding roles, we can move on to creating the example table. I’ll continue with the rest of the process in my next post.

For now, our code for creating the attributes and attribute roles should look something like this:

{% highlight java %}
final class JSONAttributeRepresentation implements AttributeRepresentation {

  private final List<Attribute> attributes;

  public JSONAttributeRepresentation(Reader JSONReader) {
  // Read in json however you want
  // Get name, type, and role for each
  // for each create attribute with the name and type
    // Attribute att = AttributeFactory.createAttribute(name, type)
    // atributes.add(att);
  }

  @Override
  public List<Attribute> newAttributeList() {
    List<Attribute> copy = new ArrayList<Attribute>();
    for(Attribute attribute: attributes) {
      Attribute att = AttributeFactory.createAttribute(attribute.getName(),
                                                       attribute.getValueType());
      copy.add(att);
    }
    return copy;
}
{% endhighlight %}


The main coding driving the application would then be similar to the following:


{% highlight java %}
AttributeRepresentation rep = new JSONAttributeRepresentation(Reader jsonReader);
List<Attributes> attributes = rep.newAttributeList();
Map<Attribute, String> roles = new HashMap<Attribute, String>();
Attribute id = attributes.get(0);
roles.put(id, Attributes.ID_NAME);
{% endhighlight %}