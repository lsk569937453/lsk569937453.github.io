---
title: 多线程文件写入

description: how to write file in multiple thread
date: 2024-05-11T07:07:07+01:00

categories:
 - rust 
 - tool
 - multi thread
 - file
 
tags:
 - multi thread
 - http
 - file
 - rust 
---

# 多线程文件写入
鉴于2年前的一次讨论[如何快速向文件中写入 1 亿个 ip](https://v2ex.com/t/845892#reply66),让我以为多线程写文件没什么用。直到昨天晚上有新的讨论，如何快速的下载25个G的文件。其中的思路就是多线程下载，然后合并。然而有个群友提出了一个新的思路，就是多线程下载后直接写入同一个文件，每一个线程都是持有的一个文件句柄，然后分段写入。
具体源代码如下：
```
async fn test1() -> Result<(), anyhow::Error> {
    let mut set = JoinSet::new();
    let byte_array: Vec<u8> = (0..=1024).map(|x| x as u8).collect();

    for start in 0..10000 {
        let cloned_bytes = byte_array.clone();
        set.spawn(async move {
            write_file(start, cloned_bytes).await;
        });
    }
    while let Some(res) = set.join_next().await {
        let out = res?;
    }
    Ok(())
}
async fn write_file(start: i32, byte_array: Vec<u8>) -> Result<(), anyhow::Error> {
    let mut file = tokio::fs::OpenOptions::new()
        .create(true)
        .write(true)
        .open("test.bin")
        .await?;
    let current = start * byte_array.len() as i32;
    file.seek(SeekFrom::Start(current as u64)).await;
    file.write_all(&byte_array).await;
    Ok(())
}
```
这样的话就省去了一个合并文件的过程，还是能节省不少时间的。
