# Lab 1: SimpleDB 

**Assigned: April 6th**

**Due: April 20th, 23:59 PM**

##  1. Implementation Hints 

For this lab, you will focus on implementing the core modules required to access stored data on disk. Before beginning to write code, we **strongly encourage** you to read through this entire document to get a feel for the high-level design of SimpleDB. 


The code for Lab1 is already given to you in Lab 0. You should rename the folder to move forward.

```console
$ mv acmdb-lab0 acmdb-lab1
```

You will need to fill in any piece of code that is not implemented.  It will be obvious where we think you should write code. You may need to add private methods and/or helper classes.  You may change APIs, but make sure our tests still run and make sure to mention, explain, and defend your decisions in your writeup.

<p>

In addition to the methods that you need to fill out for this lab,  the class interfaces contain numerous methods that you need not implement until subsequent labs. These will either be indicated per class:

```java
// Not necessary for lab1.
public class Insert implements DbIterator {
```

or per method:

```Java
public boolean deleteTuple(Tuple t) throws DbException {
    // some code goes here
    // not necessary for lab1
    return false;
}
```


The code that you submit should compile without having to modify these methods.

<p>

We suggest exercises along this document to guide your implementation, but you may find that a different order makes more sense for you.

**Here's a rough outline of one way you might proceed with your SimpleDB implementation:**

****
* Implement the classes to manage tuples, namely Tuple, TupleDesc. We have already implemented Field, IntField, StringField, and Type for you. Since you only need to support integer and (fixed length) string fields and fixed length tuples, these are straightforward.
* Implement the Catalog (this should be very simple).
* Implement the BufferPool constructor and the getPage() method.
* Implement the access methods, HeapPage and HeapFile and associated ID classes. A good portion of these files has already been written for you.
* Implement the operator SeqScan.
* At this point, you should be able to pass the ScanTest system test, which is the goal for this lab. 

***

 Section 2 below walks you through these implementation steps and the unit tests corresponding to each one in more detail. 



As you look through the interfaces we have provided you, you will see a number of references to locking, transactions, and recovery.  You do not need to support these features in this lab, but you should keep these parameters in the interfaces of your code because you will be implementing transactions and locking in a future lab. The test code we have provided you with generates a fake transaction ID that is passed into the operators of the query it runs; you should pass this transaction ID into other operators and the buffer pool.

##  2. SimpleDB Architecture and Implementation Guide 

SimpleDB consists of:


 *   Classes that represent fields, tuples, and tuple schemas;
 *   Classes that apply predicates and conditions to tuples;
 *   One or more access methods (e.g., heap files) that store relations on disk and provide a way to iterate through tuples of those relations;
 *   A collection of operator classes (e.g., select, join, insert, delete, etc.) that process tuples;
 *   A buffer pool that caches active tuples and pages in memory and handles concurrency control and transactions (neither of which you need to worry about for this lab); and,
 *   A catalog that stores information about available tables and their schemas. 


SimpleDB does not include many things that you may think of as being a part of a "database."  In particular, SimpleDB does not have:

*   (In this lab), a SQL front end or parser that allows you to type queries directly into SimpleDB.  Instead, queries are built up by chaining a set of operators together into a hand-built query plan (see [Section 2.7](#query_walkthrough)).  We will provide a simple parser for use in later labs.
*   Views.
*  Data types except integers and fixed length strings.
* (In this lab) Query optimizer.
* (In this lab) Indices.

<p>

In the rest of this Section, we describe each of the main components of SimpleDB that you will need to implement in this lab.  You should use the exercises in this discussion to guide your implementation. This document is by no means a complete specification for SimpleDB; you will need to make decisions about how to design and implement various parts of the system.  Note that for Lab 1 you do not need to implement any operators (e.g., select, join, project) except sequential scan. You will add support for additional operators in future labs.

<p>

###  2.1. The Database Class 

The Database class provides access to a collection of static objects that are the global state of the database.  In particular, this includes methods to access the catalog (the list of all the tables in the database), the buffer pool (the collection of database file pages that are currently resident in memory), and the log file. You will not need to worry about the log file in this lab. We have implemented the Database class for you.  You should take a look at this file as you will need to access these objects.

###  2.2. Fields and Tuples 

<p>Tuples in SimpleDB are quite basic.  They consist of a collection of `Field` objects, one per field in the `Tuple`. `Field` is an interface that different data types (e.g., integer, string) implement.  `Tuple` objects are created by the underlying access methods (e.g., heap files, or B-trees), as described in the next section.  Tuples also have a type (or schema), called a _tuple descriptor_, represented by a `TupleDesc` object.  This object consists of a collection of `Type` objects, one per field in the tuple, each of which describes the type of the corresponding field.

### Exercise 1

**Implement the skeleton methods in:**
***
 * src/simpledb/TupleDesc.java
 * src/simpledb/Tuple.java 

***


At this point, your code should pass the unit tests TupleTest and TupleDescTest.  At this point, modifyRecordId() should fail because you haven't implemented it yet.

###  2.3. Catalog 

The catalog (class `Catalog` in SimpleDB) consists of a list of the tables and schemas of the tables that are currently in the database.  You will need to support the ability to add a new table, as well as getting information about a particular table.  Associated with each table is a `TupleDesc` object that allows operators to determine the types and number of fields in a table.

The global catalog is a single instance of `Catalog` that is allocated for the entire SimpleDB process. The global catalog can be retrieved via the method `Database.getCatalog()`, and the same goes for the global buffer pool (using `Database.getBufferPool()`).

### Exercise 2

**Implement the skeleton methods in:**
***
 * src/simpledb/Catalog.java

***

At this point, your code should pass the unit tests in CatalogTest.


###  2.4. BufferPool 

<p>The buffer pool (class `BufferPool` in SimpleDB) is responsible for caching pages in memory that have been recently read from disk. All operators read and write pages from various files on disk through the buffer pool. It consists of a fixed number of pages, defined by the `numPages` parameter to the `BufferPool` constructor. In later labs, you will implement an eviction policy. For this lab, you only need to implement the constructor and the `BufferPool.getPage()` method used by the SeqScan operator. The BufferPool should store up to `numPages` pages. For this lab, if more than `numPages` requests are made for different pages, then instead of implementing an eviction policy, you may throw a DbException. In future labs you will be required to implement an eviction policy.

The `Database` class provides a static method, `Database.getBufferPool()`, that returns a reference to the single BufferPool instance for the entire SimpleDB process.

### Exercise 3

**Implement the `getPage()` method in:**

***
* src/simpledb/BufferPool.java

***

We have not provided unit tests for BufferPool.  The functionality you implemented will be tested in the implementation of HeapFile below. You should use the `DbFile.readPage` method to access pages of a DbFile.


<!--
When more than this many pages are in the buffer pool, one page should be
evicted from the pool before the next is loaded.  The choice of eviction
policy is up to you; it is not necessary to do something sophisticated.
-->

<!--
<p>
Notice that `BufferPool` asks you to implement
a `flush_all_pages()` method.  This is not something you would ever
need in a real implementation of a buffer pool.  However, we need this method
for testing purposes.  You really should never call this method from anywhere
in your code.
-->

###  2.5. HeapFile access method 

Access methods provide a way to read or write data from disk that is arranged in a specific way.  Common access methods include heap files (unsorted files of tuples) and B-trees; for this assignment, you will only implement a heap file access method, and we have written some of the code for you.

<p>

A `HeapFile` object is arranged into a set of pages, each of which consists of a fixed number of bytes for storing tuples, (defined by the constant `BufferPool.DEFAULT_PAGE_SIZE`), including a header.  In SimpleDB, there is one `HeapFile` object for each table in the database.  Each page in a `HeapFile` is arranged as a set of slots, each of which can hold one tuple (tuples for a given table in SimpleDB are all of the same size).  In addition to these slots, each page has a header that consists of a bitmap with one bit per tuple slot.  If the bit corresponding to a particular tuple is 1, it indicates that the tuple is valid; if it is 0, the tuple is invalid (e.g., has been deleted or was never initialized.)  Pages of `HeapFile` objects are of type `HeapPage` which implements the `Page` interface.  Pages are stored in the buffer pool but are read and written by the `HeapFile` class.

<p>

SimpleDB stores heap files on disk in more or less the same format they are stored in memory.  Each file consists of page data arranged consecutively on disk.  Each page consists of one or more bytes representing the header, followed by the _page size_ bytes of actual page content. Each tuple requires _tuple size_ * 8 bits  for its content and 1 bit for the header. Thus, the number of tuples that can  fit in a single page is:

<p>

`
    _tuples per page_ = floor((_page size_ * 8) / (_tuple size_ * 8 + 1))
`

<p>

Where _tuple size_ is the size of a tuple in the page in bytes.  The idea here is that each tuple requires one additional bit of storage in the header. We compute the number of bits in a page (by mulitplying  page size by 8), and divide this quantity by the number of bits in a tuple (including this extra header bit) to get the number of tuples per page. The floor operation rounds down to the nearest integer number of tuples (we don't want to store partial tuples on a page!)

<p>

Once we know the number of tuples per page, the number of bytes required to store the header is simply:
<p>

`
     headerBytes = ceiling(tupsPerPage/8)
`

<p>

The ceiling operation rounds up to the nearest integer number of bytes (we never store less than a full byte of header information.)

<p>

The low (least significant) bits of each byte represents the status of the slots that are earlier in the file.  Hence, the lowest bit of the first byte represents whether or not the first slot in the page is in use. The second  lowest bit of the first byte represents whether or not the second slot in the  page is in use, and so on. Also, note that the high-order bits of the last byte may not correspond to a slot that is actually in the file, since the number of slots may not be a multiple of 8. Also note that all Java virtual machines are [big-endian](http://en.wikipedia.org/wiki/Endianness).

<p>

### Exercise 4



**Implement the skeleton methods in:**

***
*    src/simpledb/HeapPageId.java
*    src/simpledb/RecordID.java
*    src/simpledb/HeapPage.java 

***


Although you will not use them directly in Lab 1, we ask you to implement <tt>getNumEmptySlots()</tt> and <tt>isSlotUsed()</tt> in HeapPage. These require pushing around bits in the page header. You may find it helpful to look at the other methods that have been provided in HeapPage or in <tt>src/simpledb/HeapFileEncoder.java</tt> to understand the layout of pages. 


You will also need to implement an Iterator over the tuples in the page, which may involve an auxiliary class or data structure.  

At this point, your code should pass the unit tests in HeapPageIdTest, RecordIDTest, and HeapPageReadTest.


<p> 

After you have implemented <tt>HeapPage</tt>, you will write methods for <tt>HeapFile</tt> in this lab to calculate the number of pages in a file and to read a page from the file. You will then be able to fetch tuples from a file stored on disk.

### Exercise 5

**Implement the skeleton methods in:**

***
* src/simpledb/HeapFile.java

***

To read a page from disk, you will first need to calculate the correct offset in the file.  Hint: you will need random access to the file in order to read and write pages at arbitrary offsets.  You should not call BufferPool methods when reading a page from disk.

<p> 
You will also need to implement the `HeapFile.iterator()` method, which should iterate through through the tuples of each page in the HeapFile. The iterator must use the `BufferPool.getPage()` method to access pages in the `HeapFile`. This method loads the page into the buffer pool and will eventually be used (in a later lab) to implement locking-based concurrency control and recovery.  Do not load the entire table into memory on the open() call -- this will cause an out of memory error for very large tables.

<p>

At this point, your code should pass the unit tests in HeapFileReadTest.


###  2.6. Operators 

Operators are responsible for the actual execution of the query plan. They implement the operations of the relational algebra.  In SimpleDB, operators are iterator based; each operator implements the `DbIterator` interface.

<p>

Operators are connected together into a plan by passing lower-level operators into the constructors of higher-level operators, i.e., by 'chaining them together.'  Special access method operators at the leaves of the plan are responsible for reading data from the disk (and hence do not have any operators below them).

<p>

At the top of the plan, the program interacting with SimpleDB simply calls `getNext` on the root operator; this operator then calls `getNext` on its children, and so on, until these leaf operators are called.  They fetch tuples from disk and pass them up the tree (as return arguments to `getNext`); tuples propagate up the plan in this way until they are output at the root or combined or rejected by another operator in the plan.

<p>

<!--
For plans that implement `INSERT` and `DELETE` queries,
the top-most operator is a special `Insert` or `Delete`
operator that modifies the pages on disk.  These operators return a tuple
containing the count of the number of affected tuples to the user-level
program.

<p>
-->

For this lab, you will only need to implement one SimpleDB operator.

### Exercise 6.

**Implement the skeleton methods in:**

***
* src/simpledb/SeqScan.java

***
This operator sequentially scans all of the tuples from the pages of the table specified by the `tableid` in the constructor.  This operator should access tuples through the `DbFile.iterator()` method.

<p>At this point, you should be able to complete the ScanTest system test. Good work!

You will fill in other operators in subsequent labs.

<a name="query_walkthrough"></a>

### 2.7. A simple query

The purpose of this section is to illustrate how these various components are connected together to process a simple query.

Suppose you have a data file, "some_data_file.txt", with the following contents:
```
1,1,1
2,2,2 
3,4,4
(Be sure to add an empty line at the end of this file!)
```
<p>
Because you haven’t implemented "insertTuple" yet, you have no way to create data files during the first several labs. We provide you with a method that can convert a `.txt` file to a `.dat` file in SimpleDB's HeapFile format. Using the following command:

<p>

```console
$ ant dist  # explore your code to a jar package into folder "dist"
$ java -jar dist/simpledb.jar convert some_data_file.txt 3  # call the jar package
```

<p>
Here, the argument "3" tells conver that the input has 3 columns.

To view the contents of this table, use the print command:
```console
$ java -jar dist/simpledb.jar print some_data_file.dat 3
```

<p>
The following code implements a simple selection query over this file. This code is equivalent to the SQL statement `SELECT * FROM some_data_file`.


```Java
package simpledb;
import java.io.*;

public class test {

    public static void main(String[] argv) {

        // construct a 3-column table schema
        Type types[] = new Type[]{ Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE };
        String names[] = new String[]{ "field0", "field1", "field2" };
        TupleDesc descriptor = new TupleDesc(types, names);

        // create the table, associate it with some_data_file.dat
        // and tell the catalog about the schema of this table.
        HeapFile table1 = new HeapFile(new File("some_data_file.dat"), descriptor);
        Database.getCatalog().addTable(table1, "test");

        // construct the query: we use a simple SeqScan, which spoonfeeds
        // tuples via its iterator.
        TransactionId tid = new TransactionId();
        SeqScan f = new SeqScan(tid, table1.getId());

        try {
            // and run it
            f.open();
            while (f.hasNext()) {
                Tuple tup = f.next();
                System.out.println(tup);
            }
            f.close();
            Database.getBufferPool().transactionComplete(tid);
        } catch (Exception e) {
            System.out.println ("Exception : " + e);
        }
    }

}
```

The table we create has three integer fields.  To express this, we create a `TupleDesc` object and pass it an array of `Type` objects, and optionally an array of `String` field names. Once we  have created this `TupleDesc`, we initialize a `HeapFile` object representing the table stored in `some_data_file.dat`. Once we have created the table, we add it to the catalog. If this were a database server that was already running, we would have this catalog information loaded. We need to load it explicitly to make this code self-contained.

Once we have finished initializing the database system, we create a query plan.  Our plan consists only of the `SeqScan` operator that scans the tuples from disk. In general, these operators are instantiated with references to the appropriate table (in the case of `SeqScan`) or child operator (in the case of e.g. Filter). The test program then repeatedly calls `hasNext` and `next` on the `SeqScan` operator. As tuples are output from the `SeqScan`, they are printed out on the command line.

We **strongly recommend** you try this out as a fun end-to-end test that will help you get experience writing your own test programs for simpledb.  You should create the file "test.java" in the src/simpledb directory with the code above, and place the `some_data_file.dat` file in the top level directory.  Then run:

```
ant
java -classpath dist/simpledb.jar simpledb.test
```

Note that `ant` compiles `test.java` and generates a new jarfile that contains it.

##  3. Logistics 

You must submit your code (see below) as well as a short (2 pages, maximum) writeup describing your approach.  This writeup should:

*    Describe any design decisions you made. These may be minimal for Lab 1.
*    Discuss and justify any changes you made to the API.
*    Describe any missing or incomplete elements of your code.
*    Describe how long you spent on the lab, and whether there was anything you found particularly difficult or confusing. 

###  3.1. Collaboration 

This lab should be manageable for a single person.  Therefore, teaming is prohibited in this project.

### 3.2. Submitting your assignment
To submit your code, please create a `acmdb-lab1` directory in your github repo. Please submit your writeup as a PDF or plain text file (.txt) in the top level of your `acmdb-lab1` directory. Please do not submit a .doc or .docx.

###  3.3. Submitting a bug 

Please submit (friendly!) bug reports to both TAs. When you do, please try to include:

* A description of the bug.
* A `.java` file we can drop in the `test/simpledb` directory, compile, and run.
* A `.txt` file with the data that reproduces the bug. We should be able to convert it to a `.dat` file using HeapFileEncoder.

###  3.4. Grading 

<p>100% of your grade will be based on whether or not your code passes the test suite we will run over it. Before handing in your code, you should make sure it produces no errors (passes all of the tests) from both  <tt>ant test</tt> and <tt>ant systemtest</tt>.


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
