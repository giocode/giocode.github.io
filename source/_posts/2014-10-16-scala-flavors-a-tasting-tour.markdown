---
layout: post
title: "Scala Flavors: a tasting tour Part 1"
date: 2014-10-15 17:50:32 -0700
comments: false
categories: scala tutorials
---

This a is the first part on a series of tutorials on Scala's language features. This series are written for anyone who are new to Scala, knows its syntax a bit but are excited and scared at the same type by the bipolarity of this `beautiful` language. In this first part, we will cover two points: {% img left ../images/scalabovolo.png 200 300 %}


- Scala's type system
- Scala's purely object-oriented language features


## Starting strong and safe

Scala's type system is strong, static and safe. What does that mean? 

### Values, variables and Types

There are two possible ways to associate a name to a value or an expression. The first one is `val`, which is used to permanently bind a value to an indentifier. Once a val is set, its value can no longer be reassigned. In other words, vals are like constants. 

<!-- more -->

{% codeblock lang:scala %}
scala> val y = 5.0
y: Double = 5.0
scala> val message = "Holla scala"
message: String = Holla scala
{% endcodeblock %}

Changing the content of `message` to _"Bye scala"_ is not possible. Instead, it will generate a compilation error because of a reassignment to a val.

The second one is `var`. You use a var for variables. Unlike vals, the value that a `var` refers to can be changed or reassigned in the future. For example, the `numberOfRabbits` enrolled to your farming program will probably keep growing.

{% codeblock lang:scala %}
scala> var numberOfRabbits = 6
numberOfRabbits: Int = 10
scala> numberOfRabbits = numberOfRabbits * 4
numberOfRabbits: Int = 40
scala> numberOfRabbits = numberOfRabbits * 4
numberOfRabbits: Int = 160
{% endcodeblock %}

Once a var is declared, its type however cannot be modified. In other words, numberOfRabbits can only contain an integer. Reassigning a string to it will not compile.

{% codeblock lang:scala %}
numberOfRabbits = "will not compile"
// Expression of type String doesn't comform to expected type Int.
{% endcodeblock %}

We have not yet explained what these types are about. Since Scala is a statically typed language, any values and variables are initialized with their type and content. Looking at the REPL outputs above, we did not declare any type when defining `y`, `message` or `numberOfRabbits`. So how did Scala know? Well, Scala is smart enough to infer the type from the right hand side of the assignment. Scala understood that "Holla scala" is definitely a string whereas the decimal 5.0 looks like a Double.

In principle, a formal way to define vals and vars is like this. 

{% codeblock lang:scala %}
// The right side must return a value of the declared type.
val identifier: type = expression
{% endcodeblock %}

In some situations, we want to declare vals and vars without giving them an initial value. In that case, the type declaration is mandatory. Otherwise, the rule of thumb is to omit the types when they can be readily inferred with a glimpse and to declare them to add clarity to your code. 

{% codeblock lang:scala %}
val z: Boolean;  // Scala follows you
val w; 			// Scala will cry out loud
{% endcodeblock %}

You must be still asking yourself why we need vals when we can use vars. Be patient as we will come back to this question when we will later discuss the choice between mutable and immutable data. For now, let's get our feet wet by reviewing some basic types and operators. 

### Basic Scala types and operators
There are 3 types of operators: 

- binary operators which take two operands on their left and right sides 
- unary prefix operators which are applied to one operand on its right
- unary postix operators which have one operand on its left 

Here are some examples of binary and unary operators for the basic numeric types. 

{% codeblock lang:scala %}
// Binary operators
scala> 20 * 1.20
Double = 24.0  

scala> 3 / 2
Int = 1

scala> val larger = 3 max 7 
larger: Int = 7

scala> val remainder = 11 % 3
remainder: Int = 2

// A unary prefix operator
scala> val decimalNumber = - 102.876
Double = - 102.876

// A unary postfix operator
decimalNumber toInt
Int = - 102
{% endcodeblock %}


Similarly, logical operators on Boolean values can be either unary or binary.

{% codeblock lang:scala %}

scala> val veracity = true
veracity: Boolean = true 

scala> var lie = ! veracity	 // a prefix operator
lie: Boolean = false

scala> veracity && lie
Boolean = false

scala> val halftruth = veracity || lie
halftruth: Boolean = true
{% endcodeblock %}


In the following, let's shed some lights on the real nature of values, types and operators in Scala. Ready? Unveil! 

## Objects orient Scala's type system
Behind the curtain, Scala is a purely object-oriented programming language. In fact, every value is an object of some type. Each type is represented by a class which serves as a blueprint for all values of that type. Moreover, every operation in Scala is a method call on some object.

{% blockquote %}
Types are classes. Values are objects. Operations are method calls. 
{% endblockquote %}


Primitive types such as String and Double are wrapper classes for Java's primitive types. Let's look at some string concatenation examples to understand this.

{% codeblock lang:scala %}
//String concatenations 
scala> "apples" + " oranges" 
String =  apples oranges 

scala> o * 3  
String =  oranges  oranges  oranges 

scala> 3 + " oranges"
3 oranges 
{% endcodeblock %}

If Scala was not awesome, we would have to rewrite the above concatenations as formal method calls:

{% codeblock lang:scala%}
// More verbose concatenations
scala> "apples".+(" oranges")
scala> "oranges".*(3)  
scala> 3.+(" oranges")
{% endcodeblock %}

Luckily, any methods with single parameters can be written in Scala as if they were built-in infix operators. Therefore, we can omit the `.` notation and the parentheses enclosing the single parameter. Instead, we add a space before and after the method name. The cool thing about that is the flexibility to name your binary operations in any way you want as long as the name is a valid identifier. 


### Method overloading 

Since Scala is object-oriented, it supports method overloading so that different methods with the same name can be defined provided their signatures are different. 

Looking at `"apples".+(" oranges")`, we see that the concatenation is performed by calling the method `+` defined on the String object _"apples"_ with the string parameter _" oranges"_. What gets return is the new String object _"apples oranges"_. Furthermore, many other signatures exist for the same operator `+`. 
When it comes to `"oranges".*(3)`, `*` is another method defined on String and it returns the _"oranges"_ string concatenated 3 times. If you look at the Scala documentation for StringOps, you will find the signature of this method to be `def *(n: Int): String`. 

Of course, Scala will complain if you call a method on one object and the method signature is not present in the class definition. 

{% codeblock lang:scala%}
scala> "apples" * 3.0
error: def *(x: Double): String cannot be resolved on String object

scala> 3 * "oranges"
error: def *(s: String): String cannot be resolved on Int object
{% endcodeblock %}

The only exception to this rule is if an _implicit conversion_ is defined for the corresponding type, then the method call can be forwarded to the converted type. But let's keep things simple for now and perhaps, we'll discuss about implicits later on.



### Create Abstract Data Types with Classes 
It is time to create your own types. Classes are used to abstract and parameterize data types that serve as building blocks for your Scala program. 

**Class definition:** you define a class using the `class` keyword followed by the class name and class parameters. Optionally, you can add statements and expressions in the class body delimited by a `{}` block to do one of the following: 

- declare and initialize value and variable fields  
- define public methods
- define private methods helper or nested classes
- execute some code, e.g. I/O actions


Let's look at a culinary example in which we first define a _Recipe_ class.

{% codeblock lang:scala %}
class Recipe(n: String, s: Int = 1, instructions: List[String] = Nil) {
  // some val and var fiels declarations and initializations
  val name = n
  val serves = s

  var description: String = ""
  var ingredients: Map[String, (Double, String)] = _
}
{% endcodeblock %}

**Class constructor** asks you to specify the parameters `s`, `n`, and `instructions` to create the `Recipe` objects. For example, you call the constructor method and pass the name, the number of serves and instruction list. As a result, you get a new _easyNoodleRecipe_ object. 

{% codeblock lang:scala %}
var easyNoodleRecipe = new Recipe("Hot Instant Noodle", 4, List("Boil water", "Cook noodle", "Serve"))
{% endcodeblock %}

Scala also allows us to define default values for some of these parameters so that when we create _Recipe_ objects that have these default values, we no longer need to specify the corresponding parameters when calling the constructor. Behind the scene, Scala simply generates additional constructors during compilation to support this.
For example, we specified a default value of 1 for `s` and a default empty list for `instructions` in the class definition. Therefore, we can create an _healthySaladRecipe_ that serves 1 by passing only the other two parameters. 

{% codeblock lang:scala %}
val healthySaladRecipe = new Recipe("Healthy green salad", instructions = List("Put greens in a bowl", "Add dressing", "Mix")) 
{% endcodeblock %}

Similarly, we can create a _lazyBurgerRecipe_ that serves one and has an empty instruction by passing only one parameter, its name.

{% codeblock lang:scala %}
var lazyBurgerRecipe = new Recipe("Big Mac Burger") 
{% endcodeblock %}

**Class parameters vs. Class fields:** there is an important difference about the scope of class parameters and var/val fields. Class parameters are not publicly visible unless they are defined with the keyword `var` or `val`. In such cases, a getter and/or a setter is generated correspondingly. In contrast, a class field is always defined with one of these keywords. Thus, Scala makes them accessible publicly. Here are some simple rules about the accessibility and visibility of class parameters and fields:

1. **Class parameters defined *without var/val* keyword:** can only be read within the class body definition. 
2. **Class parameters defined *with val*:** a getter is created and the parameter _can be read_ by a user of the class, by the object itself or by another instance of the same class. 
3. **Class parameters defined *with var*:** a getter and a setter are generated so the parameter _can be read and also reassigned_ by a user of the class, by the object itself or by another instance of the same class.

**Methods vs Functions** 
In Scala, methods and functions are defined in the same way using the syntax: 

{% codeblock lang:scala %}
def methodName(parameter1: Type1, Parameter2: Type2, ...): OutputType = {
	...
	result_expression
} 
{% endcodeblock %}

The difference between methods and functions in Scala is similar to that of instance methods and static methods in Java. In Scala, methods are invoked on an object instance with some parameters and are defined inside the class constructor. On the other hand, functions are not invoked on any object; they are simply called with some inputs and return an output. Furthermore, there are two ways to define functions in Scala. First, functions are usually defined inside a singleton object module. If that object is a companion of a class with the same name, then we can make the analogy with static methods in Java. Second, functions can alternatively be defined on the go using functions literals and assigned to a `val`. 

As previously mentionned, when a method is invoked in Scala with a single parameter, it can be interpreted as a binary operation. In fact, we can ommit the dot notation in that case when calling the method and replacing it with spaces between the object, the method name and the parameters. On the other hand, a parameterless method is usually written with a dot notation as if we access a field of the object on which we invoke the method. This programming style is known as _universal access principle_. 

Let's define some examples by defining two methods in the class Recipe's constructor. 

{% codeblock lang:scala %}
class Recipe(val n: String, val s: Int = 1, instructions: List[String] = Nil) {
  var description: String = ""
  var ingredients: Map[String, (Double, String)] = _

  // New methods 
  def hasNuts: Boolean = ingredients.keys.toList contains "nuts"

  def serve(p: Int): Map[String, (Double, String)] = this.ingredients mapValues (v => (v._1 * p, v._2))
}
{% endcodeblock %}

Now, we can test this in the Scala shell. Let's go back to our `healthySaladRecipe` and add its ingredients.

{% codeblock lang:scala %}
scala> healthySaladRecipe.ingredients = List("Tomatoes","Cucumber","Dressing","Feta cheese") zip List((1.0,"unit"), (1.0,"unit"), (1.0,"unit"),(50.0,"grams")) toMap
healthySaladRecipe.ingredients: Map[String,(Double, String)] = Map(Tomatoes -> (1.0,unit), Cucumber -> (1.0,unit), Dressing -> (1.0,unit), Feta cheese -> (50.0,grams))
{% endcodeblock %}

Remember that `healthySaladRecipe` was defined as a `val`. However, it had fields defined as `var` such as ingredients. That means that `healthySaladRecipe` has a mutable state. This is a key distinction that is important in Scala. Even though an object is bound to a val. It can still changes state even the class of that object has mutable state. Mutable state is definitely to be avoided if we follow the functional programming principles. Here, we simply use this as an example to show that there is a key difference between `var` vs. `var` and `mutability` vs. `immutability`. In the second part of this tutorial, we will get into more details on this matter. 

> A `val` that is bound to an object cannot be reassigned to another object. But that does not imply that the object does not have a mutable state! 

In the above code, we created a map of ingredients from a list of ingredients and a list of quantity. To do that, we invoked the method `zip` which is defined in `scala.collection.immutable.List` on the first list and passed the second list as parameter. This is an example of method invocation as binary operator in Scala. Then, we used a parameterless method `toMap` which is invoked on a list of pairs to create a Map. 

By default, any Recipe object will serve one person. Let's check this in the shell.

{% codeblock lang:scala %}
scala> healthySaladRecipe.serves
res: Int = 1
{% endcodeblock %}

So now, what if you invite two of your friends for lunch and you make a healthy salad for the three of you. Well, you can now invoke the `serve` method that we previously defined for that. What you will obtain is a new list of ingredients with the right quantity for each ingredient. 

{% codeblock lang:scala %}
scala> healthySaladRecipe.serve(3)
res: Map[String,(Double, String)] = Map(Tomatoes -> (3.0,unit), Cucumber -> (3.0,unit), Dressing -> (3.0,unit), Feta cheese -> (150.0,grams))
{% endcodeblock %}

Let's look closely at the implementation of the `serve` method. 

{% codeblock lang:scala %}
def serve(p: Int): Map[String, (Double, String)] = this.ingredients mapValues (v => (v._1 * p, v._2))
{% endcodeblock %}

We accessed the list of ingredients of `healthySaladRecipe` and invoked a method `mapValues` which is defined in `scala.collection.immutable.Map`. This method takes a function value as parameter and applies that function to each value of the element in the map ingredients. Above, we used a function literal, also called anonymous or lambda function, using scala's syntax: 

{% codeblock lang:scala %}
v => (v._1 * p, v._2)
{% endcodeblock %}

This function literal takes as input `v`, which has a type of a two-element tuple `(Double, String)`. Then, it returns of a value of the same type but with the ingredient quantity scaled by the number of serving `p`. We did not to specify the type signature of the function since Scala could infer to type of `v` from the type of `this.ingredients`.


We have reached the end of the Part 1 of this tour on "Scala flavors". In Part 2, we will revisit some Scala object-oriented features including _case classes_ and _companion objects_. We'll also get a taste of _first-class functions_ in Scala. Finally, we'll touch on immutability vs. mutability in Scala.


{% if page.comments %}
<div id="disqus_thread"></div>
<script type="text/javascript">
  /* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
  var disqus_shortname = 'Rico'; // required: replace example with your forum shortname

  /* * * DON'T EDIT BELOW THIS LINE * * */
  (function() {
      var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
      dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
      (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
  })();
</script>
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
<a href="http://disqus.com" class="dsq-brlink">comments powered by <span class="logo-disqus">Disqus</span></a>

{% endif%}