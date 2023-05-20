# Writeup for Lab 4

## Describe any design decisions you made.

I used a partitioned data structure ("分块" in Chinese) for optimization in `IntHistogram`. The reason of using it instead of BIT is that, I think querying is less frequent than modifying, so it's worth to sacrefice the performance of querying.

Additionally I improved several pre-prepared methods such as `computeCostAndCardOfSubplan`. I changed the naive implementation using `Set` into direcly using an integer as the bitmask.

## Discuss and justify any changes you made to the API.

I changed some APIs in `JoinOptimizer` due to the improvement mentioned above.

## Describe any missing or incomplete elements of your code.

I am lazy, so there may be some unnecessary part missing. Anyway I passed all the tests.

## Describe how long you spent on the lab, and whether there was anything you found particularly difficult or confusing.

May be 6~8 hours. I didn't remember precisely because I got covid-19, and I was sick for several days.

I spent most of the time reviewing lessons about query optimization. The coding part is not difficult.
