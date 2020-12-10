---
layout:     post

title:      "Elastic Search max open files"
subtitle:   "Increase max open files in Elastic Search (OSX) "
author: Jamie Craane
date:       2016-12-02
description: "In this post I am going to show how to increase the max open files for Elastic Search on OSX."
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: true
showtoc: false
tags:
- ElasticSearch

URL: "/2016/02/22/max_open_files_es/"
---

Elastic Search, Logstash and Kibana (ELK) is an end-to-end stack which provides realtime analytics for almost any type of structured or unstructured data.

When importing large amounts of data using Logstash to Elastic Search (ES), the chances are that ES hit the limits of the maximum files it can open. This limit is seen as an error in the ES logs with the following description: (Too many open files)

To deal with this you can increase the maximum files ES (or any process) may open using the following steps:

1. First start ES with the following option: ./elasticsearch -Des.max-open-files. This wil show the maximum number of files ES is allowed to open, for example: 

```text
[2016-02-22 06:44:09,558][INFO ][bootstrap] max_open_files [10240]
```   

2. Now execute the following commands to increase the maximum number of files a process may open:

```text
- sudo sysctl -w kern.maxfiles=32000 
- sudo sysctl -w kern.maxfilesperproc=32000
```

3. Execute the following commands to set the file limit for the terminal process (this is the terminal window to launch ES in)

```text
- ulimit -Sn 32000
- ulimit -Hn 32000
```

For ES 1.7:
Start Elastic Search with the following

```text
command: ./elasticsearch -Des.max-open-files -XX:-MaxFDLimit=true
```

For ES 2.2
Execute the following commands:

```text
export ES_JAVA_OPTS=-XX:-MaxFDLimit (this increases the maximum files the JVM is allowed to open by default, see JVM configuration for more information)

and then start ES with the following command: 
./elasticsearch -Des.max-open-files (max_open_files should be 32000 now)
```

**Heap sizes**

You may also need to increase the heap size of both Logstash and Elastic Search.
To increase the heap of LogStash execute the following command before launching Logstash: export LS_HEAP_SIZE=2g
To increase the heap of Elastic Search execute the following command before launching ES: export ES_HEAP_SIZE=8g
When importing large files using Logstash, it may benefit to increase the number of workers to speed up the importing process. The default is 1. See the following example of the elasticsearch output plugin in Logstash:

```json
elasticsearch {
    action => "index"
    hosts => ["localhost"]
    index => "logstash-%{+YYYY.MM.dd}"
    workers => 4
    flush_size => 1000
}
```
