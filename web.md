
We are excited to make an initial beta release of GraphGen; a system
towards enabling graph analytics on top of relational datasets. GraphGen allows users to _declaratively_ specify graph extraction tasks over relational databases, visually explore the extracted graphs, and write and execute graph algorithms over them, either directly using our vertex-centric framework or using existing graph libraries like the widely used NetworkX Python library. GraphGen was demonstrated at _VLDB 2015_; For a few more details see our [demo paper](http://www.vldb.org/pvldb/vol8/p2032-xirogiannopoulos.pdf) or view the [video](https://youtu.be/GDVBLv-oedQ) that explains the main functionality of the demo and core research challenges in GraphGen. We also have a followup paper in the works so stay tuned!

GraphGen currently only supports [PostgreSQL](http://www.postgresql.org/) as the backend relational database.

## This is cool, but is it really necessary?
Graph analytics and graph algorithms have proven their worth time and again, having provided substantial value to various different domains like social networks, communication networks, finance, health, and many others. However if the data stored for a particular application is not geared towards some network-specific task or the application itself is not network-centric, users will logically not choose to store their data in a native graph store or in a graph format separating out Nodes and Edges. These users would likely use a conventional, mature and often more reliable relational database. Nevertheless these users may still want to apply these graph analyses onto their data in order to power their application, perhaps though building a machine learning model or just trying to figure out the inner-workings of their company by exploring their inner e-mail network etc. GraphGen is therefore built towards enabling users who have gone with the latter choice to _efficiently_ conduct their desired in-memory graph analyses on the data stored in their normalized relational databases without the need to manually go through time and money consuming ETL processes with often sub-optimal results.

Through its simplicity and intuitiveness, GraphGen _opens up_ graph analytics on any relational dataset by enabling exploration of different types of graphs that can be inferred just by inspecting the schema. If you can think of entities and relationships between them that exist in your schema, you can build a graph out of it!

## Using GraphGen with Python
The easiest way to try out GraphGen is probably in Python through `graphgenpy`; a Python wrapper library for executing graph extraction queries in our custom DSL which is based on Datalog. The graphs that result from the extraction query are serialized to disk in a standard format and can then be _loaded_ into other graph libraries like [`NetworkX`](https://networkx.github.io/) for further analysis.

###Installing graphgenpy

To install `graphgenpy` onto your system, simply download the `graphgen-pkg`, and run:
`python setup.py install` in the uncompressed directory.

After that you can immediately use `graphgenpy` to extract and serialize your graphs onto disk and use them as you please.

__Example:__

Here is the script that loads a toy _dblp_ database that will allow you to play around with `graphgenpy`.
This script is also included in `graphgen-pkg.zip`.

```python
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT
import pprint
import sys

username = "kostasx"; #<< postgres username
password = "testdb"; #<< postgres password
dbname = "testgraphgen"; #<< change this name if you already have a valuable database with that name...I think it's safe you assume you don't

# create the new test database
try:
    conn = psycopg2.connect("dbname='postgres' user="+username+" host = 'localhost' password="+password)
except:
    print "I am unable to connect to Postgres."

conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)
cur = conn.cursor()
cur.execute("DROP DATABASE "+dbname+";")
cur.execute("CREATE DATABASE "+dbname+";")

# connect to the newly created database
try:
    conn = psycopg2.connect("dbname='"+dbname+"' user="+username+" host='localhost' password="+password+"")
except:
    print "I am unable to connect to the database"

cur = conn.cursor()
try:
    authorTable = "CREATE TABLE author (id integer NOT NULL PRIMARY KEY,name character varying(1024));"
    cur.execute(authorTable)

    conference = "CREATE TABLE conference (id integer NOT NULL PRIMARY KEY,name character varying(1024),year integer,location character varying(1024));"
    cur.execute(conference)

    pub = "CREATE TABLE publication (id integer NOT NULL PRIMARY KEY,title character varying(2048),cid integer NOT NULL REFERENCES conference(id));"
    cur.execute(pub)

    authorpub = "CREATE TABLE authorpublication (aid integer NOT NULL  REFERENCES author(id), pid integer NOT NULL REFERENCES publication(id), PRIMARY KEY(aid,pid));"
    cur.execute(authorpub)

    # # execute our Query
    authors = """INSERT INTO author (id, name) VALUES
        (1,'Anindya Datta'),
        (2,'Heiko Schuldt'),
        (3,'Sandeepan Banerjee'),
        (4,'Christophe Bobineau'),
        (5,'Sangyong Hwang'),
        (6,'Tirthankar Lahiri'),
        (7,'Evangelos Eleftheriou'),
        (8,'Quan Z. Sheng'),
        (9,'Egemen Tanin'),
        (10,'Brandon Lloyd');"""
    cur.execute(authors)

    conferences = """ INSERT INTO conference (id, name,year,location) VALUES
        (49,'VLDB',2014,'Hangzhou, China'),
        (87,'VLDB',2015,'Kailua Kona, HI USA'),
        (84,'SIGMOD',2014,'Snowbird, Utah'),
        (36,'SIGMOD',2015,'Melbourne, Australia'),
        (59,'CIDR',2015,'Asilomar, CA USA');"""
    cur.execute(conferences)

    publications = """ INSERT INTO publication (id, title,cid) VALUES
        (8,'Title 1.',49),
        (15,'Title 2.',87),
        (25,'Title 3.',84),
        (44,'Title 4.',36),
        (64,'Title 5.',59);
        """
    cur.execute(publications)
    authorpubs = """INSERT INTO authorpublication (aid, pid) VALUES
        (1,8),
        (1,64),
        (1,44),
        (2,8),
        (3,15),
        (4,15),
        (5,25),
        (5,44),
        (6,25),
        (7,25),
        (7,8),
        (8,44),
        (9,64),
        (10,64),
        (10,44),
        (10,8),
        (3,64);"""
    cur.execute(authorpubs)

    conn.commit();
    cur.close()
    conn.close()
except:
    print "Unexpected error:", sys.exc_info()[0]
```

After that, `graphgenpy` is very simple to use
```python
from graphgenpy import GraphGenerator
import networkx as nx

datalogQuery = """
Nodes(ID, Name) :- Author(ID, Name).
Edges(ID1, ID2) :- AuthorPublication(ID1, PubID), AuthorPublication(ID2, PubID).
"""

# Credentials for connecting to the database
gg = GraphGenerator("localhost","5432","testgraphgen","kostasx","password") #All these must be strings!!

# Evaluate graph extraction query and serialize the resulting graph to disk in a standard format. Return the file's name in the FS.
fname = gg.generateGraph(datalogQuery,"extracted_graph",GraphGenerator.GML)

# Load graph into NetworkX
# by default, the graph format will me gml. 'id' here is the node attribute networkx should use as a label (mandatory)
G = nx.read_gml(fname,'id')
print "Graph Loaded into NetworkX! Running PageRank..."

# Run any algorithm on the graph using NetowrkX
print nx.pagerank(G)
print "Done!"
```

##Sounds great, but how do I write queries??

We have defined a declarative language based on _Datalog_ through which users are able to express graph extraction queries by expressing how the nodes and the edges should be projected from the database.

Assume a _dblp_ database with the following tables:

- Author
- Publication
- AuthorPublication
- Conference

__Extract a graph where authors are connected to each other if they've published a paper together:__

```java

Nodes(ID, Name) :- Author(ID, Name).

Edges(ID1, ID2) :- AuthorPublication(ID1, PubID), AuthorPublication(ID2, PubID).

```

Let's take a look at how this query is formulated!

![DBLP Schema](https://docs.google.com/drawings/d/17obOWt2DYJtX0-NwIESSK8n7fSLrcAvrsJ4fxsPqpk8/pub?w=853&h=337)

There are two different types of _atoms_ (which is Datalog for predicates or table names) that can exist on the left hand side of each query; `Nodes` and `Edges`. In the current iteration of GraphGen we are supporting properties _only_ on the `Nodes`, so no support for `Edge` properties for now. These `Node` properties can be selected from the right-hand side and listed after the `ID` in the `Nodes` arguments. Note that the ID _must_ be the first constant in the arguments. The right hand side consists of all the atoms, and the variables and predicates involved in the query.

Let's break down this specific query which yields the co-authorship graph.

In the first line we are expressing how to project the Nodes in our graph. Here, the `Nodes` are selected from the `Author` table where their Id (`ID`) would be each distinct `aid` in the `Author` table (the ordering of the arguments mirrors the ordering of the attributes of an atom in its adjacent table); We are therefore creating a Node for every Author, and including the `name` attribute in the table as a Node property (aliased `Name`). **Note that the Node ID needs to be _unique_.**

The second line describes how each two `Nodes` will be connected to eachother, hence the graph's `Edges`. This line expresses that Node with `ID1` and `ID2` will be connected to each other, if there exists tuples in the AuthorPublication relation where they have the same publication id (`PubID`). **Note that `ID1` and `ID2` both have to refer to the same exact range of identifiers dictated by the ones we have given to the `Nodes` of the graph.**

The most exciting aspect about GraphGen is that by inspecting the schema users can envision various different ways that `Nodes` can be defined, as well as ways that these nodes can be connected to each other. For example from this

Another (slightly more complex) example would be using the well known _imdb_ database which amongst others includes the following tables in its schema:

- cast_info : Contains all information on the cast and crew of all movies
- role_type : Contains the role_ids and their names
- name : Contains the names of all cast and crew of all movies

__Extract a graph where actors are connected to each other if they've played in the same movie__

```java

Nodes(id, name) :- name(id, name), cast_info(_, id , _, role_id),
role_type(role_id, role_name), role_name = "actor".

Edges(id1, id2) :- cast_info(_, id1, movie_id),
cast_info(_, id2, movie_id, role_id), role_type(role_id, role_name),
role_name = "actor".

```

The first line creates a node for every person in the dataset that has been in a movie as an actor, and the second one creates links between these actors if they've played in the same movie together.

__A few things to keep in mind__:

- Think of each line of code as a single expression.
- The first thing in the parameters of the _Nodes_ atoms should be the ID of the chosen node type and should be __unique__

## Using GraphGen with Java

GraphGen is written in Java and through it, we support space efficient in-memory analytics on the extracted graph through user defined vertex-centric programs. In order to use GraphGen in Java you need to import the GraphGen library located at :
`graphgenpy/lib/GraphGen-0.0.5-SNAPSHOT-jar-with-dependencies.jar`

__Example:__
```java

// Establish Connection to Database
GraphGenerator ggen = new GraphGenerator("host", "port", "dbName",
    "username", "password");

// Define and evaluate a single graph extraction query
String datalog_query = "...";
Graph g = ggen.generateGraph(datalog_query).get(0);

// Initialize vertec-centric object
VertexCentric p = new VertexCentric(g);

// Define vertex-centric compute function
Executor program = new Executor("result_value_name") {
 @Override
 public void compute(Vertex v, VertexCentric p) {
    // implementation of compute function
  }
};
// Begin execution
p.run(program);

```

## How Do I write vertex-centric Programs??

There are only a few things that users need to know about when writing vertex centric programs using the GraphGen framework:

__1. Iterating over the current vertex's  neighbors__
```java
// v is the current vertex
for (Vertex e : v.getVertices(Direction.BOTH)) {
  //...
}
```

__2. Setting the value for the current vertex__
```java
v.setVal(...);
```

__3. Getting the current Superstep__
```java
int s = p.getSuperstep()
```

__4. Fetching one of the node's properties__
```java
int degree = v.getProperty("Degree");
```

__5. Voting the node to a halt__
```java
p.voteToHalt(v);
```


###Example of _PageRank_ Calculation using GraphGen (30 iterations)
```java
// Initialize vertec-centric object
VertexCentric p = new VertexCentric(g);

//The property id for the result will be "PageRank"
Executor pagerank = new Executor("PageRank") {
@Override
public void compute(Vertex v, VertexCentric p) {
 GenVertex gv = ((GenVertex) v);
 int degree = v.getProperty("Degree");

 if (p.getSuperstep() == 0) {
  gv.setVal(1.0 / degree); // initialize
 }

 if (p.getSuperstep() >= 1) {
  double sum = 0;
  for (Vertex e : v.getVertices(Direction.BOTH)) {
  sum += (double) ((GenVertex) e).getPrevVal();
 }

  double newPageRank = 0.15 / p.getNumOfVertices() + 0.85 * sum;
  gv.setVal(newPageRank / degree);
 }

 if (p.getSuperstep() == 30) {
  gv.setVal((double) gv.getPrevVal() * degree);
  p.voteToHalt(gv);
 }
}
};

//Run
p.run(pagerank);
```