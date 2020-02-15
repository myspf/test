# DolphinDB Go API example

### 1. 概述
目前有5个Go API的样例代码，如下表所示，均位于example目录下：

| 例子主题        | 文件名          |
|:-------------- |:-------------|
|内存写入与读取|append.go|
|dfs写入|appendDate.go|
|多线程并行写入数据库|perfBench.go|
|流数据写入|StreamWriting.go|
|流数据订阅|StreamClient.go|


这些例子的开发环境详见[`DolphinDB Go API`](https://www.2xdb.net/dolphindb/api-go/blob/master/README.md)。本文下面对每个例子分别进行简单的说明。

执行样例代码，需在`api-go/`目录下, 添加环境变量，以及更改样例代码中的节点地址、端口和用户名密码
然后在`api-go/`目录下执行go指令：
```Go
  go run ./example/conn.go
```


### 2. 内存表写入与读取

首先，调用`conn.Run`函数，创建一个名为`kt`的带主键的内存表。

使用`CreateTable`函数创建一个结构相同的内存表。使用`GetColumn`方法获取各列，对于各列的数据更改会同时更改内存表的内容。
``相对于每次添加一个Constant对象的`Append`，`AppendInt`等一次添加多个常用类型的方法效率更高``
随后，调用`conn.RunFunc`将内存表的数据添加到`kt`中。这里通过脚本使用了DolphinDB的`tableInsert`函数，这比使用多次DolphinDB SQL语句或者向列append添加到分布式表等方式效率更高。
`RunFunc()`的第二个参数是一个`Constant`类型的slice，可以将多个参数加入其中。

用conn.Run直接执行DolphinDB SQL语句，如
``
conn.Run("select count(*) from kt")
``
   
### 3. 添加数据到dfs表
    
在源代码中，主要有三个示例createDemoTable函数，用于产生模拟数据，返回Table。
在主函数中将返回的Table添加到dfs表中。

CreateDemoTable函数：使用`ddb.CreateVector`方法创建了各种类型的Vector对象，用Append方法将Constant对象添加到对应类型的Vector中，然后调用`ddb.CreateTableByVector(colnames, cols);`用这些列创建一个DolphinDB的Table。

CreateDemoTableFast函数：`ddb.CreateVector`创建Vector后，将要添加的数据append到对应类型的go slice中，然后对Vector对象调用`SetIntArray`等方法，将slice一次性添加，这样的效率将会更高。

CreateDemoTableSlow函数：与前面的区别在于使用了`ddb.CreateConstant`方法创建三种新类型，并对其调用`SetBinary`方法填充数值。

DolphinDB采用数据分区技术，按照一定的规则将大规模数据集水平分区，每个分区内的数据采用列式存储。写入由事务机制保证数据原子性，通常，每次写入会涉及到多个分区，这样多个分区同时参与某一个事务。影响写入吞吐量的主要因素包括：
* 批量写入，每次写入数据量不宜过小，一般每次数据量写入在几十兆字节比较合适。
* 每次写入数据涉及的分区数据量不宜过多。DolphinDB按照分区来列式存储，比如每次写入1000分区10列，那么每次需要写10000(1000*10)个文件，会降低写入性能。
* 开启写入缓存（Cache Engine）可以提升写入效率。
* 并行写入可以提升吞吐量，但不同的线程要保证写不同的分区，否则会写入失败。


### 4. 多线程并行写入数据库

在源代码中，主要有4个函数如下：
* createDemoTable函数，用于产生模拟数据，返回TableSP。
* genData函数，生产数据线程的主函数。
* writeData函数，写数据线程的主函数，与DolphinDB server建立连接，用``RunFunc``方法运行tableInsert脚本把模拟数据写入数据库。
* main函数，接收输入参数，创建线程，显示写入的结果。

在写入时要注意的是，多个wirter不能同时往DolphinDB分布式数据库的同一个分区写数据，所以产生数据时，保证每个线程产生的数据是不同分区的。本例中通过为每个写入线程平均分配分区的方法，保证多个写入线程写到不同的分区。

在执行前需要先将main函数中hosts与ports修改为你的的DolphinDB节点列表。
```GO
  hosts:=[]string{"192.168.1.12","192.168.1.13","192.168.1.14","192.168.1.15","192.168.1.12","192.168.1.13","192.168.1.14","192.168.1.15",
                  "192.168.1.12","192.168.1.13","192.168.1.14","192.168.1.15","192.168.1.12","192.168.1.13","192.168.1.14","192.168.1.15"}
  ports:= [] int {19162,19162,19162,19162,19163,19163,19163,19163,19164,19164,19164,19164,19165,19165,19165,19165 }
```

并在dfs上创建表：
```
def createL3DB(dbName,tableName){
         login("admin","123456")
        if(existsDatabase(dbName)){
                 dropDatabase(dbName)
        }
         db1 = database("", VALUE, datehour(2019.09.11T00:00:00)..datehour(2019.12.30T00:00:00) )//starttime, 配置newValuePartitionPolicy=add
         db2 = database("", HASH, [IPADDR, 50]) //source_address
         db3 = database("", HASH,  [IPADDR, 50]) //destination_address
         db = database(dbName, COMPO, [db1,db2,db3])
         data = table(1:0, ["fwname","filename","source_address","source_port","destination_address","destination_port","nat_source_address","nat_source_port","starttime","stoptime","elapsed_time"], [SYMBOL,STRING,IPADDR,INT,IPADDR,INT,IPADDR,INT,DATETIME,DATETIME,INT])
        db.createPartitionedTable(data,tableName,`starttime`source_address`destination_address)
}
login("admin","123456")


// create database
createL3DB("dfs://natlog","natlogrecords")

//dropDatabase("dfs://natlog")
```

go的多线程使用十分简便，用`go`指令装载相应的函数即可：
```
  for i:=0;i<10;i++{
    go finsert( tablerows, byte(i*5-1), byte(5), int(ddb.GetEpochTime()/1000), i*5, hosts, ports, i, inserttimes);
}
```
在本例中分布式数据库第一层按时间分区，第二层是按IP地址分50个HASH分区。通过为每个写入线程平均分配分区的方法（比如10个线程，50个分区，则线程1写入1-5，线程2写入6-10，其他线程依次类推），保证多个写入线程写到不同的分区。其中每个IP地址的hash值是通过API内置的getHash计算的：得到相同的hash值，说明数据属于相同分区，反之属于不同分区。

在finsert函数中，对每个节点都获得一次连接，以多节点同时写入。
```GO
  var conn ddb.DBConnection;
  conn.Init();
  success := conn.Connect(hosts[p], ports[p], username, password);  
```

### 5. 流数据写入和订阅例子
#### 4.1 代码说明
本例有3个文件，其中
* StreamWriting.go:是流数据写入的源代码文件
* StreamClient.go:是流数据订阅的源代码文件
* createStreamingTable.dos：脚本文件，用于创建例子用到的流数据表等

可以看到，流数据写入的代码与多线程写入的基本相同，同样是使用`tableInsert`来写入，区别在于：
*是流表与分布式数据库不同，流表的数据类型需要用户保证一致，不会自动转换；
*流表没有分区概念，所以多线程可以无限制地同时写入。

多线程和批量写入能显著提高吞吐量和写入性能，建议实际环境中采用多线程并批量写入流数据。

API提供PollingClient类型订阅流表的数据。PollingClient返回一个消息队列，用户可以通过轮查询的方式获取和处理数据。
```
	var client ddb.PollingClient;
	listenport :=rand.Intn(1000)+50000;
	client.New(listenport);
```
定义一个PollingClient对象，随机产生一个端口号用于监听。
然后调用`Subscribe`方法，注意其中的`host`参数不能是localhost。
```
  queue := client.Subscribe(host, port, "st1",  ddb.Def_action_name(), 0);
```
方法返回一个消息队列，不断对其调用Poll即可获得数据流。

