# Writeup for Lab 2

## Describe any design decisions you made.

I continued to use `Stream`, because they are really cooooooool!

Besides, I modified `HeapFile.HeapFileIterator`, because `Stream.reduce` is not lazy, thus it will cost double memory when iterating. I changed it from one iterator into two iterators, and the final code is just slightly longer.

## Discuss and justify any changes you made to the API.

First there is a hidden bug in `BTreeInternalPageReverseIterator`. To be specific, there is a bug in its `hasNext()`:

```java
if (nextChildId != null) {
	nextToReturn = new BTreeEntry(key, nextChildId, childId);
	nextToReturn.setRecordId(recordId);
	childId = nextChildId;
	// key = p.getKey(entry); // the wrong version
	key = (entry > 0 ? p.getKey(entry) : null); // the fixed version
	recordId = new RecordId(p.pid, entry);
	return true;
}
```

When `entry` equals to 0, `p.getKey(entry)` will throw an exception, and jump out to the `catch` block, causing the method to return `false` unexpectedly. In fact whether the next key exists does not matter. This bug costs me a lot of time, and I think it should be fixed.

By the way, I added `TupleDesc.makeCopy` in lab1 for the sake of security. But later I found it is a waste of time and memory, so I removed it.

I did not change other given code except fixing typos.

## Describe any missing or incomplete elements of your code.

Still, I am very lazy, so any unnecessary code is missing.

## Describe how long you spent on the lab, and whether there was anything you found particularly difficult or confusing.

It is hard to say how long I spent, because I went to do other things in the middle. Roughly it may be about 15 hours in total.

Most time was spent on debugging, because the methods are a bit hard to implement, and it is easy to make bugs.

I want to complain that, the comments and the README are not helpful enough. I spent a lot of time on reading the code and trying to understand it, and I was confused thinking whether I should handle some work by myself or just leave them to the given code. I think there could be more comments and explanations.
