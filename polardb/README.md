# Articles
1. [Official architecture for PolarDB for PostgreSQL](https://github.com/ApsaraDB/PolarDB-for-PostgreSQL/blob/main/doc/PolarDB-EN/Architecture.md)
2. [Architecture Introduction in 2018](https://www.alibabacloud.com/blog/the-evolution-of-alibaba-clouds-relational-database-services-architecture--polardb_579134)
3. [Zhihu answer from MingGao, the founder of PolarDB](https://www.zhihu.com/question/63987114/answer/244520478)

# Papers
1. [PolarDB Serverless: A Cloud Native Database for Disaggregated Data Centers SIGMOD2021](http://www.cs.utah.edu/~lifeifei/papers/polardbserverless-sigmod21.pdf)
2. [PolarDB meets computational storage: Efficiently support analytical workloads in cloud-native relational database Fast2020](https://www.usenix.org/system/files/fast20-cao_wei.pdf)
3. [PolarFS: an ultra-low latency and failure resilient distributed file system for shared storage cloud database](http://www.vldb.org/pvldb/vol11/p1849-cao.pdf)


# Archi
Standard PolarDB (compared to PolarDB serverless).
1. one write(RW node) multiple read(RO node). 
2. shared disks based on PolarFS.
3. no sharding, but buffer pool of each RW/RO nodes may differs.

Write path
1. RW node writes redo log to PolarFS(indicates commit succees)
2. RW node sends redo RWlsn to RO nodes.
3. RO node fetches and replay(replay could be done at read time asynchrously) redo log from PolarFS with lsns in range [local ROlsn, RWlsn]
4. RO node sends a ROlsn back to RW node, ROlsn is the lsn which the RO node has replayed.
5. RO node could support SI read elder than ROlsn.
6. RW node receives all the ROlsn from RO nodes, and purge redo log and flush dirty pages before the min{ROlsns}
7. RO node whose ROlsn is too far behind by RWlsn, it will be killed.

The keypoints are:
1. RW node cannot flush dirty pages aggresively, or RO nodes have a risk of read future pages(replayed to lsn100, but read a page of lsn 105, it's not consistent since lsn101-lsn104 is missing)
2. RO node needs to replay the redo log from pages in PolarFS every time, either when loaded into the buffer pool or when read by a user. PolarDB-PG let user session to do the replay work. This offload behavior is claimed as 30 times faster than Aurora replay latency. But it add overhead for user session, especially multiple user sessions read the same page, one session do the replay work and other sessions hung.

Does RO node need to fetch all the redo logs from PolarFS?
I think it's not needed. Since different from MySQL/PG, the standby(RO) node needs to replay all the redo logs and generate the new physical pages, PolarDB uses shared disk and RO nodes rely on RW node to flush the dirty pages to disks. RO nodes only need to consider the following case:
1. The redo log contains a page modification on exising buffer poll of RO node, then the
