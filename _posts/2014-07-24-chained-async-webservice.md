---
layout: post
title: Composable async without blocking
comments: True
---

I recently had a requirement to fetch a bunch of JSON data (actually, a *sequence* of JSON objects) from a web service and enrich that data with extra information (which is also retrieved from yet another web service). The requirement was to provide the enriched data object back to the client -- and to do so without blocking on the server. This post documents my efforts.


## First out of the blocks
My specific use case was to enrich a list of groups with the user-membership of each group. The idea can be illustrated using some dummy data and test webservice calls. Instead of using external fictitious WS calls, let's first set up some of our own to keep all functionality self-contained (this example uses activator-1.2.3):

``` scala

val group1 = Group("g001", "group1", None)
val group2 = Group("g002", "group2", None)
val group3 = Group("g003", "group3", None)

val fred = User("u001", "Fred")
val wilma = User("u002", "Wilma")
val barney = User("u003", "Barney")

val dummyGroups = Seq (group1, group2, group3)

val dummyGroupMembers = Map (
   "g001" -> Seq(fred, wilma, barney),
   "g002" -> Seq(),
   "g003" -> Seq(fred)
)

// dummy calls
def testGroup = Action {
  Ok(Json.toJson(dummyGroups))
}

def testGroupUsers(gid: String) = Action {
  val res = dummyGroupMembers(gid)
  Ok(Json.toJson(res))
}
```

..and in the `routes` file:

``` scala
GET /testgroup			  controllers.Groups.testGroup
GET /testgroup/:gid/users controllers.Groups.testGroupUsers(gid: String)
```

Next, let's create an async call to the first groups webservice:

``` scala
def findGroups() = Action.async {

  val url = "http://localhost:9000/testgroup"

  val resultFuture = WS.url(url).get()
  resultFuture.map {
    response => {

      val groups = response.json
      val groupsResult = groups.validate[Seq[Group]]

      groupsResult match {
        case s: JsSuccess[Seq[Group]] => {
          val groups: Seq[Group] = s.get
          Ok(Json.toJson(enrich(groups)))
        }
        case e: JsError => {
          NotFound("no groups found")
        }
      }
    }
  } 
}
```
So far, so good all nice and asynchro...

Wait... (Await?)

Hang on...

What's this `enrich` business?

``` scala
def enrich(grps: Seq[Group]): Seq[Group] = {
  grps map { grp => 
  	Group(grp.id, 
  	 	  grp.name, 
  		  Some(Await.result(groupUsers(grp.id), 5 seconds))) 
  }
}

def groupUsers(gid: String): Future[Seq[User]] = {

  val url = s"http://localhost:9000/testgroup/${gid}/users"

  WS.url(url).get().map {
    response => {
      val users = response.json
      val usersResult = users.validate[Seq[User]]
      usersResult match {
        case s: JsSuccess[Seq[User]] => s.get
        case _ => Seq()
      }
    }
  }
}
```

So much for non-blocking. Although the code above gets the job done, the enrichment step involves a call to `Await.result`, and this kind of thing is best avoided if at all possible.

Surely there's a better way?

## Not the first..

Of course there is. And others have had this kind of problem before. Yevgeniy Brikman (@brikis98) has a <a href="http://engineering.linkedin.com/play/play-framework-async-io-without-thread-pool-and-callback-hell">great post on the subject</a>. After reading it, I was hoping to solve my problem with a nice elegant piece of code looking something like the following:

<script src="https://gist.github.com/brikis98/5235740.js"></script>

I figured it would go something like this:

``` scala
for {
	g <- groups
	users <- groupUsers(g.id)
} yield (Group(g.name, g.id, Some(users)))
```

Unfortunately, the scala compiler had other ideas and I was greeted with messages of the following form:

``` scala
error: type mismatch; found : scala.concurrent.Future[Int] 
required: scala.collection.GenTraversableOnce[?]
```

Some googling brought me to <a href="http://stackoverflow.com/questions/20108523/combining-scala-futures-and-collections-in-for-comprehensions">this StackOverflow post</a> which provides a good explanation of why these types of mismatch occur.

## Sequencing
Where my use case differs from Yevgeniy Brikman's example is that rather than chaining an async call on to the end of *a single* call, I needed to do it *for each* item in the *sequence*. Aha! Returning to StackOverflow, I find yet another <a href="http://stackoverflow.com/questions/9992400/multiple-ws-call-in-one-action-how-to-handle-promise-objects">promising avenue</a> worth investigating that involves the user of `Promise.sequence`.

Well, it turns out that `Promise.sequence` has been deprecated, however `Future.sequence` has risen to take its place.

Here's how I ended up using it to remove the troublesome `Await` call:

``` scala
def findGroups = Action.async {

  // xs is a Future[Seq[Future[Group]]
  val xs = allGroups map { groups =>
    for {
      g <- groups
      gu = groupUsers(g.id)
    } yield ( for { u <- gu } yield (Group(g.id, g.name, Some(u))) )
  }

  // convert to Future[Future[Seq[Group]]]
  val ys = xs map (x => Future.sequence(x))
  
  // flatten to Future[Seq[Group]]
  val resultFuture = ys flatMap(x => x)

  resultFuture map { response` => 
    Ok(Json.toJson(response))
  } recover {
    case NonFatal(t) =>
      Ok(EMPTY_JSON)
  }
}
```

The `allGroups` code is just a modified version of `findGroups` that returns a `Future[Seq[Group]]` rather than pulling out the JSON.

All code for the above is <a href="https://github.com/nacmacfeegle/AsyncWSChain">available on GitHub</a>. 

Go fork, yourself!

