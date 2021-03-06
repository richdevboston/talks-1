# Spying on Hadoop with strace

by Julia Evans <br>
[`twitter.com/b0rk`][twitter]  <br>
[`github.com/jvns`][github]  <br>
[`jvns.ca`][website]  <br>

[github]: https://github.com/jvns
[twitter]:  https://twitter.com/b0rk
[website]: http://jvns.ca


# strace?!

HOW TO SPY

# Hadoop?!

A distributed filesystem + tools for querying + transforming data.

# all you need to know about Hadoop

HDFS is a distributed filesystem

# Let's use hadoop!

```
$ hadoop fs -ls /
wikipedia.csv
````

# Let's use hadoop!

DEMO DEMO DEMO

# Learning about HDFS

+ read documentation
+ Google errors
+ search through logs
+ ask your friends


# strace <br> = <br> wizardry

# System calls: <br> a OS's API

# System calls

+ open
+ write
+ sendto/recvfrom

# How to strace

```
$ strace hadoop fs -ls /penguin
```

# How to strace

DEMO DEMO

</section>
<section data-background="strace-garbage.png">

# open

<pre>
strace -e open hadoop fs -ls /panda
open("<b>/etc/hadoop/mapred-site.xml</b>", O_RDONLY) = 274
open("/etc/hadoop/yarn-site.xml", O_RDONLY) = 274
open("/etc/hadoop/hdfs-site.xml", O_RDONLY) = 274
</pre>

# sendto

<pre>
$ strace snakebite ls /unicorn
connect(8, {sa_family=AF_INET, sin_port=htons(<b>8200</b>),
    sin_addr=inet_addr("<b>10.147.177.170</b>")}, 16) = 0
sendto(8,
    "\nB\n5\n3\n(BP-1019336183-10.165.43.39-1400088409498\20\211\200\200\200\4\30\361\7\22\tsnakebite\20\0\30\200\200\200@",
    75, 0, NULL, 0) = 75
</pre>
```

# recvfrom

```
strace hadoop fs -get /wikipedia.csv
recvfrom(8, "ot, it's a painting. Thomas Graeme apparently lived in
the mid-18th century, according to the [[Graeme Park]] article. The
rationale also says that this image is "used on the biography
page about him by USHistory.org of Graeme Park." I cannot quite
figure out what this means, but I am guessing that it means the
uploader took this image from a page hosted on USHistory.org. A
painting of a man who lived in the mid-18th century is likely to be
the public domain, as claimed, but we have no good source", 512, 0,
NULL, NULL) = 512
```

# write

+ spy on log files!

# don't strace in production

<img src="warning.png" width=300px>

# Time for strace!

# Toy Hadoop cluster

* 1 master machine (namenode) <b>hadoop-m-0</b>
* 2 worker machines (datanode) <b>hadoop-w-0, hadoop-w-1</b>
* 20 GB of disk space
* one 14 GB file `/wikipedia.csv`

<pre class="medium">
$ snakebite ls -h /
14.1G        /wikipedia.csv
</pre>

# How HDFS works

* The **namenode** knows where all the files are
    * hadoop-m-0
* The **datanodes** store the files
    * hadoop-w-0, hadoop-w-1

# How HDFS works

* Files are split into **blocks**
* Blocks for `/wikipedia.csv`

```
     Bytes |   Block ID | # Locations |       Hostnames
 134217728 | 1073742025 |           1 |      hadoop-w-1
 134217728 | 1073742026 |           1 |      hadoop-w-1
 134217728 | 1073742027 |           1 |      hadoop-w-0
 134217728 | 1073742028 |           1 |      hadoop-w-1
 134217728 | 1073742029 |           1 |      hadoop-w-0
 134217728 | 1073742030 |           1 |      hadoop-w-1
 ....
 134217728 | 1073742136 |           1 |      hadoop-w-0
  66783720 | 1073742137 |           1 |      hadoop-w-1
```


# How to read a file

* Ask the namenode where the blocks for that are 
* Ask the data nodes for each block

# Let's strace it!

<pre class="medium">
snakebite cat /wikipedia.csv | head
strace -e recvfrom,sendto,connect \
     snakebite cat /wikipedia.csv | head
</pre>

# 1: Where are the blocks?

(the namenode is at 10.240.98.73)


<pre>
connect(4, {sa_family=AF_INET, sin_port=htons(<b>8020</b>),
        sin_addr=inet_addr("<b>10.240.98.73</b>")}, 16)
sendto(4, "\n\v<b>getFileInfo</b>\22.org.apache.hadoop.hdfs.protocol.ClientProtocol\30\1", 63, 0, NULL, 0) = 63
sendto(4, "\n\16/<b>wikipedia.csv</b>", 16, 0, NULL, 0) = 1
sendto(4, "\n\21<b>getBlockLocations</b>\22.org.apache.hadoop.hdfs.protocol.ClientProtocol\30\1", 69, 0, NULL, 0) = 69
sendto(4, "\n\16/<b>wikipedia.csv</b>\20\0\30\350\223\354\2378", 24, 0, NULL, 0) = 24
</pre>

# 1: Where are the blocks?

<pre>
recvfrom(4, "\255\202\2\n\251\202\2\10\350\223\354\2378\22\233\2\n7\n'BP-572418726-10.240.98.73-1417975119036\20\311\201\200\200\4\30\261\t \200
\200\200@\20\0\32\243\1\nk\n\01610.240.146.168\22%<b>hadoop-w-1</b>.c.stracing-hadoop.internal\32$358043f6-051d-4030-ba9b-3cd0ec283f6b \332\206\3(\233\
207\0030\344\206\0038\0\20\200\300\323\356&\30\200\240\354\372\32 \200\200\216\344\4(\200\240\354\372\0320\233\327\304\346\242)8\1B\r/default-ra
ckP\0X\0`\0 \0*\10\n\0\22\0\32\0\"\0002\1\0008\1B'DS-3fa133e4-2b17-4ed1-adca-fed4767a6e6f\22\236\2\n7\n'BP-572418726-10.240.98.73-1417975119036\
20\312\201\200\200\4\30\262\t \200\200\200@\20\200\200\200@\32\243\1\nk\n\01610.240.146.168\22%<b>hadoop-w-1</b>.c.stracing-hadoop.internal\32$358043f6
-051d-4030-ba9b-3cd0ec283f6b \332\206\3(\233\207\0030\344\206\0038\0\20\200\300\323\356&\30\200\240\354\372\32 \200\200\216\344\4(\200\240\354\3
72\0320\233\327\304\346\242)8\1B\r/default-rackP\0X\0`\0 \0*\10\n\0\22\0\32\0\"\0002\1\0008\1B'DS-3fa133e4-2b17-4ed1-adca-fed4767a6e6f\22\237\2\
n7\n'BP-572418726-10.240.98.73-1417975119036\20\313\201\200\200\4\30\263\t \200\200\200@\20\200\200\200\200\1\32\243\1\nk\n\01610.240.109.224\22
%<b>hadoop-w-0</b>.c.stracing-hadoop.internal\32$bd6125d3-60ea-4c22-9634-4f6f352cfa3e \332\206\3(\233\207\0030\344\206\0038\0\20\200\300\323\356&\30\20
0\200\342\335\35 \200\240\202\201\2(\200\200\342\335\0350\271\353\304\346\242)8\1B\r/default-rackP\0X\0`\0 \0*\10\n\0\22\0\32\0\"\0002\1\0008\1B
'DS-c5ef58ca-95c4-454d-adf4-7ceaf632c035\22\237\2\n7\n'BP-572418726-10.240.98.73-1417975119036\20\314\201\200\200\4\30\264\t \200\200\200@\20\20
0\200\200\300\1\32\243\1\nk\n\01610.240.146.168\22%<b>hadoop-w-1</b>.c.stracing-hadoop.inte"..., 33072, 0, NULL, NULL) = 32737
</pre>

# 1: Where are the blocks?

* Blocks for `/wikipedia.csv`

<pre>
     Bytes |   Block ID | # Locations |       Hostnames
 134217728 | 1073742025 |           1 |      <b>hadoop-w-1</b>
 134217728 | 1073742026 |           1 |      <b>hadoop-w-1</b>
 134217728 | 1073742027 |           1 |      <b>hadoop-w-0</b>
 134217728 | 1073742028 |           1 |      hadoop-w-1
 134217728 | 1073742029 |           1 |      hadoop-w-0
 134217728 | 1073742030 |           1 |      hadoop-w-1
 ....
 134217728 | 1073742136 |           1 |      hadoop-w-0
  66783720 | 1073742137 |           1 |      hadoop-w-1
</pre>


# 2: Getting a block

(<b>10.240.146.168</b> is hadoop-w-1)
<pre>
connect(5, {sa_family=AF_INET, sin_port=htons(<b>50010</b>),
        sin_addr=inet_addr("<b>10.240.146.168</b>")}, 16) = 0
sendto(5, "\nK\n>\n2\n'BP-572418726-10.240.98.73-1417975119036\20\311\201\200\200\4\30\261\t\22\10\n\0\22\0\32\0\"\0\22\tsnakebite\20\0\30\200\2
00\200@", 84, 0, NULL, 0) = 84
</pre>

# 2: Getting a block


<pre>
OpTransferBlockProto
header {
  baseHeader {
    block {
      poolId: "BP-572418726-10.240.98.73-1417975119036"
      blockId: <b>1073742025</b>
      generationStamp: 1201
    }
  }
  clientName: "snakebite"
}
</pre>

# 2: Getting a block

* Blocks for `/wikipedia.csv`

<pre>
     Bytes |   Block ID | # Locations |       Hostnames
 <b>134217728</b> | <b>1073742025</b> |           1 |      hadoop-w-1
 134217728 | 1073742026 |           1 |      hadoop-w-1
 134217728 | 1073742027 |           1 |      hadoop-w-0
 134217728 | 1073742028 |           1 |      hadoop-w-1
 134217728 | 1073742029 |           1 |      hadoop-w-0
 134217728 | 1073742030 |           1 |      hadoop-w-1
 ....
 134217728 | 1073742136 |           1 |      hadoop-w-0
  66783720 | 1073742137 |           1 |      hadoop-w-1
</pre>


# 2: Getting a block

<pre>
recvfrom(5, "<b>title,id,language</b>,wp_namespace,is_redirect,revision_id,contributor_ip,contributor_id,contributor_username,timestamp,is_minor,is_bot
,reversion_id,comment,num_characters\nIvan Tyrrell,6126919,,0,true,264190184,,37486,Oddharmonic,1231992299,,,,\"Added defaultsort tag, categorie
s.\",2989\nInazuma Raigor\305\215,9124432,,0,,224477516,,2995750,ACSE,1215564370,,,,/* Top division record */ rm jawp reference,5557\nJeb Bush,1
89322,,0,,299771363,66.119.31.10,,,1246484846,,,,/* See also */,43680\nTalk:Goranboy (city),18941870,,1,,", 512, 0, NULL, NULL) = 512
</pre>


# 3: Where is the block?

It's on hadoop-w-1!

<pre class="big">
$ ssh hadoop-w-1
</pre class="big">

# 3: Where is the block?

<pre>
$ cd /hadoop/dfs/data/current/BP-572418726-10.240.98.73-1417975119036/current/finalized
$ ls -l blk_<b>1073742025</b>
-rw-r--r-- 1 hadoop hadoop <b>134217728</b> Dec  8 02:08 blk_1073742025
</pre>

# 3: Where is the block?

<pre>
$ head blk_<b>1073742025</b>
<b>title,id,language</b>,wp_namespace,is_redirect,revision_id,contributor_ip,contributor_id,contributor_username,timestamp,is_minor,is_bot,reversion_id,comment,num_characters
Ivan Tyrrell,6126919,,0,true,264190184,,37486,Oddharmonic,1231992299,,,,"Added defaultsort tag, categories.",2989
Inazuma Raigorō,9124432,,0,,224477516,,2995750,ACSE,1215564370,,,,/* Top division record */ rm jawp reference,5557
Jeb Bush,189322,,0,,299771363,66.119.31.10,,,1246484846,,,,/* See also */,43680
Talk:Goranboy (city),18941870,,1,,233033452,,627032,OOODDD,1219200113,,,,talk page tag  using [[Project:AutoWikiBrowser|AWB]],52
</pre>

# strace the namenode!

# strace the datanode!

# The moral

strace is a fun way to learn about systems!

# don't strace in production

<img src="warning.png" width=300px>

# Learning about internals = great

# Internals

* let you manage smarter systems
* Now you know why you shouldn't set your block size to 4KB!

# Further directions:

* strace the scheduler!
* see how map/reduce jobs get submitted!
* strace a map job while it's running!

# Questions?

Julia Evans <br>
[`twitter.com/b0rk`][twitter]  <br>
[`github.com/jvns`][github]  <br>
[`jvns.ca`][website]  <br>

[github]: https://github.com/jvns
[twitter]:  https://twitter.com/b0rk
[website]: http://jvns.ca


