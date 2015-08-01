---
layout: page
title: "Sabermetrics with Apache Spark"
date: May 24, 2015 
comments: true
sharing: true
footer: true
---


It is no secret analytics have made a big impact in sports. The quest for an objective understanding of the game has even a name: "sabermetrics". Analytics have proven invaluable in many aspects: from [building dream teams under tight cap constraints](http://www.sfgate.com/warriors/article/Golden-State-Warriors-at-the-forefront-of-NBA-5753776.php), to [selecting game-specific strategies](https://en.wikipedia.org/wiki/Moneyball) to actively [engaging with fans](http://www.nhl.com/ice/news.htm?id=754184) and so on. No matter what the goal is, gaining useful insights requires a quick ability to process big data and handle its complexities. In this following, we analyze NCAA Men's college basketball game stats gathered during one season. As sports-data experts, we are going to leverage Spark's graph processing library to answer several questions for retrospection.

[__Apache Spark__](http://spark.apache.org/) is a fast and general-purpose technology, which greatly simplify the parallel processing of large data that is distributed over a computing cluster. While Spark handles different types of processing, here we will focus on its graph processing capability. In particular, our goal is to expose the powerful yet generic graph-aggregation operator of Spark:`aggregateMessages`. We can think of this operator as a version of MapReduce for aggregating the neighborhood information in graphs.

In fact, many graph processing algorithms such as PageRank rely on iteratively accessing the properties of neighbouring vertices and adjacent edges. By applying `aggregateMessages` on the NCAA College Basketball datasets, we will: 

- identify the basic mechanisms and undersand the patterns for using `aggregateMessages`
- apply `aggregateMessages` to create custom graph aggregation operations  
- optimize the performance and efficiency of `aggregateMessages`

## NCAA College Basketball datasets

As an illustrative example, the [NCAA College Basketball](http://www.ncaa.com/sports/basketball-men) datasets consist of two CSV datasets. This first one `teams.csv` contains the list of all college teams that played in NCAA Division I competition. Each team is associated with a 4 digit id number. The second dataset `stats.csv` contains the score and statistics of every game played during the 2014-2015 regular season. 

### Loading team data into RDDs

To start, we parse and load these datasets into RDDs (__Resilient Distributed Datasets__), which are the core Spark abstraction for any data that is distributed and stored over a cluster. First, we create a class `GameStats` that records a team's statistics during a game.

```scala
case class GameStats(
    val score: Int,
    val fieldGoalMade:   Int,
    val fieldGoalAttempt: Int, 
    val threePointerMade: Int,
    val threePointerAttempt: Int,
    val threeThrowsMade: Int,
    val threeThrowsAttempt: Int, 
    val offensiveRebound: Int,
    val defensiveRebound: Int,
    val assist: Int,
    val turnOver: Int,
    val steal: Int,
    val block: Int,
    val personalFoul: Int
)
```

### Loading game stats into RDDs

We also add the following methods to `GameStats` in order to know how efficient a team's offense was:  
```scala
// Field Goal percentage
def fgPercent: Double = 100.0 * fieldGoalMade / fieldGoalAttempt

// Three Point percentage
def tpPercent: Double = 100.0 * threePointerMade / threePointerAttempt

// Free throws percentage
def ftPercent: Double = 100.0 * threeThrowsMade / threeThrowsAttempt
override def toString: String = "Score: " + score 
```

Next, we create a couple of classes for the games' result. 
```scala
abstract class GameResult(
    val season:     Int, 
    val day:        Int,
    val loc:        String
)


case class FullResult(
    override val season:    Int, 
    override val day:       Int,
    override val loc:       String, 
    val winnerStats:        GameStats,
    val loserStats:         GameStats 
) extends GameResult(season, day, loc)
```

`FullResult` has the year and day of the season, the location where the game was played and the game statistics of both the winning and losing teams. 

Next, we will create a statistics graph of the regular seasons. In this graph, the nodes are the teams whereas each edge corresponds to a specific game. To create the graph, let us parse the CSV file `teams.csv` into the RDD `teams` : 

```scala
val teams: RDD[(VertexId, String)] =
    sc.textFile("./data/teams.csv").
    filter(! _.startsWith("#")).
    map {line =>
        val row = line split ','
        (row(0).toInt, row(1))
    }
```

We can check the first few teams in this new RDD:
```scala
scala> teams.take(3).foreach{println}
(1101,Abilene Chr)
(1102,Air Force)
(1103,Akron)
```

We do the same thing to obtain an RDD of the game results, which will have a type `RDD[Edge[FullResult]]`. We just parse `stats.csv` and record the fields that we need: (1) the ID of the winning team, (2) the ID of the losing team, and (3) the game statistics of both teams:

```scala
val detailedStats: RDD[Edge[FullResult]] =
    sc.textFile("./data/stats.csv").
    filter(! _.startsWith("#")).
    map {line =>
        val row = line split ','
        Edge(row(2).toInt, row(4).toInt, 
            FullResult(
                row(0).toInt, row(1).toInt, 
                row(6),
                GameStats(      
                                score = row(3).toInt,
                        fieldGoalMade = row(8).toInt,
                     fieldGoalAttempt = row(9).toInt, 
                     threePointerMade = row(10).toInt,
                  threePointerAttempt = row(11).toInt,   
                      threeThrowsMade = row(12).toInt,
                   threeThrowsAttempt = row(13).toInt, 
                     offensiveRebound = row(14).toInt,
                     defensiveRebound = row(15).toInt,
                               assist = row(16).toInt,
                             turnOver = row(17).toInt,
                                steal = row(18).toInt,
                                block = row(19).toInt,
                         personalFoul = row(20).toInt
                ),
                GameStats(
                                score = row(5).toInt,
                        fieldGoalMade = row(21).toInt,
                     fieldGoalAttempt = row(22).toInt, 
                     threePointerMade = row(23).toInt,
                  threePointerAttempt = row(24).toInt,
                      threeThrowsMade = row(25).toInt,
                   threeThrowsAttempt = row(26).toInt, 
                     offensiveRebound = row(27).toInt,
                     defensiveRebound = row(28).toInt,
                               assist = row(20).toInt,
                             turnOver = row(30).toInt,
                                steal = row(31).toInt,
                                block = row(32).toInt,
                         personalFoul = row(33).toInt
                )
            )
        )
    }
```

We could avoid typing all that by using the nice [spark-csv package](http://spark-packages.org/package/databricks/spark-csv) that reads CSV files into `SchemaRDD`. Let's check what we got:

```scala
scala> detailedStats.take(3).foreach(println)
Edge(1165,1384,FullResult(2006,8,N,Score: 75-54))
Edge(1393,1126,FullResult(2006,8,H,Score: 68-37))
Edge(1107,1324,FullResult(2006,9,N,Score: 90-73))
```

We then create our score graph using the collection of teams (of type `RDD[(VertexId, String)]`) as vertices and the collection `detailedStats` (of type `RDD[(VertexId, String)]`) as edges:
```scala
scala> val scoreGraph = Graph(teams, detailedStats)
``` 

For curiosity, let's see which team has won against the 2015 NCAA national champ Duke during regular season. It seems Duke has lost only four games during the regular season:
```scala
scala> scoreGraph.triplets.filter(_.dstAttr == "Duke").foreach(println)
((1274,Miami FL),(1181,Duke),FullResult(2015,71,A,Score: 90-74))
((1301,NC State),(1181,Duke),FullResult(2015,69,H,Score: 87-75))
((1323,Notre Dame),(1181,Duke),FullResult(2015,86,H,Score: 77-73))
((1323,Notre Dame),(1181,Duke),FullResult(2015,130,N,Score: 74-64))
```


## Aggregating Game Stats

Having our graph ready, let's start aggregating the stats data in `scoreGraph`. In Spark, `aggregateMessages` is the operator for that kind of jobs. 
For example, let's find out the average field goals made per game by the winners. In other words, the games that a team lost will not be counted. To get the average for each team, we first need to have the number of games won by the team and the total field goals that the team made in those games:

```scala
// Aggregate the total field goals made by winning teams
type Msg = (Int, Int)
type Context = EdgeContext[String, FullResult, Msg] 
val winningFieldGoalMade: VertexRDD[Msg] = scoreGraph aggregateMessages(
    // sendMsg
    (ec: Context) => ec.sendToSrc(1, ec.attr.winnerStats.fieldGoalMade),
    
    // mergeMsg
    (x: Msg, y: Msg) => (x._1 + y._1, x._2+ y._2)
)
```

### AggregateMessage operator 

There is a lot going on in the above call to `aggregateMessages`. So let's see it working in slow motion. When we called `aggregateMessages` on the `scoreGraph`, we had to pass two functions as arguments.

#### SendMsg
The first function has a signature `EdgeContext[VD, ED, Msg] => Unit`. It takes an `EdgeContext` as input. Since it does not return anything, its return type is `Unit`. This function is needed for sending message between nodes. 

Ok, but what is that `EdgeContext` type? `EdgeContext` represents an edge along with its neighboring nodes. It can access both the edge attribute and the source and destination nodes' attributes. In addition, `EdgeContext` has two methods to send messages along the edge to its source node or to its destination node. These methods are `sendToSrc` and `sendToDst` respectively. Then, the type of messages being sent through the graph is defined by `Msg`. Similar to vertex and edge types, we can defined the concrete type that `Msg` takes as we wish. 

#### Merge
In addition to `sendMsg`, the second function that we need to pass to `aggregateMessages` is a `mergeMsg` function with the following signature `(Msg, Msg) => Msg`. As its name implies, `mergeMsg` is used to merge two messages received at each node into a new one. Its output must also be of type `Msg`. Using these two functions, `aggregateMessages` returns the aggregated messages inside a `VertexRDD[Msg]`. 

#### Example
In our example, we need to aggregate the number of games played and the number of field goals made. Therefore, `Msg` is simply a pair of `Int`. Furthermore, each edge context needs to send a message to only its source node, i.e. the winning team. This is because we want to compute the total field goals made by each team for only the games that it won. The actual message sent to each "winner" node is the pair of integers `(1, ec.attr.winnerStats.fieldGoalMade)`. Here, `1` serves as a counter for the number of games won by the source node. The second integer, which is the number of field goals in one game, is extracted from the edge attribute.

As we set out to compute the average field goals per winning game for all teams, we need to apply the `mapValues` operator to the output of `aggregateMessages` as follows:
```scala
// Average field goals made per Game by the winning teams
val avgWinningFieldGoalMade: VertexRDD[Double] = 
    winningFieldGoalMade mapValues (
        (id: VertexId, x: Msg) => x match {
            case (count: Int, total: Int) => total.toDouble/count
})

scala> avgWinningFieldGoalMade.take(5).foreach(println)
(1260,24.71641791044776) 
(1410,23.56578947368421)
(1426,26.239436619718308)
(1166,26.137614678899084)
(1434,25.34285714285714)
```
   
## Abstracting out the aggregation

That was kind of cool! We can surely do the same thing for the average points per game scored by winning teams:

```scala
// Aggregate the points scored by winning teams
val winnerTotalPoints: VertexRDD[(Int, Int)] = scoreGraph.aggregateMessages(
    // sendMsg
    triplet => triplet.sendToSrc(1, triplet.attr.winnerStats.score), 
    // mergeMsg
    (x, y) => (x._1 + y._1, x._2+ y._2)
)

// Average field goals made per Game by winning teams 
var winnersPPG: VertexRDD[Double] = 
            winnerTotalPoints mapValues (
                (id: VertexId, x: (Int, Int)) => x match {
                    case (count: Int, total: Int) => total.toDouble/count
                })
        
scala> winnersPPG.take(5).foreach(println)
(1260,71.19402985074628)
(1410,71.11842105263158)
(1426,76.30281690140845)
(1166,76.89449541284404)
(1434,74.28571428571429)
```

What if the coach wants to know the top 5 teams with the highest average three pointer made per winning game? By the way, he might also ask which teams are the most efficient in three pointers. 

### Keeping things DRY

We can copy and modify the previous code but that would be quite repetitive. Instead, let's abstract out the average aggregation operator so that it can work on any statistics that the coach needs. Luckily, Scala's higher-order functions are there to help in this task. 

Let's define functions that take a team's `GameStats` as input and returns specific statistic that we are interested in. For now, we will need the number of three pointer made and the average three pointer percentage: 
```scala
// Getting individual stats
def threePointMade(stats: GameStats) = stats.threePointerMade
def threePointPercent(stats: GameStats) = stats.tpPercent
```

Then, we create a generic function that takes as inputs a stats graph and one of the functions defined above, which has a signature `GameStats => Double`. 

```scala
// Generic function for stats averaging
def averageWinnerStat(graph: Graph[String, FullResult])(getStat: GameStats => Double): VertexRDD[Double] = {
    type Msg = (Int, Double)
    val winningScore: VertexRDD[Msg] = graph.aggregateMessages[Msg](
        // sendMsg
        triplet => triplet.sendToSrc(1, getStat(triplet.attr.winnerStats)), 
        // mergeMsg
        (x, y) => (x._1 + y._1, x._2+ y._2)
    )
    winningScore mapValues (
        (id: VertexId, x: Msg) => x match {
            case (count: Int, total: Double) => total/count
        })
}
```        

Now, we can get the average stats by passing the functions `threePointMade` and `threePointPercent` to `averageWinnerStat`:
```scala
val winnersThreePointMade = averageWinnerStat(scoreGraph)(threePointMade) 
val winnersThreePointPercent = averageWinnerStat(scoreGraph)(threePointPercent) 
```

With little efforts, we can tell coach which five winning teams score the highest number of threes per game:
```scala
scala> winnersThreePointMade.sortBy(_._2,false).take(5).foreach(println)
(1440,11.274336283185841)
(1125,9.521929824561404)
(1407,9.008849557522124)
(1172,8.967441860465117)
(1248,8.915384615384616)
```
While we are at it, let's find out the five most efficient teams in three pointers:
```scala
scala> winnersThreePointPercent.sortBy(_._2,false).take(5).foreach(println)
(1101,46.90555728464225)
(1147,44.224282479431224)
(1294,43.754532434101534)
(1339,43.52308905887638)
(1176,43.080814169045105)
```
Interestingly, the teams that made the most three pointers per winning game are not always the one who are the most efficient at it. But it is ok because at least, they won those games. 

### Coach wants more numbers

Coach seems to argue against that argument. He asks us to get the same statistics but he wants the average over all games that each team has played. 

We then have to aggregate the information at all nodes, and not only at the destination nodes. To make our previous abstraction more flexible, let's create the following types:
```scala
trait Teams
case class Winners extends Teams 
case class Losers extends Teams
case class AllTeams extends Teams
```

We modify the previous higher-order function to have a extra argument `Teams`, which will help us specify at which nodes we want to collect and aggregate the required game stats. The new function becomes: 
 
```scala
def averageStat(graph: Graph[String, FullResult])(getStat: GameStats => Double, tms: Teams): VertexRDD[Double] = {
    type Msg = (Int, Double)
    val aggrStats: VertexRDD[Msg] = graph.aggregateMessages[Msg](
        // sendMsg
        tms match {
            case _ : Winners => t => t.sendToSrc((1, getStat(t.attr.winnerStats)))
            case _ : Losers  => t => t.sendToDst((1, getStat(t.attr.loserStats)))
            case _       => t => {
                t.sendToSrc((1, getStat(t.attr.winnerStats)))
                t.sendToDst((1, getStat(t.attr.loserStats)))
            }
        }
        , 
        // mergeMsg
        (x, y) => (x._1 + y._1, x._2+ y._2)
    )

    aggrStats mapValues (
        (id: VertexId, x: Msg) => x match {
            case (count: Int, total: Double) => total/count
            })
    }
```

Now, `aggregateStat` allows us to choose whether we want to aggregate the stats for winners only, for losers only or for all teams. Since coach wants the overall stats averaged over all games played, we aggregate the stats by passing the `AllTeams()` flag in `aggregateStat`. In this case, we define the `sendMsg` argument in `aggregateMessages` to send the required stats to both the source (the winner) and to the destination (the loser) using `EdgeContext`'s `sendToSrc` and `sendToDst` functions respectively. This mechanism is pretty straightforward. We just need make sure we send the right information to the right node. In this case, we send the `winnerStats` to the winner and the `loserStats`to the loser. 

Ok, you got the idea now. So, let's apply it to please our coach. Here are the teams with the overall highest three pointers per page: 
```scala
// Average Three Point Made Per Game for All Teams 
val allThreePointMade = averageStat(scoreGraph)(threePointMade, AllTeams())   
scala> allThreePointMade.sortBy(_._2, false).take(5).foreach(println)
(1440,10.180811808118081)
(1125,9.098412698412698)
(1172,8.575657894736842)
(1184,8.428571428571429)
(1407,8.411149825783973) 
```

And here are the five most efficient teams overall in three pointers per Game:
```scala
// Average Three Point Percent for All Teams
val allThreePointPercent = averageStat(scoreGraph)(threePointPercent, AllTeams())

scala> allThreePointPercent.sortBy(_._2,false).take(5).foreach(println)
(1429,38.8351815824302)
(1323,38.522819895594)
(1181,38.43052051444854)
(1294,38.41227053353959)
(1101,38.097896464168954)
```

Actually, there is only a 2% difference between the most efficient team and the one in 50th position. Most NCAA teams are therefore pretty efficient behind the line. I bet coach knew that already!

### Average Points Per Game

We can also reuse the `averageStat` function to get the average points per game for the winners. In particular, let's take a look at the two teams that won games with the highest and lowest scores:

```
// Winning teams
val winnerAvgPPG = averageStat(scoreGraph)(score, Winners())

scala> winnerAvgPPG.max()(Ordering.by(_._2))
res36: (org.apache.spark.graphx.VertexId, Double) = (1322,90.73333333333333)

scala> winnerAvgPPG.min()(Ordering.by(_._2))
res39: (org.apache.spark.graphx.VertexId, Double) = (1197,60.5)
```
Apparently, the most defensive team can win game by scoring only 60 points whereas the most offensive team can score an average of 90 points. 

Next, let us average the points per game for all games played and look at the two teams with the best and worst offense during the 2015 season:
```scala
// Average Points Per Game of All Teams
val allAvgPPG = averageStat(scoreGraph)(score, AllTeams())

scala> allAvgPPG.max()(Ordering.by(_._2))
res42: (org.apache.spark.graphx.VertexId, Double) = (1322,83.81481481481481)

scala> allAvgPPG.min()(Ordering.by(_._2))
res43: (org.apache.spark.graphx.VertexId, Double) = (1212,51.111111111111114)

```

To no surprise, the best offensive team is the same as the one who scores most in winning games. To win games, 50 points is not enough in average for a team to win games.

## Defense Stats: The D matters as in Direction

Above, we obtained some statistics such as field goals or three point percentage that a team achieves. What if we want to aggregate instead the average points or rebounds that each team concedes to their opponents? To compute that, we define a new higher-order function `averageConcededStat`. Compared to `averageStat`, this function needs to send the `loserStats` to the winning team and the `winnerStats` to the losing team. To make things more interesting, we are going to make the team name as part of the message `Msg`.

```scala
def averageConcededStat(graph: Graph[String, FullResult])(getStat: GameStats => Double, rxs: Teams): VertexRDD[(String, Double)] = {
    type Msg = (Int, Double, String)
    val aggrStats: VertexRDD[Msg] = graph.aggregateMessages[Msg](
        // sendMsg
        rxs match {
            case _ : Winners => t => t.sendToSrc((1, getStat(t.attr.loserStats), t.srcAttr))
            case _ : Losers  => t => t.sendToDst((1, getStat(t.attr.winnerStats), t.dstAttr))
            case _       => t => {
                t.sendToSrc((1, getStat(t.attr.loserStats),t.srcAttr))
                t.sendToDst((1, getStat(t.attr.winnerStats),t.dstAttr))
            }
        }
        , 
        // mergeMsg
        (x, y) => (x._1 + y._1, x._2+ y._2, x._3)
    )

    aggrStats mapValues (
        (id: VertexId, x: Msg) => x match {
            case (count: Int, total: Double, name: String) => (name, total/count)
        })
}

```
With that, we can calculate the average points conceded by winning and losing teams as follows:
```scala
val winnersAvgConcededPoints = averageConcededStat(scoreGraph)(score, Winners())
val losersAvgConcededPoints = averageConcededStat(scoreGraph)(score, Losers())

scala> losersAvgConcededPoints.min()(Ordering.by(_._2))
res: (VertexId, (String, Double)) = (1101,(Abilene Chr,74.04761904761905))

scala> winnersAvgConcededPoints.min()(Ordering.by(_._2))
res: (org.apache.spark.graphx.VertexId, (String, Double)) = (1101,(Abilene Chr,74.04761904761905))

scala> losersAvgConcededPoints.max()(Ordering.by(_._2))
res: (VertexId, (String, Double)) = (1464,(Youngstown St,78.85714285714286))

scala> winnersAvgConcededPoints.max()(Ordering.by(_._2))
res: (VertexId, (String, Double)) = (1464,(Youngstown St,71.125))
```
The above tells us that Abilene Christian University is the most defensive team. They concede the least points whether they win a game or not. On the other hand, Youngstown has the worst defense. 


## Joining Aggregated Stats into Graphs 

The previous example shows us how flexible the `aggregateMessages` operator is. We can define the type `Msg` of the messages to be aggregated to fit our needs. Moreover, we can select which nodes receive the messages. Finally, we can also define how we want to merge the messages. 

As a final example, let us aggregate many statistics about each team and join this information into the nodes of the graph. To start, we create its own class for the team stats: 

```scala
// Average Stats of All Teams 
case class TeamStat(
        wins: Int  = 0      // Number of wins
     ,losses: Int  = 0      // Number of losses
        ,ppg: Int  = 0      // Points per game
        ,pcg: Int  = 0      // Points conceded per game
        ,fgp: Double  = 0   // Field goal percentage
        ,tpp: Double  = 0   // Three point percentage
        ,ftp: Double  = 0   // Free Throw percentage
     ){
    override def toString = wins + "-" + losses
}
```

Then, we collect the average stats for all teams using `aggregateMessages` below. For that, we define the type of the message to be an 8-element tuple that holds the counter for games played, wins, losses and other statistics that will be stored in `TeamStat` as listed above.
```scala
type Msg = (Int, Int, Int, Int, Int, Double, Double, Double)

val aggrStats: VertexRDD[Msg] = scoreGraph.aggregateMessages(
        // sendMsg
        t => {
                t.sendToSrc((   1,
                                1, 0, 
                                t.attr.winnerStats.score, 
                                t.attr.loserStats.score,
                                t.attr.winnerStats.fgPercent,
                                t.attr.winnerStats.tpPercent,
                                t.attr.winnerStats.ftPercent
                           ))
                t.sendToDst((   1,
                                0, 1, 
                                t.attr.loserStats.score, 
                                t.attr.winnerStats.score,
                                t.attr.loserStats.fgPercent,
                                t.attr.loserStats.tpPercent,
                                t.attr.loserStats.ftPercent
                           ))
             }
        , 
        // mergeMsg
        (x, y) => ( x._1 + y._1, x._2 + y._2, 
                    x._3 + y._3, x._4 + y._4,
                    x._5 + y._5, x._6 + y._6,
                    x._7 + y._7, x._8 + y._8
                )
    )
```

Given the aggregate message `aggrStats`, we map them into a collection of `TeamStat`
```scala
val teamStats: VertexRDD[TeamStat] = aggrStats mapValues {
        (id: VertexId, m: Msg) => m match {
            case ( count: Int, 
                    wins: Int, 
                  losses: Int,
                  totPts: Int, 
              totConcPts: Int, 
                   totFG: Double,
                   totTP: Double, 
                   totFT: Double)  => TeamStat( wins, losses,
                                                totPts/count,
                                                totConcPts/count,
                                                totFG/count,
                                                totTP/count,
                                                totFT/count)

        }
}
```

Next, let us join the `teamStats` into the graph. For that, we first create a class `Team` as a new type for the vertex attribute. `Team` will have the name and the `TeamStat`: 
```scala
case class Team(name: String, stats: Option[TeamStat]) {
    override def toString = name + ": " + stats
}
```

Next, we use the `joinVertices` operator that we have seen in the previous chapter:
```scala
// Joining the average stats to vertex attributes
def addTeamStat(id: VertexId, t: Team, stats: TeamStat) = Team(t.name, Some(stats))

val statsGraph: Graph[Team, FullResult] = 
    scoreGraph.mapVertices((_, name) => Team(name, None)).
               joinVertices(teamStats)(addTeamStat)
```

We can see that the join has worked well by printing the first three vertices in the new graph `statsGraph`:
```scala
scala> statsGraph.vertices.take(3).foreach(println)
(1260,Loyola-Chicago: Some(17-13))
(1410,TX Pan American: Some(7-21))
(1426,UT Arlington: Some(15-15))
```

To conclude this task, let us find out the top 10 teams in the regular seasons. To do so, we define an `Ordering` for `Option[TeamStat]` as follows:
```scala
import scala.math.Ordering 
object winsOrdering extends Ordering[Option[TeamStat]] {
    def compare(x: Option[TeamStat], y: Option[TeamStat]) = (x, y) match {
        case (None, None)       => 0 
        case (Some(a), None)    => 1
        case (None, Some(b))    => -1
        case (Some(a), Some(b)) => if (a.wins == b.wins) a.losses compare b.losses
                                   else a.wins compare b.wins
    }
}
```

Finally!
```scala
import scala.reflect.classTag
import scala.reflect.ClassTag
scala> statsGraph.vertices.sortBy(v => v._2.stats,false)(winsOrdering, classTag[Option[TeamStat]]).
     |                             take(10).foreach(println)
(1246,Kentucky: Some(34-0))
(1437,Villanova: Some(32-2))
(1112,Arizona: Some(31-3))
(1458,Wisconsin: Some(31-3))
(1211,Gonzaga: Some(31-2))
(1320,Northern Iowa: Some(30-3))
(1323,Notre Dame: Some(29-5))
(1181,Duke: Some(29-4))
(1438,Virginia: Some(29-3))
(1268,Maryland: Some(27-6))
```

Note that the `ClassTag` parameter is required in `sortBy` to make use of Scala's reflection. That is why we had the above imports. 


## Performance optimization with TripletFields


In addition to the `sendMsg` and `mergeMsg`, `aggregateMessages` can also take an optional argument `tripletsFields` which indicates what data is accessed in the EdgeContext. The main reason for explicitly specifying such information is to help optimize the performance of the `aggregateMessages` operation. 

In fact, `TripletFields` represents a subset of the fields of an _EdgeTriplet_ and it enables GraphX to populate only those fields when necessary.

The default value is TripletFields.All which means that the `sendMsg` function may access any of the fields in the EdgeContext. Otherwise, the tripletFields argument is used to tell GraphX that only part of the EdgeContext will be required so that an efficient join strategy can be used. All possible options for the tripletsFields are listed below:

- `TripletFields.All`: expose all the fields (source, edge, and destination).
- `TripletFields.Dst`: expose the destination and edge fields but not the source field.
- `TripletFields.EdgeOnly`: expose only the edge field.
- `TripletFields.None`: None of the triplet fields are exposed.
- `TripletFields.Src`: expose the source and edge fields but not the destination field.

Using our previous example, if we are interested in computing the total number of wins and losses for each team, we will not need to access any field of the `EdgeContext`. In this case, we should use `TripletFields.None` to indicate so:

```scala
// Number of wins of the teams
val numWins: VertexRDD[Int] = scoreGraph.aggregateMessages(
    triplet => {
        triplet.sendToSrc(1)         // No attribute is passed but an integer
    },
    (x, y) => x + y,
    TripletFields.None
)

// Number of losses of the teams
val numLosses: VertexRDD[Int] = scoreGraph.aggregateMessages(
    triplet => {
        triplet.sendToDst(1)         // No attribute is passed but an integer
    },
    (x, y) => x + y,
    TripletFields.None
)
```

To see, that this works, let's print the top 5 and bottom 5 teams:

```scala
scala> numWins.sortBy(_._2,false).take(5).foreach(println)
(1246,34)
(1437,32)
(1112,31)
(1458,31)
(1211,31)

scala> numLosses.sortBy(_._2, false).take(5).foreach(println)
(1363,28)
(1146,27)
(1212,27)
(1197,27)
(1263,27)
```

Should you want the name of the top 5 teams, you need to access the `srcAttr` attribute. In this case, we need to set the `tripletFields` to `TripletFields.Src`:

Kentucky as undefeated team in regular season:

```scala
val numWinsOfTeams: VertexRDD[(String, Int)] = scoreGraph.aggregateMessages(
    t => {
        t.sendToSrc(t.srcAttr, 1)         // Pass source attribute only
    },
    (x, y) => (x._1, x._2 + y._2),
    TripletFields.Src
)
```

Et voila!

```scala
scala> numWinsOfTeams.sortBy(_._2._2, false).take(5).foreach(println)
(1246,(Kentucky,34))
(1437,(Villanova,32))
(1112,(Arizona,31))
(1458,(Wisconsin,31))
(1211,(Gonzaga,31))

scala> numWinsOfTeams.sortBy(_._2._2).take(5).foreach(println)
(1146,(Cent Arkansas,2))
(1197,(Florida A&M,2))
(1398,(Tennessee St,3))
(1263,(Maine,3))
(1420,(UMBC,4))
```

Kentucky has not lost any of its 34 games during the regular season. Too bad that they could not make it into the championship final. 


## Warning about MapReduceTriplets operator 

Prior to Spark 1.2, there was no `aggregateMessages` method in `Graph`. Instead, the now deprecated `mapReduceTriplets` was the primary aggregation operator. The API for `mapReduceTriplets` is: 

```scala
class Graph[VD, ED] {
  def mapReduceTriplets[Msg](
      map: EdgeTriplet[VD, ED] => Iterator[(VertexId, Msg)],
      reduce: (Msg, Msg) => Msg)
    : VertexRDD[Msg]
}
```

Compared to `mapReduceTriplets`, the new operator `aggregateMessages` is more expressive as it employs the message passing mechanism instead of returning an iterator of messages as `mapReduceTriplets` does. In addition, `aggregateMessages` explicitly requires the user to specify the TripletFields object for performance improvement as we explained above. In addition to API improvements, `aggregateMessages` is optimized for performance. 


Because `mapReduceTriplets` is now deprecated, we will not discuss it further. If you have to use it with earlier versions of Spark, you can refer to the Spark programming guide.


## Summary 

In brief, AggregateMessages is a useful and generic operator that provides a functional abstraction for aggregating neighbourhood information in Spark graphs. Its definition is summarized below:

```scala
class Graph[VD, ED] {
  def aggregateMessages[Msg: ClassTag](
      sendMsg: EdgeContext[VD, ED, Msg] => Unit,
      mergeMsg: (Msg, Msg) => Msg,
      tripletFields: TripletFields = TripletFields.All)
    : VertexRDD[Msg]
}
```

This operator applies a user defined `sendMsg` function to each edge in the graph using an `EdgeContext`. Each `EdgeContext` access the required information about the edge and passes that information to its source node and/or destination node using the `sendToSrc` and/or `sendToDst` respectively. After all messages are received by the nodes, the `mergeMsg` function is used to aggregate those messages at each node. 

## Some interesting readings

1. [__Six keys to sports analytics.__](http://newsoffice.mit.edu/2015/mit-sloan-sports-analytics-conference-0302)
2. [Moneyball: The Art Of Winning An Unfair Game](https://en.wikipedia.org/wiki/Moneyball)
3. [__Golden State Warriors at the forefront of NBA data analysis__](http://www.sfgate.com/warriors/article/Golden-State-Warriors-at-the-forefront-of-NBA-5753776.php)
4. [__How Data and Analytics Have Changed 'The Beautiful Game'__](http://www.huffingtonpost.com/travis-korte/soccer-analytics-data_b_5512271.html)
5. [NHL, SAP partnership to lead statistical revolution](http://www.nhl.com/ice/news.htm?id=754184)

