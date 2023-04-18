# Lab 3: SimpleDB Operators

**Assigned: April 14th, 2021**<br>
**Due: May 5th, 2021 11:59 PM**


In this lab assignment, you will write a set of operators for SimpleDB 
to implement table modifications (e.g., insert and delete records), 
selections, joins, and aggregates. These will build on top of the
foundation that you wrote in Lab 1 and Lab 2 to provide you with a 
database system that can perform simple queries over multiple tables.


Additionally, you will also utilize the buffer eviction code you have 
developed from your lab 3. You do not need to implement transactions or 
locking in this lab.


The remainder of this document gives some suggestions about how to start
coding, describes a set of exercises to help you work through the lab, 
and discusses how to hand in your code. This lab requires you to
write a fair amount of code, so we encourage you to **start early**!


## 1. Getting started 

You should begin with the code you submitted for Lab 1 and Lab 2 (if you did not submit code for Lab 1 and Lab 2, or your solution didn't work properly, contact us to discuss options). We have provided you with extra test cases for this lab that are not in the original code distribution you received in Lab 1 and Lab 2. We reiterate that the unit tests we provide are to help guide your implementation along, but they are not intended to be comprehensive or to establish correctness.

You will need to add these new test cases to your release. The easiest way to do this is to untar the new code in the same directory as your top-level simpledb directory, as follows:


* Make a copy of your Lab 2 solution by typing:
```console
$ cp -r acmdb-lab2 acmdb-lab3
```
* Change to the directory that contains your top-level simpledb code:
```console
$ cd acmdb-lab3
```
* Download the new tests and skeleton code for Lab 3 from [acmdb-lab3-supplement.tar.gz](assets/acmdb-lab3-supplement.tar.gz). (You can also find this file on Canvas.) Then place it in the acmdb-lab3 folder.

* Extract the new files for Lab 3 by typing:
```console
$ tar -xvzf acmdb-lab3-supplement.tar.gz
```

**Note**: Intellij/Eclipse users will take one more step for their code to compile: add new libraries `zql.jar` and `jline-0.9.94.jar` in your project. You may have already done this step from Lab 2; in that case you can ignore this step . Your code should now compile.


### 1.1. Implementation hints 


As before, we **strongly encourage** you to read through this entire document to 
get a feel for the high-level design of SimpleDB before you write code. 

We suggest exercises along this document to guide your implementation, but
you may find that a different order makes more sense for you.  As before,
we will grade your assignment by looking at your code and verifying that
you have passed the test for the ant targets `test` and
`systemtest`. See Section 3.4 for a complete discussion of
grading and list of the tests you will need to pass.

Here's a rough outline of one way you might proceed with your SimpleDB
implementation;  more details on the steps in this outline, including
exercises, are given in Section 2 below.



*  Implement the operators `Filter` and `Join` and
verify that their corresponding tests work. The Javadoc comments for
these operators contain details about how they should work.  We have given you implementations of
  `Project` and `OrderBy` which may help you
  understand how other operators work.

*  Implement `IntegerAggregator` and `StringAggregator`. Here, you will write the
logic that actually computes an aggregate over a particular field across
multiple groups in a sequence of input tuples. Use integer division for
computing the average, since SimpleDB only supports integers. `StringAggegator`
only needs to support the COUNT aggregate, since the other operations do not
make sense for strings.  

*  Implement the `Aggregate` operator.  As with other
operators, aggregates implement the `DbIterator` interface
so that they can be placed in SimpleDB query plans.  Note that the
output of an `Aggregate` operator is an aggregate value of an
entire group for each call to `next()`, and that the
aggregate constructor takes the aggregation and grouping fields.

*  Implement the methods related to tuple insertion, deletion, and page
eviction in `BufferPool`. You do not need to worry about
transactions at this point.

*  Implement the `Insert` and `Delete` operators.
Like all operators,  `Insert` and `Delete` implement
`DbIterator`, accepting a stream of tuples to insert or delete
and outputting a single tuple with an integer field that indicates the
number of tuples inserted or deleted.  These operators will need to call
the appropriate methods in `BufferPool` that actually modify the
 pages on disk.  Check that the tests for inserting and
deleting tuples work properly.  

  Note that SimpleDB does not implement any kind of consistency or integrity
checking, so it is possible to insert duplicate records into a file and
there is no way to enforce primary or foreign key constraints.



At this point you should be able to pass the tests in the ant
  `systemtest` target, which is the goal of this lab.

  

  You'll also be able to use the provided SQL parser to run SQL
  queries against your database!  See [Section 2.6](#parser) for a
  brief tutorial and a description of an optional contest to see who can write the fastest SimpleDB implementation. (This contest is for fun and will not be graded.)




Finally, you might notice that the iterators in this lab extend the
`Operator` class instead of implementing the DbIterator
interface.  Because the implementation of <tt>next</tt>/<tt>hasNext</tt>
is often repetitive, annoying, and error-prone, `Operator`
implements this logic generically, and only requires that you implement
a simpler <tt>readNext</tt>.  Feel free to use this style of
implementation, or just implement the `DbIterator` interface if you prefer.
To implement the DbIterator interface, remove `extends Operator`
from iterator classes, and in its place put `implements DbIterator`.



## 2. SimpleDB Architecture and Implementation Guide 


### 2.1. Filter and Join 

Recall that SimpleDB DbIterator classes implement the operations of the
relational algebra. You will now implement two operators that will enable
you to perform queries that are slightly more interesting than a table
scan.



* *Filter*: This operator only returns tuples that satisfy
a `Predicate` that is specified as part of its constructor. Hence,
it filters out any tuples that do not match the predicate.

* *Join*: This operator joins tuples from its two children according to
a `JoinPredicate` that is passed in as part of its constructor.
We only require a simple nested loops join, but you may explore more
interesting join implementations. Describe your implementation in your lab
writeup.



**Exercise 1.**

  Implement the skeleton methods in:

***  
  *  src/simpledb/Predicate.java
  *  src/simpledb/JoinPredicate.java
  *  src/simpledb/Filter.java
  *  src/simpledb/Join.java
  *  src/simpledb/HashEquiJoin.java
  
***  
  
  At this point, your code should pass the unit tests in
  PredicateTest, JoinPredicateTest, FilterTest, JoinTest and HashEquiJoinTest. Furthermore,
  you should be able to pass the system tests FilterTest, JoinTest and HashEquiJoinTest.



### 2.2. Aggregates 

An additional SimpleDB operator implements basic SQL aggregates with a
`GROUP BY` clause. You should implement the five SQL aggregates
(`COUNT`, `SUM`, `AVG`, `MIN`,
`MAX`) and support grouping.  You only need to support aggregates
over a single field, and grouping by a single field. 



In order to calculate aggregates, we use an `Aggregator`
interface which merges a new tuple into the existing calculation of an
aggregate. The `Aggregator` is told during construction what
operation it should use for aggregation.  Subsequently, the client code
should call `Aggregator.mergeTupleIntoGroup()` for every tuple in the child
iterator. After all tuples have been merged, the client can retrieve a
DbIterator of aggregation results. Each tuple in the result is a pair of
the form `(groupValue, aggregateValue)`, unless the value
of the group by field was `Aggregator.NO_GROUPING`, in which
case the result is a single tuple of the form `(aggregateValue)`.



Note that this implementation requires space linear in the number of
distinct groups.  For the purposes of this lab, you do not need to worry
about the situation where the number of groups exceeds available memory.

 

**Exercise 2.**

  Implement the skeleton methods in:
  
***  
  *  src/simpledb/IntegerAggregator.java
  *  src/simpledb/StringAggregator.java
  *  src/simpledb/Aggregate.java
  
***  

  At this point, your code should pass the unit tests
  IntegerAggregatorTest, StringAggregatorTest, and
  AggregateTest. Furthermore, you should be able to pass the AggregateTest system test.


### 2.3. HeapFile Mutability 

Now, we will begin to implement methods to support modifying tables. We
begin at the level of individual pages and files.  There are two main sets
of operations:  adding tuples and removing tuples.

**Removing tuples:** To remove a tuple, you will need to implement
`deleteTuple`.
Tuples contain `RecordIDs` which allow you to find
the page they reside on, so this should be as simple as locating the page 
a tuple belongs to and modifying the headers of the page appropriately.

**Adding tuples:** The `insertTuple` method in
`HeapFile.java` is responsible for adding a tuple to a heap
file.  To add a new tuple to a HeapFile, you will have to find a page with
an empty slot. If no such pages exist in the HeapFile, you
need to create a new page and append it to the physical file on disk.  You will
need to ensure that the RecordID in the tuple is updated correctly.

**Exercise 3.**

  Implement the remaining skeleton methods in:

***  
  *  src/simpledb/HeapPage.java
  *  src/simpledb/HeapFile.java<br>
     (Note that you do not necessarily need to implement writePage at this point).

***
  
  

  To implement HeapPage, you will need to modify the header bitmap for
  methods such as <tt>insertTuple()</tt> and <tt>deleteTuple()</tt>. You may
  find that the <tt>getNumEmptySlots()</tt> and <tt>isSlotUsed()</tt> methods we asked you to
  implement in Lab 1 serve as useful abstractions.  Note that there is a
  <tt>markSlotUsed</tt> method provided as an abstraction to modify the filled
  or cleared status of a tuple in the page header.

  
   Note that it is important that the <tt>HeapFile.insertTuple()</tt>
   and <tt>HeapFile.deleteTuple()</tt> methods access pages using
   the <tt>BufferPool.getPage()</tt> method; otherwise, your
   implementation of transactions in the next lab will not work
   properly.
 

  Implement the following skeleton methods in <tt>src/simpledb/BufferPool.java</tt>:  
  
***  
  *  insertTuple()
  *  deleteTuple()
  
***  
    
  
  These methods should call the appropriate methods in the HeapFile that
  belong to the table being modified (this extra level of indirection is
  needed to support other types of files &mdash; like indices &mdash; in the
  future).

  

  At this point, your code should pass the unit tests in HeapPageWriteTest and
  HeapFileWriteTest. We have not provided additional unit tests for `HeapFile.deleteTuple()` or `BufferPool`. 

 


### 2.4. Insertion and deletion 

Now that you have written all of the HeapFile machinery to add and remove
tuples, you will implement the `Insert` and `Delete`
operators. 



For plans that implement `insert` and `delete` queries,
the top-most operator is a special `Insert` or `Delete`
operator that modifies the pages on disk.  These operators return the number
of affected tuples. This is implemented by returning a single tuple with one
integer field, containing the count.



* *Insert*: This operator adds the tuples it reads from its child
operator to the `tableid` specified in its constructor.  It should
use the `BufferPool.insertTuple()` method to do this.

* *Delete*: This operator deletes the tuples it reads from its child
operator from the `tableid` specified in its constructor.  It
should use the `BufferPool.deleteTuple()` method to do this.



 

**Exercise 4.**

  Implement the skeleton methods in:
  
***  
  *  src/simpledb/Insert.java
  *  src/simpledb/Delete.java
  
***  

  At this point, your code should pass the unit tests in InsertTest. We
  have not provided unit tests for `Delete`. Furthermore, you
  should be able to pass  the InsertTest and DeleteTest system tests.


<a name="query_walkthrough"></a>
### 2.5. Query walkthrough

The following code implements a simple join query between two tables, each
consisting of three columns of integers.  (The file
`some_data_file1.dat` and `some_data_file2.dat` are
binary representation of the pages from this file). This code is equivalent
to the SQL statement:

```sql
SELECT * 
  FROM some_data_file1, some_data_file2 
  WHERE some_data_file1.field1 = some_data_file2.field1
  AND some_data_file1.id > 1
```

For more extensive examples of query operations, you may find it helpful to
browse the unit tests for joins, filters, and aggregates.
 
```java
package simpledb;
import java.io.*;

public class jointest {

    public static void main(String[] argv) {
        // construct a 3-column table schema
        Type types[] = new Type[]{ Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE };
        String names[] = new String[]{ "field0", "field1", "field2" };

        TupleDesc td = new TupleDesc(types, names);

        // create the tables, associate them with the data files
        // and tell the catalog about the schema  the tables.
        HeapFile table1 = new HeapFile(new File("some_data_file1.dat"), td);
        Database.getCatalog().addTable(table1, "t1");

        HeapFile table2 = new HeapFile(new File("some_data_file2.dat"), td);
        Database.getCatalog().addTable(table2, "t2");

        // construct the query: we use two SeqScans, which spoonfeed
        // tuples via iterators into join
        TransactionId tid = new TransactionId();

        SeqScan ss1 = new SeqScan(tid, table1.getId(), "t1");
        SeqScan ss2 = new SeqScan(tid, table2.getId(), "t2");

        // create a filter for the where condition
        Filter sf1 = new Filter(
                                new Predicate(0,
                                Predicate.Op.GREATER_THAN, new IntField(1)),  ss1);

        JoinPredicate p = new JoinPredicate(1, Predicate.Op.EQUALS, 1);
        Join j = new Join(p, sf1, ss2);

        // and run it
        try {
            j.open();
            while (j.hasNext()) {
                Tuple tup = j.next();
                System.out.println(tup);
            }
            j.close();
            Database.getBufferPool().transactionComplete(tid);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```



Both tables have three integer fields. To express this, we create
a `TupleDesc` object and pass it an array of `Type`
objects indicating field types and `String` objects
indicating field names. Once we have created this `TupleDesc`, we initialize
two `HeapFile` objects representing the tables.  Once we have
created the tables, we add them to the Catalog. (If this were a database
server that was already running, we would have this catalog information
loaded; we need to load this only for the purposes of this test).



Once we have finished initializing the database system, we create a query
plan.  Our plan consists of two `SeqScan` operators that scan
the tuples from each file on disk, connected to a `Filter`
operator on the first HeapFile, connected to a `Join` operator
that joins the tuples in the tables according to the
`JoinPredicate`.  In general, these operators are instantiated
with references to the appropriate table (in the case of SeqScan) or child
operator (in the case of e.g., Join). The test program then repeatedly
calls `next` on the `Join` operator, which in turn
pulls tuples from its children. As tuples are output from the
`Join`, they are printed out on the command line.

<a name="parser"></a>
### 2.6. Query Parser and Contest

We've provided you with a query parser for SimpleDB that you can use
to write and run SQL queries against your database once you
have completed the exercises in this lab.

The first step is to create some data tables and a catalog.  Suppose
you have a file `data.txt` with the following contents:

```
1,10
2,20
3,30
4,40
5,50
5,50
```

You can convert this into a SimpleDB table using the
`convert` command (make sure to type <tt>ant</tt> first!):

```
java -jar dist/simpledb.jar convert data.txt 2 "int,int"
```

This creates a file `data.dat`. In addition to the table's
raw data, the two additional parameters specify that each record has
two fields and that their types are `int` and
`int`.



Next, create a catalog file, `catalog.txt`,
with the following contents:

```
data (f1 int, f2 int)
```

This tells SimpleDB that there is one table, `data` (stored in
`data.dat`) with two integer fields named `f1`
and `f2`.

Finally, invoke the parser.
You must run java from the
command line (ant doesn't work properly with interactive targets.)
  From the `simpledb/` directory, type:

```
java -jar dist/simpledb.jar parser catalog.txt
```

You should see output like:

```
Added table : data with schema INT(f1), INT(f2), 
SimpleDB> 
```

Finally, you can run a query:

```
SimpleDB> select d.f1, d.f2 from data d;
Started a new transaction tid = 1221852405823
 ADDING TABLE d(data) TO tableMap
     TABLE HAS  tupleDesc INT(d.f1), INT(d.f2), 
1       10
2       20
3       30
4       40
5       50
5       50

 6 rows.
----------------
0.16 seconds

SimpleDB> 
```

The parser is relatively full featured (including support for SELECTs,
INSERTs, DELETEs, and transactions), but does have some problems
and does not necessarily report completely informative error
messages.  Here are some limitations to bear in mind:


*  You must preface every field name with its table name, even if
  the field name is unique (you can use table name aliases, as in the
  example above, but you cannot use the AS keyword.)

*  Nested queries are supported in the WHERE clause, but not the
 FROM clause.
  
*  No arithmetic expressions are supported (for example, you can't
 take the sum of two fields.)
  
*  At most one GROUP BY and one aggregate column are allowed.
  
*  Set-oriented operators like IN, UNION, and EXCEPT are not
 allowed.

*  Only AND expressions in the WHERE clause are allowed.

*  UPDATE expressions are not supported.
  
*  The string operator LIKE is allowed, but must be written out
 fully (that is, the Postgres tilde [~] shorthand is not allowed.) 

**Contest**

We have built a SimpleDB-encoded version of the DBLP database; the needed files are located at: [dblp_data.tar.gz](assets/dblp_data.tar.gz). You can also find it on Canvas.

You should download the file and unpack it. It will create four files in the dblp_data directory. Move them into the simpledb/dataset directory. The following commands will acomplish this, if you run them from the simpledb directory:

```
mkdir dataset
cd dataset
tar xvzf dblp_data.tar.gz  # make sure to put the download tar file in current folder
mv dblp_data/* .
rm -r dblp_data.tar.gz dblp_data
```

You can then run the parser with:

```
java -jar dist/simpledb.jar parser dataset/dblp_simpledb.schema
```

Or, you can simply use the script startSequential.sh provided in the Lab 3 distribution under the /bin folder:

```
cd bin
chmod 755 startSequential.sh
./startSequential.sh ../dataset/dblp_simpledb.schema
```

You can report the total runtime for the following three queries (where total runtime is the sum of the runtime of each of the individual queries):

```sql
SELECT p.title
FROM papers p
WHERE p.title LIKE 'selectivity';
```

```sql
SELECT p.title, v.name
FROM papers p, authors a, paperauths pa, venues v
WHERE a.name = 'E. F. Codd'
AND pa.authorid = a.id
AND pa.paperid = p.id
AND p.venueid = v.id;
```

```sql
SELECT a2.name, count(p.id)
FROM papers p, authors a1, authors a2, paperauths pa1, paperauths pa2
WHERE a1.name = 'Michael Stonebraker'
AND pa1.authorid = a1.id 
AND pa1.paperid = p.id 
AND pa2.authorid = a2.id 
AND pa1.paperid = pa2.paperid
GROUP BY a2.name
ORDER BY a2.name;
```

Note that each query will print out its runtime after it executes.

You may wish to create optimized implementations of some of the operators; in particular, a fast join operator (e.g., not nested loops) will be essential for good performance on queries 2 and 3.

There is currently no optimizer in the parser, so the queries above have been written to cause the parser generate reasonable plans. Here are some helpful hints about how the parser works that you may wish to exploit while running these queries:

* The table on the left side of the joins in these queries is passed in as the first DbIterator parameter to Join.
* Expressions in the WHERE clause are added to the plan from top to bottom, such that first expression will be the bottom-most operator in the generated query plan. For example, the generated plan for Query 2 is:

```
Project(Join(Join(Filter(a),pa),p))
```

Our reference implementation can run Query 1 in about .35 seconds, Query 2 in about 1.5 seconds, and Query 3 in about 3.75 seconds. We implemented a special-purpose join operator for equality joins but did little else to optimize performance.


## 3. Logistics 

You must submit your code (see below) as well as a short (2 pages, maximum)
writeup describing your approach.  This writeup should:



*  Describe any design decisions you made, including your choice of page
  eviction policy. If you used something other than a nested-loops join,
  describe the tradeoffs of the algorithm you chose.

*  Discuss and justify any changes you made to the API.

*  Describe any missing or incomplete elements of your code.

*  Describe how long you spent on the lab, and whether there was anything
  you found particularly difficult or confusing.

* (Optional) Describe the total running time of the sql contest.


###  3.1. Collaboration 

This lab should be manageable for a single person.  Therefore, teaming is prohibited in this project.

### 3.2. Submitting your assignment
To submit your code, please create a `acmdb-lab3` directory in your github repo. Please submit your writeup as a PDF or plain text file (.txt) in the top level of your `acmdb-lab3` directory. Please do not submit a .doc or .docx.

###  3.3. Submitting a bug 

Please submit (friendly!) bug reports to both TAs. When you do, please try to include:

* A description of the bug.
* A `.java` file we can drop in the `test/simpledb` directory, compile, and run.
* A `.txt` file with the data that reproduces the bug. We should be able to convert it to a `.dat` file using HeapFileEncoder.

###  3.4. Grading 

<p>100% of your grade will be based on whether or not your code passes the test suite we will run over it on docker. Before handing in your code, you should make sure it produces no errors (passes all of the tests) from both  <tt>ant test</tt> and <tt>ant systemtest</tt>.


**Important:** before testing, we will replace your <tt>build.xml</tt> and the entire contents of the <tt>test</tt> directory with our version of these files.  This means you cannot change the format of <tt>.dat</tt> files!  You should also be careful changing our APIs. You should test that your code compiles the unmodified tests. 

In other words, we will pull your repo, replace the files mentioned above, compile it, and then grade it.  It will look roughly like this:

```
[replace build.xml and test]
$ git checkout -- build.xml test\
$ ant test
$ ant systemtest
```

<p>If any of these commands fail, we'll be unhappy, and, therefore, so will your grade.
We've had a lot of fun designing this assignment, and we hope you enjoy hacking on it!
