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
If we want to get the top K of 100M IP addresses, we should first mock 100M IP addresses.
## ~~Use Guava and ThreadPool~~
The first way that comes to my mind is to use multithreaded. So I chose **Guava** to write the file. The code is bellowing:
```
    File file = new File(fileName);
    CharSink charSink = Files.asCharSink(file, Charsets.UTF_8, FileWriteMode.APPEND);
    for (int i = 0; i < 10*1000*1000; i++)
    charSink.write(CommonUtils.generateIp());
```
My current computer configuration is 8 GB RAM and dual-core CPU.I set the coreSize and maxCoreSize of the Thread pool to 10. So I can use 10 thread to write the file concurrently. The program has been running for 5 minutes and has not finished writing, I think I have to get a new solution.

<font color=red>Never use multithread to write the same file!!! The only way to write the file using multithread is to write different section of the files between different thread.If your multithread code is like above,it is no using and foolish. Because the file descriptor depends on the system call. When multithread writes the same file, the file system just send a notification to system and wait the system to copy data. So the single thread and multithread have no difference in writing. </font>

## Use Memory Mapped File
The first time I knew Memory Mapped File is the advantage of **Kafka**. Why do the Kafka have a very good write performance? Because when the producer send the message to Kafka, Kafka write the message to memory without to write the message to disk directly. If the size of message increase to the size of page size,the system will flush the memory to disk automatically.

So I use the **MappedByteBuffer** which is offered by JDK1.8 to write file. The code is like that:
```
  RandomAccessFile file = new RandomAccessFile(otherFolder, "rw");
  MappedByteBuffer out = file.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, Integer.MAX_VALUE);
  for (int i = 0; i < 100000000; i++) {
    byte[] bytes = CommonUtils.generateIp().getBytes();
    out.put(bytes);
  }
```
It takes about 56 second to write 100M to file.

 ## How to optimize the problem

 Our goal is to write the data to file. Since the process is determined, should we reduce the size of data?

### Compression ###
Since our data is byte array. So we could compress the data. And we could write the data after compression

### Optimize the data  ###
The IP address is the 32-unsigned integer. So we could use 4 bytes to store one IP address. For example,


| Integer | IP address | 
|-------|--------|
|  0 | 0.0.0.0 |
|  128   |   0.0.0.128 |
|  256   |   0.0.1.0 |
|  0x7fffffff |    127.255.255.255 |


The convert code is bellowing:
```
 public static String ipIntToString(int num) {
        StringBuilder stringBuilder=new StringBuilder();
        stringBuilder.append((num >> 24) & 0xff);
        stringBuilder.append(".");
        stringBuilder.append((num >> 16) & 0xff);
        stringBuilder.append(".");
        stringBuilder.append((num >> 8) & 0xff);
        stringBuilder.append(".");
        stringBuilder.append(num & 0xff) ;
        return stringBuilder.toString();
    }
```
Since the relationship between 32-unsigned integer and the IP address is one-to-one. So we could use 4 bytes to store one IP address. In order to reduce the IO, we generate 100M IP address in memory and write it to file one time.  The total size of the 100M IP in memory is
```
4*100M=400M
```
So the memory allocated is not very much. We could write it. The final code is bellowing:
```
FileOutputStream stream = new FileOutputStream(otherFolder);
int len = 100 * 1000 * 1000 * 4;
byte[] bytes = new byte[len];
for (int i = 0; i < 100 * 1000 * 1000; i++) {
          bytes[i] = (byte) i;
}
stream.write(bytes);
```
It takes less than 5 second to write the data in to the file. And the size of the file is about 380 MB which is one-fifth size of the writing string.  


## Disadvantage and Advantage of MMAP
**Disadvantage**:As a programmer, you do not know when the system will flush the data. And if the power is off, the data in memory will be lost. In order to avoid the lost of data due to power off, it also offers method to flush the data manually.
```
    public final MappedByteBuffer force() 
```
**Advantage**:Read and write performance is very high. Though it writes to visual memory, but I monitor the memory of the program and do not find any change.

## Conclusion and Doubt
If we want to write one file as fast as possible, we should reduce the size of the data as much as possible. If you want to write the file by multithreaded, every thread should have the different section to write. Without a doubt, the MMAP is a good solution to write a large file (how to write the file as fast as possible). But we should also trade off the availability and performance. If we want to improve the availability, we have to use the flush the data manually which could reduce the performance.


Doubt1:The bottleneck is io, if I use a high-speed disk, will the program become faster?

Answer:Yes

Doubt2:The MMAP use the virtual memory. The method is like that. The size is long type. Why the size must be no greater than Integer.MAX_VALUE?
```
public abstract MappedByteBuffer map(MapMode mode,long position, long size)

size – The size of the region to be mapped; must be non-negative and no greater than Integer.MAX_VALUE
```
