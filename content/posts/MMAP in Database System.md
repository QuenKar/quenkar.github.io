---
title: "Note: Are You Sure You Want to Use MMAP in Your Database Management System?"
date: 2023-11-02T19:57:00+08:00
draft: false
categories: ["database","论文阅读笔记"]
---

> Are You Sure You Want to Use MMAP in Your Database Management System?
> https://db.cs.cmu.edu/mmap-cidr2022/

## What's MMAP (💩)？

MMAP is a system call that maps a file into memory. It is a common technique for database systems to access data.

### MMAP OverView

![mmap overview.png](/imgs/mmap_overview.png)

Figure 1 shows a step-by-step overview of how to access a file
(“cidr.db”) with mmap. 
1. A program calls mmap and receives a
pointer to the memory-mapped file contents.
2. The OS reserves
part of the program’s virtual address space but does not load any
part of the file. 
3. The program accesses the file’s contents using the
pointer. 
4. The OS attempts to retrieve the page. 
5. Since no valid
mapping exists for the specified virtual address, the OS triggers a
page fault to load the referenced part of the file from secondary
storage into a physical memory page. 
6. The OS adds an entry to
the page table that maps the virtual address to the new physical
address. 
7. The initiating CPU core also caches this entry in its local
translation lookaside buffer (TLB) to accelerate future accesses.

### MMAP Benifits
 1. 方便实现内存管理
 2. 不会显示调用系统调用(i.e., read/write)
 3. 避免了在用户空间copy buffer

### Problems with MMAP
1. 事务安全
2. I/O 延迟（an I/O request that does complete, or that takes excessive time to complete）
3. 错误处理
4. Performance

![mmap_performance](/imgs/mmap_performance.png)

可以注意到mmap虽然有上面的那些优点，但是在性能上并没有和文件I/O有明显的优势

### When you should not use mmap in your DBMS:
-  You need to perform updates in a transactionally safe fashion.
- You want to handle page faults without blocking on slow I/O
or need explicit control over what data is in memory.
- You care about error handling and need to return correct results.
- You require high throughput on fast persistent storage devices.
### When you should maybe use mmap in your DBMS:
- Your working set (or the entire database) fits in memory and
the workload is read-only.
- You need to rush a product to the market and do not care about
data consistency or long-term engineering headaches.
- Otherwise, never.

## Thoughts
对于我目前所做的时序数据库(`GreptimeDB`)方面的一些工作来说，虽然时序数据库没有事务安全的要求，但是时序数据库的数据量一般都是很大的，一般都是写多读少，用的是append-only的，而且数据的存储也可能在Amazon S3，Azure Blob Storage，GCS这些地方，总的来说，感觉mmap不太适合时序数据库场景的。（By the way, InfluxDB 2020年开始不用mmap了

## Reference

- https://questdb.io/blog/2020/08/19/memory-mapping-deep-dive/
