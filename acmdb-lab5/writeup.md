# Writeup for Lab 5

## Describe any design decisions you made.

I used BFS instead of recursive DFS to traverse the dependency graph in `BufferPool`. I think it is more efficient.

## Discuss and justify any changes you made to the API.

I added `.equals()` for `Tuple` because there seems to be a bug in the test.

Additionally, I changed `DbException` and `TransactionAbortedException` to `RuntimeException`, because I need to throw them when working on stream. This may be ugly, but is the simplest way to pass the test, without changing streams back to for-loops.

## Describe any missing or incomplete elements of your code.

There is no missing or incomplete elements in my code for lab5. I passed all the tests.

## Describe how long you spent on the lab, and whether there was anything you found particularly difficult or confusing.

6~8 hours roughly. I think lab5 is more difficult than all the labs before. There is no pre-written code for dependency graph or locks, so I have to try several ways to implement them.