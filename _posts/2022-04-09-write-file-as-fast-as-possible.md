---
title: How to calculate top 100 from 100M ips(-)-Write File As Fast As Possible

description: how to write 100M IP addresses as fast as possible

categories:
 - How to calculate top 100 from 100M ips  
 
tags:
 - Java
 - TopK  
 - Memory Mapped File
  
---
## Background
If  we want to get the top K of 100M IP addresses,we should first mock 100M Ip addresses.
## Use Guava and ThreadPool
The first way that comes to my mind is to use multithread. So I chose **Guava** to write the file.The code is belowing:
```
    File file = new File(fileName);
    CharSink charSink = Files.asCharSink(file, Charsets.UTF_8, FileWriteMode.APPEND);
    for (int i = 0; i < 10*1000*1000; i++)
    charSink.write(CommonUtils.generateIp());
```
My current computer configuration is 8GB RAM and dual core cpu.I set the coreSize and maxCoreSize of the Threadpool to 10. So I can use 10 thread to write the file concurrencyly. The program has been running for 5 minutes and has not finished writing, I think I have to get a new solution.

## Use Memory Mapped File
The first time I knew Memory Mapped File is the advantage of **Kakfa**.Why do the kafka have a vey good write performace?Because when  the producer send the message to kafka,kafka write the message to memory without to write the message to disk directly.If the size of message increase to the size of page size,the system will flush the memory to disk automaticlly.

So I use the **MappedByteBuffer** which is offered by JDK1.8 to write file.The code is like that:
```
  RandomAccessFile file = new RandomAccessFile(otherFolder, "rw");
  MappedByteBuffer out = file.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, Integer.MAX_VALUE);
  for (int i = 0; i < 100000000; i++) {
    byte[] bytes = CommonUtils.generateIp().getBytes();
    out.put(bytes);
  }
```
It takes about 56 second to write 100M to file.

## Disadvantage and Advantage of MMAP
**Disadvantage**:As a programmer,you do not know when the system will flush the data.And if the power is off,the data in memory will be lost. In order to avoid the lost of data due to power off,it also offer method to flush the data mannually.
```
    public final MappedByteBuffer force() 
```
**Advantage**:Read and write performance is very high.Though it write to visual memory,but I monitor the memory of the program and do not find any change.

## Conclusion and Doubt
Without a doubt,the MMAP is the fastest about the question(how to write the file as fast as possible).But we should also tradeoff the availability and performace.If we want to improve the availability,we have to use the flush the data mannually which could reduce the performace.

Doubt1:I use Java to write 100M ips to the file.Could other programming language be more faster than 56 second?

Doubt2:I use single thread to implement that.Could multithread be more faster?

Doubt3:The bottleneck is io, if I use a high-speed disk, will the program become faster?

Doubt4:The MMAP use the virtual memory.The method is like that.The size is long type.Why the size must be no  greater than Integer.MAX_VALUE?
```
public abstract MappedByteBuffer map(MapMode mode,long position, long size)

size – The size of the region to be mapped; must be non-negative and no greater than Integer.MAX_VALUE
```
