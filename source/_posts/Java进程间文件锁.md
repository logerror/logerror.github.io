---
title: Java进程间共享锁
categories: Java
tags:
  - lock file
description: Java进程间共享锁
abbrlink: 2202924434
date: 2017-08-12 11:47:44
---

****背景****     
业务系统运行时产生了大量的小文件（大小在10kb左右，包含一些表单的内容，rest接口的访问记录），考虑到将这些内容存储在数据库中对数据库压力较大，因此转为文件存储，但由于restful接口访问频繁，导致产生了大量的小文件，最终导致服务器inode占用率过高，触发告警


****原因****    

什么是inode?
inode包含文件的元信息，具体来说有以下内容：

　　* 文件的字节数

　　* 文件拥有者的User ID

　　* 文件的Group ID

　　* 文件的读、写、执行权限

　　* 文件的时间戳，共有三个：ctime指inode上一次变动的时间，mtime指文件内容上一次变动的时间，atime指文件上一次打开的时间。

　　* 链接数，即有多少文件名指向这个inode

　　* 文件数据block的位置

大量小文件的产生导致服务器inode占用过高，触发告警，如果inode继续增加可能会导致无法创建文件。


****解决思路****

将多个小文件合并为一个大文件处理，思路有很多：



1. 可以合并为一个大的文本文件，每次记录内容的偏移量，读的时候根据起始位置和偏移量来获取内容。
2. 仍保持单个文件的独立性，将多个文件合并为一个大的压缩文件。


经过尝试发现，方案1可能会导致读写错误，具体表现为文本不但会增加也可能会删除，删除后要更新所有文件的起始值和偏移值，这反而加重了数据库的负担，而且偏移量一旦计算错误将会导致所有的文件内容读到的结果都是错误的。

方案二保持了文件的独立性，只需记录文件路径即可，即便某个文件路径错误也不影响其他文件的读取，同理，删除也不会影响其他的文件读取，唯一要解决的便是并发环境即分布式环境的问题（同时读写文件），经测试和运行满足生产环境需求，且由于是一个压缩文件，inode占用率不会飙升，因此采用了方案二，并已生产部署。


在此不详细讨论如何生成并读写压缩文件（将产生的多个文件放入压缩包中即可），着重说明如何保证同一时刻一个压缩文件不能被同时写入。

****详细实现****    
	
   Java 提供了文件锁FileLock类，利用这个类可以控制不同程序(JVM)对同一文件的并发访问，实现进程间文件同步操作。FileLock是java 1.4 版本后出现的一个类，它可以通过对一个可写文件(w)加锁，保证同时只有一个进程可以拿到文件的锁，这个进程从而可以对文件做访问；而其它拿不到锁的进程要么选择被挂起等待，要么选择去做一些其它的事情， 这样的机制保证了众进程可以顺序访问该文件。也可以看出，能够利用文件锁的这种性质，在一些场景下，虽然我们不需要操作某个文件， 但也可以通过 FileLock 来进行并发控制，保证进程的顺序执行，避免数据错误。 


    
	1.保证获取到的文件对象唯一，每次写入动作时取到的文件是同一个


		private static synchronized File getInstance(String path) throws ZipException {
			if (map.containsKey(path)) {
				return map.get(path);
			} else {
				CompressedFile compressedFile = new CompressedFile(path);
				zipMap.put(path, compressedFile);
				return compressedFile;
			}
		}
  2.使用FileLock锁确保同一时刻只有一个进程可以写文件,使用try with resource自动关闭文件流   

        public static String writeZipContent(String filePath, String subPath, InputStream fis) throws Exception {
			File zipFile = getInstance(filePath);
			synchronized (zipFile) {
				File f = new File(filePath);
				if (!f.getParentFile().exists()) {
					f.getParentFile().mkdirs();
				}
				try (RandomAccessFile randomAccessFile = new RandomAccessFile(new File(zipFilePath + ".lock"), "rws"); FileChannel channel = randomAccessFile.getChannel()) {
					FileLock fileLock;
					do {
						fileLock = channel.lock();
					} while (fileLock == null || !fileLock.isValid());
	
					//do something
	
					return "path";
				} catch (Exception ex) {
					logger.error(ex.getMessage(), ex);
					throw ex;
				}
			}
    }



****补充说明****   
在java.io.RandomAccessFile类的open方法，提供了参数实现独占的方式打开文件：

        RandomAccessFile raf = new RandomAccessFile(file, "rws");

其中的“rws”参数，rw代表读取和写入，s代表了同步方式，也就是同步锁。







​ 
​