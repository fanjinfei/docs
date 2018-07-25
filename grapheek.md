[1.] How did I come to develop my own graph base:
------------------
> ``
> This article is a return of experience more than a year after the beginning of the development.

Disclaimer: troll and subjectivity are waiting for you and maybe some information, who knows?

[2.] Once upon a time there was a recommendation algorithm
---------------------
Last year, I had the opportunity to develop a graphical database in pure Python as part of the project I was doing with a company in Nantes specialized in the monitoring of public markets and the linking of companies: Jurismarchés.

In this project, it was necessary to develop a recommendation algorithm to propose public contracts similar to those which had been consulted by the user, which is not a problem in itself but which becomes quickly when the volume of documents in the base is important (in this project there were several million).

The basic principles were simple: to extract the textual contents of the documents available, tokenize these documents, to delete the tokens without added values ​​(for example the articles: 'the', 'the', 'the', etc ...), to index the documents according to tokens, then identify similar documents through simple principles, for example:

-    large number of common tokens
-    a large number of high added value tokens (those whose frequency in the document is well above average frequencies in all documents)

In short, use the "classic" principles of natural language analysis (if you are interested in the subject, I highly recommend watching NLTK).
It took me a little bit of thinking before designing a relevant referral system, but I would not go into detail here.

The question then was to serialize the data in a database to avoid loading all the data in memory each time I wanted to run the algorithm, then write the query that would magically produce, from of one or more given documents, a list of documents to recommend.

> ``
>   The relational databases were clearly not appropriate: it would have been necessary, at a time, to make at least 5 joins on tables of several million lines which would have posed important problems of performance, the goal being to propose recommendations in less than 100ms.

Passionate about graph algorithms and graphical databases since reading this book, I turned to graphs and told myself that it should be possible to do something useful.

Working almost exclusively with Python, I then turned to NetworkX, a magnificent graph manipulation library, provided with a large number of well implemented and documented algorithms ... happy inspiration: by implementing my recommendation algorithms with NetworkX, I obtained the expected performance.

But, problem, NetworkX works exclusively in memory (at least the official version, there were attempts to fork to add persistence but did not go very far) and it was necessary to wait several minutes to fully load the graph. memory ... hum ... hardly compatible with a production version using Django frontend where each process would have its own version of the graph in memory (which represented several GB of data)

In short, I arrived at the point where:

- I had a recommendation algorithm that worked by relying on a graph
- I could load this graph in memory
- But each web worker had his own version of the graph, each taking several GB in memory
- The different graph versions quickly became inconsistent (oh yes, I forgot to say it: permanently new documents were added and induced the addition of nodes and links)
- And last, most scary, for any devops: the possibility of a crash that would have lost the memory graph and would have required a full reload, de facto blocking the recommendation system for several minutes and for each Web worker ... not glop

>  So, no choice: you had to find a database database graph, ideally Python.

[3.]  In search of the ideal graph base
-------------------
After some Google research, I realize the obvious:

>  If we want to use an Open Source graphical database, we are almost obliged to use a Java database

language to which I am particularly reluctant (too wordy, too heavy, not funny), fortunately, there are Python bindings for most basic graphs and even a library (Bulbs) that allows you to build models (Django) and which allows you to create simple queries.

So I start by playing with Bulbs and Neo4J ... good starting impressions:

*  Neo4J seems to be a product well done:
    *  it is possible to visualize its nodes and relationships in a browser, the query language is well done, we feel that it is clearly inspired by the SQL
    * it seems to do the job but the performance on my graph algos are not terrible: from 10 to 100x less than my implementation with NetworkX
*  Bulbs is also a nice project: I can quickly connect to the Neo4J database, add nodes, add relationships, all in pure Python

But quickly, I disillusion:

*  The performances are not at the rendezvous
*  (I'm probably bad but) I can not find a way to dump a database into a file with a nice format without causing these Java exceptions that make me vomit since my first experiences with Tomcat - certainly, I could copy the database if one day , I want to relocate it to another machine, but this solution does not please me (by experience, the binary formats can quickly be a wound, especially, when the version that generated it is no longer available)
*  It is rather quickly necessary to write queries with the query language before sending them by Bulbs <=> rather quickly, we no longer feel like writing Python and, conversely, we are invaded by the query language and our code becomes ugly: impression of mixing HTML with a popular web prog language if you know what I mean: p

[4.]  After the disappointment, maybe the grail?
----------------------------
In short, abandon Neo4J and Bulbs

>  Warning: Neo4J and Bulbs are both great projects, for Neo4J, I just had to spend 6 months upgrading to Java and I managed to auto-switch to edit XML configuration files, when at Bulbs, I do not doubt that it has its place in an application stack if you are comfortable with Neo4J and its query language.

New Google search: this time, I'm not looking directly at a graph base but I look at the different DSL that allow to query it, with the hope of finding the equivalent of SQL in the world of databases graphs: a query language standardized.

Well, it does not exist but there is still a DSL that seems to be a de facto standard: Gremlin, language designed by Tinkerpop

There, big slap, beautiful discovery: a hyper-synthetic DSL, well designed that allows to write at the speed of light complex graph algorithms in a few lines, even in one-liner, when the equivalent in relational database would require dozens or even hundreds of lines of SQL code ...

If you do not know this DSL, I invite you to discover it in this video, the speaker, Marko Rodriguez is simply bluffing by his knowledge of the bases graphs but also by his speed of speech and the speed with which he chained the different examples and graph algorithms.

Another advantage of Gremlin: there are drivers for most databases Graphs, I could particularly use it with:

-  Neo4J (Java)
-  OrientDB (Java)
-  Titan (Java)
-  ArangoDB (C / C ++)

>  It's a good thing to have a common dialect between all these databases, especially in the production environment since it makes it easy to change the database without having to recode all your queries.

*Note that for the latest versions of Neo4J, it is still advisable to use the query language of Neo4J*

[5.]()  Reimplementation of the recommendation algorithm with Gremlin
--------------------------------
After the discovery of Gremlin, I decided to reimplement my recommendation algorithm: it is relatively simple, Gremlin indeed relies on Groovy, a dynamic language itself based on Java, which has many similarities with Python, this which is not to displease me.

In fact, the most complex part is still interfacing with the database: if, on paper, many database graphs are supported by Gremlin, the installation of the drivers is always a little laborious (except Titan, but c is normal, this database was started by Gremlin developers) and you often have to look for a database version compatible with a supplied driver.

But in short, I can test the performance of the algorithm with the basics mentioned above (Neo4J / OrientDB / Titan / ArangoDB) and it's a new disappointment: it takes between 300ms (Titan) and 1s (Neo4J) to get the results of the algorithm of recommendation, it is not acceptable (at the maximum, it was necessary to have results after 100ms)

I run a profiling and finds that part of the time is necessarily devoted to the passage through the different interfacing layers, a return / return request / response similar to that:

-  Implementing the Python query
-  Launch via bulbs the request that sends it to rexster (a server defined by Tinkerpop that allows you to expose endpoints and ultimately execute Gremlin requests)
-  Launch of Gremlin query (with transition from Groovy to Java) on the graph database
-  Retrieving the answer
-  And we pass the layers in the opposite direction

In short, too many layers to pass, even with simple algorithms and small-sized graphs, we always have latencies too important.

Then there seems to be only one alternative: prepare the queries and save them in the databases to run them on demand

But:

-  the performance gain is still not enough
-  some of the code is in the base itself which is still a problem
-  Python code no longer looks like Python code

> It is at this moment that I make the decision to try to implement a base graph in pure Python:

*  Performance with NetworkX was good
*  NetworkX relies heavily on Python dictionaries
*  It is missing "just":
    *  Data persistence
    *  A language as pleasant to use as Gremlin to do the "path traversal"
*  For persistence of data, using a KVS (Key Value Store) is a logical option: a KVS is never, conceptually, a big, persistent dictionary.

[6.]  GrapheekDB: initial technical choices
-----------------------------
When I started designing GrapheekDB, I made the following choices:

*  Creating an abstraction layer to communicate with KVS or Python dictionaries
*  Selection of KVS with the following characteristics:
    *  Existence of transactions: to guarantee the consistency of the stored data (to have acceptable performance despite the intrinsic slowness of Python, I knew that I would have to denormalize massively and that this denormalization should not induce inconsistencies)
    *  Ability to use all available disk space <=> the size of the graph base should not be limited by available RAM or performance degrades when the size of the stored data exceeds that of the memory lively (what we see with Redis for example)
    *  Good performance at least in reading, ideally in writing: it is a very subjective choice, I knew that for my needs, the number of readings would be several orders higher than the number of writings.
*  Development of a stand-alone server to allow multiple processes to access the central graph base
*  While retaining the ability to work directly with the underlying data
*  By guaranteeing a unified API that we work directly with the data or that we go through a server, in other words, code a "proxy" that has exactly the same API as if we go into "live"
*  The API had to be a mix between Gremlin and Django and stay "python-like":
    *  Gremlin, because it's a beautiful DSL, extremely expressive
    *  Django, because the first project in which the base graph was used is a Django project and it was better to have a "familiar" API

>  Incidentally, as I know myself and I always tend to favor speed in the well-known equation of project managers (time / features / quality / costs), I add the following elements and constraints:

*  Large number of tests: GrapheekDB is my first open source project and as one of my friends says, open source is like slips, when you show it, it has to be clean
*  High level of code coverage by tests, ideally 100% (magré all the reserves that can be done on the meaning of 100% coverage rate ...)

My main goal was:

> Obtain performance, for the recommendation algorithm, superior to those obtained with existing graph bases (in Java)

This objective conditioning the availability of the code (Open Source).

To those who will not fail to argue that it might have been better to try to tune an existing Java database, I say: yes, but I only have one life and then oh, GrapheekDB, it's opensource and gratos! ;)

(That's a real answer, huh!)

In short, you will understand, I reached my goal: in the end, the performances were good, I got recommendations after 10 to 50ms

If you want to see what GrapheekDB looks like, you can find slides from the presentation made at school 42

There is also a video

[7.] Balance sheet time
---------------------------
*  The base is stable (I have an instance that runs for more than a year and has a hundred thousand knots).
*  The project seems to interest Python developers since there are generally between 50 and 100 daily downloads.

**Personally :**  

*  At the moment, the result satisfies me, in the sense that it meets my needs.
*  I consider this open source project a fulfillment and a fair return to the OpenSource community after two decades of using open source tools and limited contributions (to Django)
*  I'm all the more pleased that this tool quickly allows all Python developers to play with a base graph and a DSL close to Gremlin
*  My code is definitely not the purest in the world (deflating ankles)
*  But I still code so fast: p (reglangling of the ankles), sometimes too much (pchhhhhh ....)
*  Fortunately on the advice of Adrien, I set up Coverage, it allowed me to really test in depth, detect vicious bugs and get a very stable result.  

**What I would like to improve:**  
*  The documentation :
    *  I am relatively happy with the tutorial
    *  but the code is very little documented
    *  it is clearly missing a user documentation, which is all the more unfortunate that interesting functions have been added since the writing of the tutorial
*  Writing performance, even if my basic graph needs are WORM (Write Once Read Many), sometimes WFRM (Write Few Read Many), it is clear that GrapheekDB is fishing on this point :(
*  The index system: there are clearly missing indexes to accelerate interval type filters  
**For further :**  
I am often asked if I consider that GrapheekDB is stable enough to be used in production: the answer for me is yes because having created the code, I can correct any bugs, but for others, it is rather: I think that you have to try (and normally, there is a tendency to hear "no";))

The fact is, that at present:

*  GrapheekDB is deployed in production.
*  I make sure to maintain a very high level of tests (> 2000) and coverage (~ 100%)
*  I quickly fix the bugs that are logged on Bitbucket
*  I add features when they seem to me to have an interest for all users
In short:  

> If you're a Python developer and you want to play with basic graphs that interests me too, I can only advise you to play with GrapheekDB, it's a real graph base, with a dialect inspired by Gremlin and filters inspired by Django

**To go elsewhere**  
If you're looking for a richer graph base, more powerful and a good DSL that goes with it, I can only advise you to go see Titan, well, it's in Java, we can not have everything ...

*The links*  

-  Python package from GrapheekDB: https://pypi.python.org/pypi/grapheekdb
-  Repository of the project: https://bitbucket.org/nidusfr/grapheekdb
-  Slides: http://slides.com/nidus_en/grapheekdb#/
-  Video of the presentation made at school 42: http://www.dailymotion.com/video/x1ql18e_conf-42-meetup-paris-py-4_tech
