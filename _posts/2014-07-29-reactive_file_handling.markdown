---
layout: post
title:  "Reactive file handling using Akka and inotify"
date:   2014-07-29 03:00:00
categories: reactive
---

At Glassbeam, everyday we deal with terabytes of data. As part of SCALAR platform, we have to ingest many files and parse them at real time. The files arrive asynchronously and fast. These files are then picked up for parsing in the order in which they are arriving. For such a demanding requirement, we had to design a system which should not only be concurrent and asynchronous but also scalable. 

Our customers can send files either through FTP/SFTP/SCP, upload files through UI and so on. We needed a system which could trigger events as and when files arrive. The file arrival event is then trapped to make some entries in our database for book keeping purposes and further passed on to parsing engine. Our choice was Akka which scales well for this scenario and since we are a predominantly a Scala shop, Akka fits well into the ecosystem. We developed a module and internally we call it "Watcher" using Akka and Linux inotify. There were several challenges implementing such a system although it looks very simple on a first glance. In the next few sections, some of the pitfalls we faced are discussed.

<h4>Oracle JDK7 WatchService API</h4>

The first natural choice to trap file events was to use JDK7 WatchService API [1]. WatchService works by registering a file path on which we expect files to arrive. Once the files start appearing on this path, the API will notify for the client code to take action. Internally, this API uses  Linux inotify library [2] which is the notification system at the filesystem level and is specific to the platform in question. This wasn't apparent on a first glance. In the dev environement everything was working as expected, however on AWS, the notification wasn't triggering no matter what.

Interestingly, on a closer look it became clear that the Amazon AMI image we were using wasn't supporting inotify by default. We tried a lot of different AMIs and finally fixated on CentOS image which has inotify.

Just when things were going right...

<h4>WatchService API's MODIFY event</h4>

For a lot of practical purposes, WatchService's ENTRY_MODIFY event [3] is sufficient for getting a file event. But just getting a file event isn't enough. The file event should trigger when the file has been completely copied to the watched path. Otherwise, the file may be sent to parsing whilst the file is being copied from the customer end. This is not acceptable and will lead to race conditions. This scenario is especially valid when the files are sent via FTP/SFTP/SCP. When the file is sent via FTP/SFTP/SCP, the files are copied in chunks and WatchService API will treat each chunk as a file and hence trigger as many MODIFY events as the number of chunks. This is the drawback of WatchService API as it cannot let the client code know when the file got completely copied.

The reason that was citied in some of the forums for this seeming limitation is that Oracle JDK aims to achieve portability by letting go some of the platform specific dependency. Thus events such as CLOSE_WRITE which clearly marks the end of file even though the underlying protocol used may be FTP/SFTP/SCP. Since CLOSE_WRITE event was a vital event, we decided to let go of JDK WatchService API altogether. Instead, we adapted an open source project aimed specifically to address this problem and exposes all the native inotify's events including CLOSE_WRITE.

We have open sourced this "Watcher" module <a href="https://github.com/glassbeam/watcher">https://github.com/glassbeam/watcher</a> under GPL v3 license.

<h4>Conclusion</h4>

Although, APIs such as WatchService are designed to be portable, it doesn't serve for a lot of scenarios. The API must have had options to enable some of the inotify events thereby using the same API at the discretion of the user. Akka helped to scale and react to events asynchronously and therefore we gained a huge profit by not spinning threads ourselves.

<p>
[1]: <a href="http://docs.oracle.com/javase/7/docs/api/java/nio/file/WatchService.html">http://docs.oracle.com/javase/7/docs/api/java/nio/file/WatchService.html</a>
</p>
<p>
[2]: <a href="http://man7.org/linux/man-pages/man7/inotify.7.html">http://man7.org/linux/man-pages/man7/inotify.7.html</a>
</p>
<p>
[3]: <a href="http://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardWatchEventKinds.html#ENTRY_MODIFY">http://docs.oracle.com/javase/7/docs/api/java/nio/file/StandardWatchEventKinds.html#ENTRY_MODIFY</a>
</p>