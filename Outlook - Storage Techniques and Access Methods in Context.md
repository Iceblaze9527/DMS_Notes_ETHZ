---
tags: DMS
---

# Outlook: Storage Techniques and Access Methods in Context
[TOC]

## 4. Storage
### [4.1 Object Storage: Amazon S3](https://hackmd.io/rT-WrXEdS4aKEng3Um1KlQ#241-Amazon%E2%80%99s-Simple-Storage-Service-S3)
![](https://i.imgur.com/0862Wzo.png =200x)

Amazon Simple Storage Service (S3) is an object storage service in the cloud that acts as the persistent storage that is available to applications

#### Difference from Traditional Database
- Object Store: `Bucket ID - Object ID - Object`
    - partial processing of the data for specific formats (S3 Select)
    - adding using defined functions (S3 Object Lambda)
- RESTful interface: `HTTP(S) PUT/GET/DELETE`
- Update: immutable, no in-place updates
- Cache Replacement: LRU replacement
- CAP: high availability

#### Performance Implication
- High CPU overhead (because of HTTP)
- I/O is extra expensive (network bandwidth, latency, interface)

### [4.2 Snowflake](https://hackmd.io/rT-WrXEdS4aKEng3Um1KlQ#3-Cloud-Native-Databases)
![](https://i.imgur.com/sXevcKi.png =600x)

- A data warehouse specialized for analytical queries developed entirely on the cloud (cloud native)
- Separates compute (nodes running VMs with a local disk) from storage (Amazon’s S3)

#### Micro-partitions
![](https://i.imgur.com/XXtbPzI.jpg =600x)

Micro-partitions are Snowflake’s name for extents

- Size: ranges between 50 and 500 MB (before compression, the data is always compressed when in S3), to reduce http requests
- Each micro partition has metadata describing what is inside
- The metadata can be read without reading the whole micro-partition, and is used to read just the part of the micro-partition that is relevant (pre-computed stats)
- Columnar Storage

#### Tricks Used
- **Horizontal partitioning of the tables:** allows to read the table in parallel, to put different parts of the table in different processing nodes, and – if organized accordingly- allows to read only the needed tuples instead of all the table
- **Columnar format:** the preferred storage format for analytics, improves instruction cache locality (same data types), enables vectorized processing, facilitates projection operations (SQL), allows to process only the part of the table that is relevant
- **Storage-level processing:** to read only the part of the file that is needed (make sequentially going over memory possible)

#### Pruning
Snowflake does not use indexes
- Indexes require a lot of space (even bigger than data itself!)
- Indexes induce random accesses (very bad for slow storage like S3)
- Indexes need to be maintained and selected correctly

Instead, it uses the metadata to store precomputed information that allows to filter micro-partitions (min/max, #distinct values,#nulls, bloom filters, etc.)
- The metadata is much smaller than an index and easier to load than a whole index 
- By splitting table in potentially many micro-partitions, it can significantly optimize the data movement to and from storage

#### Writing to Disk
Snowflake uses S3's immutability to implement snapshots of the data (like shadow paging):
- When a micro-partition is modified, a new file is written
- The old micro-partition can be kept or discarded 

Allows time travel (read the data in the past up to 90 days) and provides fault-tolerance (the old data can be recovered from the old micro-partitions)

## 5. Access Methods 1: Denormalized Tables
### 5.1 Motivation
- Joins are expensive
- It makes sense to store table together if tables are more often accessed together than on their own
- Reduces I/O, save space

### 5.2 Implementation: Clustered Tables (Oracle)
Some systems allow to cluster tables into the same segment so that blocks contain data form both tables (indexed by the common attributes/keys)

![](https://hackmd.io/_uploads/rkoBIH64j.png =600x)

### 5.3 Comments
- It is a form of materialized view with a join already performed
- Updates can become expensive

## 6. Access Methods 2: Log Structured Databases
### 6.1 Log-structured File
- Instead of storing tuples and modifying them as needed, keep a record of how the data was modified (a log)
- New entries are appended at the end of the file
- Index the tuples and their modifications (entry could contain pointer to previous entry referring to that tuple)

![](https://hackmd.io/_uploads/H1tUPBaNo.png =600x)

### 6.2 Optimization
![](https://hackmd.io/_uploads/ryuW2SaNj.png =200x)

- Periodically compact the log: Remove the history and add one entry for every tuple

### 6.3 Comments
- In cloud storage, file storage is typically append only
- It minimizes the cost of making the data persistent (very expensive for OLTP databases with many transactions)
- Not for heavy query analytics but very good for in memory transactional workloads
- Increasingly being used in a number of databases, especially cloud databases

## 7. Access Methods 3: No Indices
### 7.1 Micro-partitions (Snowflake)
#### Motivation
Indices are expensive, so it depends whether to use them

#### Pruning based on metadata
- The header for a micro-partition contains information about the data.
- For range search, for example, if the header contains the min and max in the data, we can decide we do not need that micro-partition simply by looking at the header
