polebug is translating

PostgreSQL's Hash Indexes Are Now Cool
=======
Since I just committed the last pending patch to improve hash indexes to PostgreSQL 11, and since most of the improvements to hash indexes were committed to PostgreSQL 10 which is expected to be released next week, it seems like a good time for a brief review of all the work that has been done over the last 18 months or so.  Prior to version 10, hash indexes didn't perform well under concurrency, lacked write-ahead logging and thus were not safe in the face either of crashes or of replication, and were in various other ways second-class citizens.  In PostgreSQL 10, this is largely fixed.

While I was involved in some of the design, credit for the hash index improvements goes first and foremost to my colleague Amit Kapila, whose [blog entry on this topic is worth reading][1].  The problem with hash indexes wasn't simply that nobody had bothered to write the code for write-ahead logging, but that the code was not structured in a way that made it possible to add write-ahead logging that would actually work correctly.  To split a bucket, the system would lock the existing bucket (using a fairly inefficient locking mechanism), move half the tuples to the new bucket, compact the existing bucket, and release the lock.  Even if the individual changes had been logged, a crash at the wrong time could have left the index in a corrupted state.  So, Amit's first step was to redesign the locking mechanism.  The [new mechanism][2] allows scans and splits to proceed in parallel to some degree, and allows a split interrupted by an error or crash to be completed at a later time.  Once that was done  a bunch of resulting bugs fixed, and some refactoring work completed, another patch from Amit added [write-ahead logging support for hash indexes][3].

In the meantime, it was discovered that hash indexes had missed out on many fairly obvious performance improvements which had been applied to btree over the years.  Because hash indexes had no support for write-ahead logging, and because the old locking mechanism was so ponderous, there wasn't much motivation to make other performance improvements either.  However, this meant that if hash indexes were to become a really useful technology, there was more to do than just add write-ahead logging.  PostgreSQL's index access method abstraction layer allows indexes to retain a backend-private cache of information about the index so that the index itself need not be repeatedly consulted to obtain relevant metadata.  btree and spgist indexes were using this mechanism, but hash indexes were not, so my colleague Mithun Cy wrote a patch to [cache the hash index's metapage using this mechanism][4].  Similarly, btree indexes have an optimization called "single page vacuum" which opportunistically removes dead index pointers from index pages, preventing a huge amount of index bloat which would otherwise occur.  My colleague Ashutosh Sharma wrote a patch to [port this logic over to hash indexes][5], dramatically reducing index bloat there as well.  Finally, btree indexes have since 2006 had a feature to avoid locking and unlocking the same index page repeatedly -- instead, all tuples are slurped from the page in one shot and then returned one at a time.  Ashutosh Sharma also [ported this logic to hash indexes][6], but that optimization didn't make it into v10 for lack of time.  Of everything mentioned in this blog entry, this is the only improvement that won't show up until v11.

One of the more interesting aspects of the hash index work was the difficulty of determining whether the behavior was in fact correct.  Changes to locking behavior may fail only under heavy concurrency, while a bug in write-ahead logging will probably only manifest in the case of crash recovery.  Furthermore, in each case, the problems may be subtle.  It's not enough that things run without crashing; they must also produce the right answer in all cases, and this can be difficult to verify.  To assist in that task, my colleague Kuntal Ghosh followed up on work initially begun by Heikki Linnakangas and Michael Paquier and produced a WAL consistency checker that could be used not only as a private patch for developer testing but actually [committed to PostgreSQL][7].  The write-ahead logging code for hash indexes was extensively tested using this tool prior to commit, and it was very successful at finding bugs.  The tool is not limited to hash indexes, though: it can be used to validate the write-ahead logging code for other modules as well, including the heap, all index AMs we have today, and other things that are developed in the future.  In fact, it already succeeded in [finding a bug in BRIN][8].

While wal_consistency_checking is primarily a developer tool -- though it is suitable for use by users as well if a bug is suspected -- upgrades were also made to several tools intended for DBAs.  Jesper Pedersen wrote a patch to [upgrade the pageinspect contrib module with support for hash indexes][9], on which Ashutosh Sharma did further work and to which Peter Eisentraut contributed test cases (which was a really good idea, since those test cases promptly failed, provoking several rounds of bug-fixing).  The pgstattuple contrib module also [got support for hash indexes][10], due to work by Ashutosh Sharma.

Along the way, there were a few other performance improvements as well.  One thing that I had not realized at the outset is that when a hash index begins a new round of bucket splits, the size on disk tended to abruptly double, which is not really a problem for a 1MB index but is a bit unfortunate if you happen to have a 64GB index.  Mithun Cy addressed this problem to a degree by writing a patch to allow the doubling to be [divided into four stages][11], meaning that we'll go from 64GB to 80GB to 96GB to 112GB to 128GB instead of going from 64GB to 128GB in one shot.  This could be improved further, but it would require deeper restructuring of the on-disk format and some careful thinking about the effects on lookup performance.

A report from [a tester who goes by "AP"][12] in July tipped us off to the need for a few further tweaks.  AP found that trying to insert 2 billion rows into a newly-created hash index was causing an error.  To address that problem, Amit modified the bucket split code to [attempt a cleanup of the old bucket just after each split][13], which greatly reduced the accumulation of overflow pages.  Just to be sure, Amit and I also [increased the maximum number of bitmap pages][14], which are used to track overflow page allocations, by a factor of four.

While there's [always more that can be done][15], I feel that my colleagues and I -- with help from others in the PostgreSQL community -- have accomplished our goal of making hash indexes into a first-class feature rather than a badly neglected half-feature.  One may well ask, however, what the use case for that feature may be.  The blog entry from Amit to which I referred (and linked) at the beginning of this post shows that even for a pgbench workload, it's possible for a hash index to outperform btree at both low and high levels of concurrency.  However, in some sense, that's really a worst case.  One of the selling points of hash indexes is that the index stores the hash value, not the actual indexed value - so I expect that the improvements will be larger for wide keys, such as UUIDs or long strings.  They will likely to do better on read-heavy workloads, as we have not optimized writes to the same degree as reads, but I would encourage anyone who is interested in this technology to try it out and post results to the mailing list (or send private email), because the real key for a feature like this is not what some developer thinks will happen in the lab but what actually does happen in the field.

In closing, I'd like to thank Jeff Janes and Jesper Pedersen for their invaluable testing work, both related to this project and in general.  Getting a project of this magnitude correct is not simple, and having persistent testers who are determined to break whatever can be broken is a great help.  Others not already mentioned who deserve credit for testing, review, and general help of various sorts include Andreas Seltenreich, Dilip Kumar, Tushar Ahuja, Álvaro Herrera, Michael Paquier, Mark Kirkwood, Tom Lane, and Kyotaro Horiguchi.  Thank you, and thanks as well to anyone whose work should have been mentioned here but which I have inadvertently omitted.

---  
via：https://rhaas.blogspot.jp/2017/09/postgresqls-hash-indexes-are-now-cool.html?showComment=1507079869582#c6521238465677174123  

作者：[作者名][a]  
译者：[译者ID](https://github.com/id)  
校对：[校对者ID](https://github.com/id)  
本文由[LCTT]（https://github.com/LCTT/TranslateProject）原创编译，[Linux中国]（https://linux.cn/）荣誉推出


[a]:http://rhaas.blogspot.jp
[1]:http://amitkapila16.blogspot.jp/2017/03/hash-indexes-are-faster-than-btree.html
[2]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=6d46f4783efe457f74816a75173eb23ed8930020
[3]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=c11453ce0aeaa377cbbcc9a3fc418acb94629330
[4]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=293e24e507838733aba4748b514536af2d39d7f2
[5]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=6977b8b7f4dfb40896ff5e2175cad7fdbda862eb
[6]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=7c75ef571579a3ad7a1d3ee909f11dba5e0b9440
[7]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=a507b86900f695aacc8d52b7d2cfcb65f58862a2
[8]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=7403561c0f6a8c62b79b6ddf0364ae6c01719068
[9]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=08bf6e529587e1e9075d013d859af2649c32a511
[10]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=e759854a09d49725a9519c48a0d71a32bab05a01
[11]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=ea69a0dead5128c421140dc53fac165ba4af8520
[12]:https://www.postgresql.org/message-id/20170704105728.mwb72jebfmok2nm2@zip.com.au
[13]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=ff98a5e1e49de061600feb6b4de5ce0a22d386af
[14]:https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=ff98a5e1e49de061600feb6b4de5ce0a22d386af
[15]:https://www.postgresql.org/message-id/CA%2BTgmoax6DhnKsuE_gzY5qkvmPEok77JAP1h8wOTbf%2Bdg2Ycrw%40mail.gmail.com