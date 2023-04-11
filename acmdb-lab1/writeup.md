# Writeup for Lab 1

## Describe any design decisions you made.

I used `Stream` a lot, this is mostly just because I'm interested in it, and I think it is really cool. Besides, using `Stream` when implementing some iterators is really convenient. I don't like to make unnecessary wheels.

I used `ArrayList` in `TupleDesc`, because I think simple array is kind of hard to use, and there should not be any performance issue since it will not be modified.

## Discuss and justify any changes you made to the API.

First I have to say that there are some typos in the README or the code. I was annoyed by lots of warnings, so I fixed them as what IDEA suggested.

I also added `.stream()` for `HeapPage`, because I want to use `Stream` in `HeapFileIterator`. This can avoid making wheels.

## Describe any missing or incomplete elements of your code.

I am very lazy, so any code unnecessary for lab1 is missing.

## Describe how long you spent on the lab, and whether there was anything you found particularly difficult or confusing.

Lab 1 is easy. I spent three nights of a weekend to finish it, though much of the time was wasted on reading the README and trying to install the environment.

I suggest that this project should be updated, since time has passed a lot. Also, the typos and warnings are annoying, please fix them if possible.
