---
layout: post
author: Eric Miranda
tags: [database, golang]
---

[![NO AI](https://raw.githubusercontent.com/nuxy/no-ai-badge/master/badge.svg)](https://github.com/nuxy/no-ai-badge)
# mydb
Working at Target scale has made me insecure about my skills. With some services handling upwards of 160k RPS, I was quickly reminded of
how little I knew about building software at scale.

To remedy this, I dived into [DDIA](https://www.google.co.in/books/edition/Designing_Data_Intensive_Applications/zFheDgAAQBAJ?hl=en).
I realised quickly just how poor I am at understanding abstract concepts[^1] and so I chose to implement one of the chapters: a storage engine.

As I work through an implementation, I'll try to list out the pros & cons of the solution.
Computer science is all about tradeoffs and this becomes very evident when by trying to solve a con, 
we introduce a bunch more!

# Constraints & Requirements
- Our DB handles only key-value data
- I ignore a lot of details, such as concurrent writes, complex key-value data, reading while merging, etc. to focus
on the broader algorithmic ideas
- Smarter variations to my implementation exist that address various cons, but let's ignore that for the sake of simplicity!

# Implementations
### 1. Naive and Simple (How Everyone Sees Me)
Let's use a log and append every key-value to the end of the file. We read from the file by scanning through
all the keys. We'll call this file our `logfile`.

**Pros**
- Superfast writes
- Smooth brain like

**Cons**
1. O(n) reads
2. Disk space of overwritten keys are never reclaimed

### 2. Naive Byte Offset
Let's solve con 1 by using a key-value index. The key is the key being inserted into the database
and value is the byte offset of that key in the `logfile`.

**Cons**
1. [x] ~~O(n) reads~~
2. [ ] Disk space of overwritten keys are never reclaimed
3. [ ] Writes have become more expensive (as we have to maintain an index)
4. [ ] Index as metadata increases storage size
5. [ ] On restarts, we need to scan the entire log to rebuild the index which is very expensive
6. [ ] No. of keys is the amount of RAM you have
7. [ ] Unable to make range queries efficiently (give me all keys between 'apple' and 'bat')

Solving a single con resulted in another 5! Tradeoffs!

#### Byte Offset + Merging & Compaction
Let's solve con 2 & 5.
Rather than maintaining one massive `logfile` we'll write the log to disk after every x mb. This is called a segment.
On this segment, we could run
compaction (replacing old key-values with only the latest occurrence). Further, we could also merge multiple segments
into 1, allowing for compaction across segments, referred to as merging.

**Cons**
1. [x] ~~Disk space of overwritten keys are never reclaimed~~
2. [ ] Writes are still expensive
3. [ ] Index as metadata increases storage size
4. [x] ~~On restarts, we need to scan the entire log to rebuild the index which is very expensive (Now we only require
   to scan smaller, compacted segments)~~
5. [ ] No. of keys is the amount of RAM you have
6. [ ] Unable to make range queries efficiently (give me all keys / values between 'apple' and 'bat')


### SSTables
Let's solve con 5 & 6, using an SSTable!
Simply put, an SSTable is a logfile with key-value records sorted by the key. This provides a few benefits
since the keys are sorted:
1. We can use the k-way merge algorithm (the merge step in mergesort) while merging & compacting. Since we didn't know whether the incoming key was the latest in the previous approach, we needed to maintain an index in memory.
Everytime we saw that key, we overwrote it. However, if the case where the number of unique keys is greater
than available RAM, we're stuck.  
SSTables allow us to look at only the first entry in all the segment iterators to know
which key is the latest (newest segment wins). This allows us to bring down our space requirements greatly
2. We can maintain a sparse index (every key doesn't need an entry, we can get an approximation & then search
linearly), reducing memory requirements
3. Since keys are sorted, compression algorithms can take advantage of data locality

**Cons**
1. [ ] Writes remain expensive
2. [ ] Index as metadata increases storage size, although relatively less due to sparse indexing
3. [x] ~~No. of keys is the amount of RAM you have~~
4. [x] ~~Unable to make range queries efficiently (give me all keys / values between 'apple' and 'bat')~~
5. [ ] Smooth brain takes time to understand the DSA required for implementation

### Conclusion
Wohoo! We've just built a discount, mom-i-want-an-ordered-key-value-storage-engine-mom-we-have-one-at-home
[rocksDB](), [levelDB]()!

### What I've Learned
Backing this article is an actual implementation at my github: [mydb](https://github.com/ericmiranda7/mydb).
The storage engine is named `Nob`, expressing how noobish I felt writing one.

This was perhaps the most complex (in terms of technicality) project I've worked on, having to build
- My own in memory tree for enabling SSTables,
- Using k-way merge type algorithms for the merging & compaction process,
- Upper bound binsearch for finding block offsets
- File IO with byte offsetting

What helped me immeasurably during the implementation was
- Unit testing &
- Breaking down complex requirements into smaller, simpler tasks

It's amazing how far the 2 techniques above can get you but I often find that I lack the discipline
to stick with it, diving straight into a complex task only to find myself 2 weeks later pulling my hair
out staring at the broken state of my system.

mydb is still a very broken implementation, but unfortunately I must continue with DDIA and so I'll bring
this project to an end.

### What I Should've Done Better
- As I didn't plan on writing an article for this, I didn't maintain code checkpoints for previous implementations.
It would've been nice to insert some pseudocode for each implementation.

### What Comes Next
- I'll probably take a serious stab at [flyio challenges](https://fly.io/dist-sys/), that seems to tie
in well with the distributed concepts I'm learning from DDIA!

[^1]: The book is very well-written, I am just very well-unread


































