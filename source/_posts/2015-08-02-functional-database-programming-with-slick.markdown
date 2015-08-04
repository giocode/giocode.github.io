---
layout: post
title: "Slick Database Programming in Scala"
date: 2015-06-02 19:38:49 -0700
comments: false
categories: 
---


## Awesome are Scala Collections. 
Working with Scala collections is simply a joy and apparently effortless! Their interfaces are rich, consistent and nicely separated from implementation. Yet, we have many to choose from and each collection offers different characteristics in terms of performance, mutability and laziness. {% img right ./images/slick.png 260 260 %}Chances are you will find the right tool for the job. In addition, Scala collections work great with for-comprehensions. With Scala, not only do we have power but also we have a flexibility to express our computations: either in a Map-Reduce-Filter pattern or simply using a for expression. With Scala collections, we can always get rid of boilerplate and understand performance. 

###So is Slick!
With the Slick library, database programming would also enjoy such simplicity. Slick is a cool library that enable to query and access relational databases in plain Scala. Slick enables us to treat database tables almost as if they were Scala collections. In addition, it also gives us full control over when a database access happens and which data is transferred. In addition to its functional relational mapping features, Slick queries are type safe expressions. In other words, the compiler can catch inconsistencies at compile-time. Furthermore, we can compose simple queries into more complex ones and we can do so before even running them against the database. Apart from that, Slick also simplifies database managementt asks from creating schema and data models to managing connections and query transactions.

In this tutorial, I will highlight Slick's functional features and use the database of a Wine restaurant as an illustrative example. 

<!-- more -->


## Getting Started with Slick

Programming a Slick database application consists of the following steps: 

1. Configuring the database driver dependencies in SBT. 
2. Defining the database schema.
3. Creating Slick queries
4. Connecting to the database 
5. Run the queries or CRUD operations

Let's go through these steps in detail.

###Configuring Build Dependencies

We start by creating an SBT Scala project and configure the build settings in build.sbt:

```scala
name := "wine-slick-example"

version := "1.0"

scalaVersion := "2.11.6"

libraryDependencies += "org.scala-lang" % "scala-compiler" % scalaVersion.value

libraryDependencies ++= List(
  "com.typesafe.slick" %% "slick"           % "2.1.0",
  "com.h2database"      % "h2"              % "1.3.0",
  "ch.qos.logback"      % "logback-classic" % "1.1.2"
)
```

This file declares the library dependencies that a Slick project at least needs: the Slick library, the database driver library and a logging library. Here, we have chosen H2 database for simplicity. If you were to use MySQL or PostgreSQL, then the H2 dependency should be replaced by the corresponding JDBC driver. 

Once the SBT configurations are set, we can build our Slick application and start by importing the H2 Library: 

```

import scala.slick.driver.H2Driver
import scala.slick.driver.H2Driver.simple._

```

These help Slick to understand the functions provided by the database-specific driver (e.g. H2 driver here). It is convenient to import everything from `H2Driver.simple` to make sure all required implicit conversions and extension methods are in place when Slick needs them.

### Describing the Database Schema

After these simple configurations, we need define the database table and the type of each row in the table. In our example, we will only define one table of wines. Therefore, let's create a case class `Wine` and define its properties: 

```scala
// Message type 
final case class Wine(color: String, maker: String, name: String, year: Int, 
					  country: String, price: Double, id: Long = 0L){
    override def toString = f"$name $year ($country ~ $color)" 
}
```

Next, let's define the table `WineTable`:  

```scala
// Define a Table[Wine]
final class WineTable(tag: Tag) extends Table[Wine](tag, "wine") {
    def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
    def color = column[String]("color")
    def maker = column[String]("maker")
    def name = column[String]("name")
    def year = column[Int]("year")
    def country = column[String]("country")
    def price = column[Double]("price")
    
    def * = (color, maker, name, year, country, price, id) <> (Wine.tupled, Wine.unapply)
}
```

In the above code, we tell Slick three important things: 

- First, we tell Slick to define a table of wine by extending the abstract class `Table` with the type parameter `Wine`. By convention (since Slick 2.0), we also need to pass an extra tag argument with the table name. Note that the tag is however automatically passed by Slick and we do not need to specify it. 
- Second, we specify the type, name and any constraint for each column in the table by defining a get method for each property of `Wine`. For example, we add the constraint for `id` to be an auto-incrementing primary key.
- Finally, the projecion operator `*` is defined to map each row in `WineTable` to the Scala class `Wine` using `Wine`'s default `tupled` and `unapply` method.


###Creating queries 

In Slick, we can create queries even before connecting to the database. To start, let us create a `TableQuery` object `wines` that select all wines listed in our database. 


```scala
// Create a query to select the whole table 
lazy val wines = TableQuery[WineTable]
```

This is the starting point of every other query and it is a convention to define `wines` as a `lazy val`. At this time, we do not want to run the query yet. Now, we can define other queries using operators such as `filter`. For example, let's define queries for red wines or Canadian wines: 

```scala
// Another query for red wines
val redWines = wines.filter(_.color === "Red")

// Canadian wines
val canadianWines = wines.filter(_.country === "Canada")
```

**Warning:** 
Note the use of `===` instead of `==` for comparing two values for equality and `=!=` instead of `!=` for inequality. The other comparison operators are the same as in standard Scala code: `<`, `<=`, `>=`, `>`. If you accidentally use `==` instead of `===`, it will compile fine but your query results will be empty! 

For each `Query` or `TableQuery`, there exist handy methods called `selectStatement` that prints the SQL code that will be run. For instance, we can quickly check into SBT console for the above queries: 

```scala
scala> wines.selectStatement
res: String = select x2."color", x2."maker", x2."name", x2."year", x2."country", x2."price", x2."id" from "wine" x2

scala> canadianWines.selectStatement
res: String = select x2."color", x2."maker", x2."name", x2."year", x2."country", x2."price", x2."id" from "wine" x2 where x2."country" = 'Canada'
```

### Connecting to the database and running the queries 

Since we do not have yet a database, let's use the function `Database.forURL` and pass the database URL and the corresponding driver to create a `Database` object:

```scala
// Create a H2 in-memory database
def db = Database.forURL(
	url = "jdbc:h2:mem:chat-database;DB_CLOSE_DELAY=-1",
	driver = "org.h2.Driver"
)
```

Now, we can connect to the database by opening a new session with the method `withSession`. Within the sesssion, we are going to: 

- Create our wine table using `wines.ddl.create`. 
- Insert the list of wines we got before into the new table.
- Run the queries we previously created. 

All these must be done within the session. Moreover, the session argument must be set as implicit, otherwise we would have to specify it explicitly everytime we access and query the database. If we do not, we will run into an error that reads like this: __error: could not find implicit value for parameter session: scala.slick.jdbc.JdbcBackend#SessionDef__

```scala
db.withSession { implicit session =>

    // Create the wine table:
    println("Creating wine database")
    wines.ddl.create

    // Create and insert our selection of data:
    println("\nInserting wine data")
    wines ++= wineData

    // Run the test query and print the results:
    println("\nPrinting all wines:")
    wines.run.foreach(println)
    
    println("\nThese are the wines from Canada:")
    canadianWines.run.foreach(println)
}
```

Here what the results look like on standard output: 

```text
Printing all wines:

Malbec Reserva 2009 (Argentina ~ Red)
Cote de Sambre et Meusu Pinot Gris 2013 (France ~ White)
Whispering Angel Rosé 2014 (France ~ Rosé)
Cabernet Sauvignon 2005 (United States ~ Red)
Mcgregor Shiraz 2009 (South Africa ~ Red)
Marlborough Sauvignon Blanc 2014 (New Zealand ~ White)
Rosé Sec VQA Cave Spring 2010 (Canada ~ Rosé)
Paragon Edna Valley Riesling 2012 (United States ~ White)
Conca del Riu Anoia Barcelona de Nit 2011 (Spain ~ White)
Amarone della Valpolicella Riserva Classico Sergio 2009 (Italy ~ Red)
The Prisoner 2013 (United States ~ Red)
Ice wine Vidal Inniskillin 2012 (Canada ~ Red)
Vin de glace Cabernet 2013 (Canada ~ Red)
Pinot Grigio Rosé Folonari 2014 (Italy ~ Rosé)

These are the wines from Canada:

Rosé Sec VQA Cave Spring 2010 (Canada ~ Rosé)
Ice wine Vidal Inniskillin 2012 (Canada ~ Red)
Vin de glace Cabernet 2013 (Canada ~ Red)
```

## More CRUD wines with Slick

###Select statement

In our previous queries, we got back a sequence of `Wine`. What if all we need is the list of French winemakers? For such kind of SELECT queries, we use the `map` operator and pass a function to select whatever we can access from the table. 

```scala
// List of French winemakers 
val frenchWinemakers = wines.filter(_.country === "France").
                             map(_.maker)

// Corresponding SQL statement
// select x2."maker" from "wine" x2 where x2."country" = 'France'

// Run within session
frenchWinemakers.run.foreach(println)
Château Bon-Baron
Chateau d'Esclans
```

In the above session, we assumed that we run the queries within `db.withSession{}`. If you want to run queries in the console, you can either use ``db.withSession{anotherQuery.run}` everytime or create an implicit session as follows: 

```scala
implicit val session = db.createSession
```
Note that this is only useful for experimentation purpose within Scala REPL. It is recommended not to create session manually. Instead, use `db.withSession` so that the resources are managed by Slick automatically.

###For Comprehensions When You Need it
As I mentionned in the beginning, working with Slick tables are very similar to handling Scala collections. In addition to the combinators such as `map` and `filter`, we can also rely on _for comprehensions` to express queries. A valid alternative to getting the french makers is: 

```scala
val frenchWinemakers = 
	for {
		w <- wines
		if w.country === "France"
	} yield w.maker
}
```

Which one should you use totally depends on your taste and needs. 

###Inserting rows

Imagine our restaurant just received a nice selection of Australina wines. We can ask Slick to insert them into the database using `insert`: 

```scala
    val aussies = Seq(
        Wine("Red", "Chandon", "Chandon Australia Heathcote Shiraz", 2001, "Australia", 45.0 ),
        Wine("Rosé", "Jacob's Creek", "Sparkling Moscato Rosé", 2014, "Australia", 24.0),
        Wine("Red", "Chandon", "Vintage Collection Cuvée 500", 2008, "Australia", 59.95),
        Wine("Red", "Cranswick", "Riverina Cabernet Merlot", 1999, "Australia", 24.0)
    )
    
    db.withSession { implicit session => 
        // Insert Australian selection into wines table
        wines ++= aussies
        
        // Insert one wine 
        wines += Wine("Red", "Castle Glen Australia", "Montepulicano", 2002, "Australia", 39)
        
        // Showing all australian wines 
        wines.filter(_.country === "Australia").run.foreach(println)
    }
```

Here is the output:

```text
Chandon Australia Heathcote Shiraz 2001 (Australia ~ Red)
Sparkling Moscato Rosé 2014 (Australia ~ Rosé)
Vintage Collection Cuvée 500 2008 (Australia ~ Red)
Riverina Cabernet Merlot 1999 (Australia ~ Red)
Montepulicano 2002 (Australia ~ Red)
```

###Updating rows

Our restaurant is making a summer holiday promotion to Italian wine bottles. We are asked to update the price with 10% discount. Updating is done with a succession of `map` and `update` calls. Here, we first need to select the price to be updated and call `update` to update them. Unfortunately, it is not yet possible to update with scalar expression or transformations of the price value. 

```scala
// Discount all french wine bottles between $20 and $40 to $20
val frenchWines = wines.filter(_.country === "France")
db.withSession { implicit session => 
    
   	val selectedPrices = 
        for {
            w <- wines
            if (w.country === "France")
            if (w.price >= 20.0 && w.price <= 40.0)
        } yield w.price
    selectedPrices.update(20.0)      
}
```

Here are the French wines before discount:

```text
Cote de Sambre et Meusu Pinot Gris 2013 (France ~ White ~ 27.0)
Whispering Angel Rosé 2014 (France ~ Rosé ~ 35.0)
```

And after the discount: 

```text
Cote de Sambre et Meusu Pinot Gris 2013 (France ~ White ~ 20.0)
Whispering Angel Rosé 2014 (France ~ Rosé ~ 20.0)
```

###Sorting and limiting results 

Slick also provide mechanisms for ordering and limiting the results of queries. These are the standard collection methods `sortBy`, `take` and `drop`. For instance, let us query the 3 least expensive wines in our selection: 

```scala
// 3 least expensive 
val cheapWines = wines.sortBy(_.price).take(3)

// Wines by country sorted by price
val winesByCountry = wines.sortBy(w => (w.country, w.price))
db.withSession { implicit session => 

    println("\n>> Our cheapest wines:\n")
    cheapWines.run.foreach(println)
    
    println("\n>> Wines by Country:\n")
    winesByCountry.run.foreach(println)
}
```

Here are the results: 

```text
>> Our cheapest wines:

Pinot Grigio Rosé Folonari 2014 (Italy ~ Rosé ~ 11.9)
Mcgregor Shiraz 2009 (South Africa ~ Red ~ 14.0)
Rosé Sec VQA Cave Spring 2010 (Canada ~ Rosé ~ 14.95)

>> Wines by Country sorted by country and region:

Malbec Reserva 2009 (Argentina ~ Red ~ 38.0)
Sparkling Moscato Rosé 2014 (Australia ~ Rosé ~ 24.0)
Riverina Cabernet Merlot 1999 (Australia ~ Red ~ 24.0)
Montepulicano 2002 (Australia ~ Red ~ 39.0)
Chandon Australia Heathcote Shiraz 2001 (Australia ~ Red ~ 45.0)
Vintage Collection Cuvée 500 2008 (Australia ~ Red ~ 59.95)
Rosé Sec VQA Cave Spring 2010 (Canada ~ Rosé ~ 14.95)
Vin de glace Cabernet 2013 (Canada ~ Red ~ 39.95)
Ice wine Vidal Inniskillin 2012 (Canada ~ Red ~ 49.95)
Cote de Sambre et Meusu Pinot Gris 2013 (France ~ White ~ 20.0)
Whispering Angel Rosé 2014 (France ~ Rosé ~ 20.0)
Pinot Grigio Rosé Folonari 2014 (Italy ~ Rosé ~ 11.9)
Amarone della Valpolicella Riserva Classico Sergio 2009 (Italy ~ Red ~ 50.0)
Marlborough Sauvignon Blanc 2014 (New Zealand ~ White ~ 23.0)
Mcgregor Shiraz 2009 (South Africa ~ Red ~ 14.0)
Conca del Riu Anoia Barcelona de Nit 2011 (Spain ~ White ~ 40.0)
Paragon Edna Valley Riesling 2012 (United States ~ White ~ 26.0)
Cabernet Sauvignon 2005 (United States ~ Red ~ 28.0)
The Prisoner 2013 (United States ~ Red ~ 51.0)
```


###Deleting rows

Deleting rows are simply done through a sequence of `filter` and `delete`. For example, if we decide not to sell Italian wines anymore, we could use: 

```scala
wines.filter(_.country === "Italy").delete”
```
Delete returns the number of deleted rows.

###Transactions and rollback

When we run queries with `db.withSession`, each query runs independently. Hence, it is possible for some queries to faile while for some others to succeed. In many applications, we often want to combine queries within a transaction so that either all queries succeed or all fail. This is done simply by wrapping the queries within a `session.withTransaction` block inside `db.withSession`. For instance, we can perform a batch of price updates together as follows: 

```scala
// Select the prices between lo and hi
def discount(lo: Double, hi: Double) =
  wines.filter(w => w.price >= lo && w.price >= hi).map(_.price)

db.withSession { implicit session =>
  session.withTransaction {
    discount(20.0, 30.0).update(20.0)
    discount(30.0, 35,0).update(30.0)
    discount(40.0, 50.0).update(50.0)
  }
  wines.run
}
```

The block passed to `withTransaction` is executed as a single transaction. If an exception is thrown, Slick rolls back the transaction at the end of the block. Note that Slick only rolls back the database operations. The remaining Scala code are executed and their effects are not rolled back.

### Summary 

We are going to stop here for now. But, there are still a lot to cover about this cool library. To summarize, handling database access and queries is very smooth and effortless with Slick. Using a lifted embedding approach, we can define queries and run them later after we connect to the database. Slick database operations are very similar to Scala collection methods. We select rows and columns with `filter` and `map`. Then, we can order and limit our results with `sortBy`, `take` and `drop`. Updating, deleting and inserting rows are also a breeze. In addition, we can use for expression to write complex queries or compose simple ones. In my next post, I hope to explore more about Slick's type system, other operations such as joins and aggregates as well as the new reactive streams support in Slick 3.0. I hope you could appreciate the functional aspects of database programming in Slick and use some of these in your applications. 