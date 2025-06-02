# Chrome IndexedDB index performance issue

I noticed some performance issues in IndexedDB with indexes in large object stores where the index values in the records changes many times. Like if you have an index on the "foo" property, and the values you have in "foo" in your records change over time.

In this example I made, it creates 500,000 nearly identical records in an object store with one index. 1,000 of those records have 1 for the index value, and the rest have 0. `getAll(1)` takes about 100ms on my machine.

Then, I scan over all the records and change the values of the indexed property. Still 1,000 of them are 1 and 499,000 are 0, but those values are on different records than before. `getAll(1)` is now a little slower.

Repeat this many times and it keeps getting slower:

![Chart showing Chrome getting slower over time](https://github.com/dumbmatter/chrome-indexeddb-index-perf/raw/master/chart.png)

I ran the same test in Firefox 139 and did not observe any slowdown. Even after 1,000 iterations, it still takes about 100 ms.

I also ran the same test in Chrome 100 to see if this issue has been around for a while... and it has! Chrome 100 performs similarly to Chrome 136.

In the web app where I noticed this behavior, I have user reports that some queries like this are taking minutes when they should be taking <1 second.

## It might be a subtle bug!

A slightly different version of this code does not get slower.

index2.html is the same code except when shuffling the index values, it's written more efficiently. The difference is:

- index.html - uses openCursor to scan over the entire store, only actually calling `cursor.update` on objects where the index value needs tochange
- index2.html - after switching all 1s to 0s with `getAll(1)` (fast since there are only 1,000 of them), it precomputes the primary keys of the 1,000 new records we want to have index value 1 and uses `cursor.continue(nextKey)` to skip reading the other objects into memory

In both cases, the same writes to the object store are happening (1000 updated from 1 to 0, and 1000 updated from 0 to 1)
