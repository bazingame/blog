---
title: "Redis设计与实现之「单机数据库的一些功能实现」"
summary: Redis 单机数据库的一些功能实现
date: 2021-08-20
weight: 1
tags: ["redis"]
---

## 数据库系统

![server.png (5660×4460) (amazonaws.com)](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/36ffd043-9f7a-4efc-b417-a7ba6d6e34ca/server.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAT73L2G45O3KS52Y5%2F20210821%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20210821T163910Z&X-Amz-Expires=86400&X-Amz-Signature=7ee1fea6746db9b50d5055e6f1d859434db3b1990f29aab638e44f0d39967496&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22server.png%22)

```c
struct redisServer {
  /* General */
	redisDb *db; /* 数组，保存服务器中所有数据库 */
	int dbnum    /* Total number of configured DBs */

	/* Networking */
	list *clients;              /* List of active clients */
	list *clients_to_close;     /* Clients to close asynchronously */
	client *current_client;     /* Current client executing the command. */
	
	/* time cache */
	mstime_t mstime;            /* 'unixtime' in milliseconds. */
	ustime_t ustime;            /* 'unixtime' in microseconds. */
	
	/* RDB / AOF loading information */
	/* Configuration */
	/* AOF persistence */
	/* Replication (master) */
	/* Replication (slave) */
	/* Cluster */
	/* Scripting */
	/* Lazy free */
	/* cpu affinity */
	....
}
```

每个数据库都是由 redis.h/redisDb 结构表示

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB 键空间 */ 
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

### **键空间**

redisDb中的 [dict 字典](https://www.notion.so/Redis-385250b9ad224a25ae7f58f20a4c6165) 保存了所有的键值对，称为键空间。

每个键值对

- 键是一个字符串对象
- 值是五种基本类型对象之一

**键的过期时间**

过期时间为 `expires` 字段，也是 dict 字典，其中保存了所有键的过期时间

每个键值对

- 键指针，指向键空间中的对象
- 值过期时间(unix timestamp milliseconds)

设置过期时间命令：

```c
/* EXPIRE key seconds */
void expireCommand(client *c) {
    expireGenericCommand(c,mstime(),UNIT_SECONDS);
}

/* EXPIREAT key time */
void expireatCommand(client *c) {
    expireGenericCommand(c,0,UNIT_SECONDS);
}

/* PEXPIRE key milliseconds */
void pexpireCommand(client *c) {
    expireGenericCommand(c,mstime(),UNIT_MILLISECONDS);
}

/* PEXPIREAT key ms_time */
void pexpireatCommand(client *c) {
    expireGenericCommand(c,0,UNIT_MILLISECONDS);
}
// 底层都转化为 expireGenericCommand 命令
void expireGenericCommand(client *c, long long basetime, int unit)
void setExpire(client *c, redisDb *db, robj *key, long long when) {}
```

如何过期的在后面讲→

**命令在键空间阶段的执行过程**

- SET

```c
// 入口
void setCommand(client *c) {}  
// 实现了 SET 、 SETEX 、 PSETEX 和 SETNX 命令。
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {}
// 高层次的 SET 操作函数 -> 增加引用计数..
void genericSetKey(client *c, redisDb *db, robj *key, robj *val, int keepttl, int signal) {}
// 将键值对 key 和 val 添加到数据库中，上层增加引用计数
void dbAdd(redisDb *db, robj *key, robj *val) {}
// 尝试将给定键值对添加到字典中
int dictAdd(dict *d, void *key, void *val){}
// 字典的插入操作
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing){}
```

- DEL

```c
void delCommand(redisClient *c) {}
void delGenericCommand(client *c, int lazy) {}
// 从数据库中删除给定的键，键的值，以及键的过期时间。
int dbSyncDelete(redisDb *db, robj *key) {}
// 从字典中删除包含给定键的节点
int dictDelete(dict *ht, const void *key) {}
// 字典的删除操作
static int dictGenericDelete(dict *d, const void *key, int nofree){}
```

- GET

```c
void getCommand(client *c) {}
int getGenericCommand(client *c) {}
// 执行读取操作而从数据库中查找返回 key 的值。
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply) {}
// 为执行读取操作而取出键 key 在数据库 db 中的值。 更新命中/不命中信息
robj *lookupKeyRead(redisDb *db, robj *key) {}
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags) {}
// 从数据库 db 中取出键 key 的值（对象）
robj *lookupKey(redisDb *db, robj *key, int flags) {}
// 字典的查找操作
dictEntry *dictFind(dict *d, const void *key){}
```

### 服务端

**初始化服务器 `void initServer(void) {}`**

- 初始化配置，加载、解析配置文件
- 初始化内部变量
- 事件循环
- socket监听
- 时间事件、文件事件
- 启动事件循环

```c
server.c/main(){
	void initServerConfig(void) {} // 初始化配置，给配置参数赋初始值
	void loadServerConfig(char *filename, char *options) {} // 从给定文件中载入服务器配置
	void initServer(void) {          // 真正初始化服务器内部变量，客户端链表、数据库、全局变量和共享对象等
		createSharedObjects();     //初始化共享变量...
		server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR); // 初始化事件处理器状态...
		...
		server.db = zmalloc(sizeof(redisDb)*server.dbnum); // 创建数据库

		/* Open the TCP listening socket for the user commands. */ // 启动监听
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
        exit(1);
    if (server.tls_port != 0 &&
        listenToPort(server.tls_port,server.tlsfd,&server.tlsfd_count) == C_ERR)
        exit(1);
    ...
		/* Create the Redis databases, and initialize other internal state. */
    for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
        server.db[j].expires_cursor = 0;
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&objectKeyPointerValueDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
        server.db[j].defrag_later = listCreate();
        listSetFreeMethod(server.db[j].defrag_later,(void (*)(void*))sdsfree);
    }
		...
		if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }

		/* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */ // 对于监听的socket创建对应的文件事件。
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
		...
		aeMain(server.el); // 事件处理器的主循环
	}  
}
```

**命令在接收后阶段的执行过程**

- 命令解析(通信协议, RESP)

  > redis 协议:    `\\r\\n`(CRLF)区分命令请求的若干参数， `*n` 表示n个参数,  `$n`  表示参数字符串长度。 如 `SET KEY VALUE` ，转换协议后为： `*3\\r\\n$3\\r\\nSET\\r\\n$3\\r\\nKEY\\r\\n$5\\r\\nVALUE\\r\\n` 特点：

  - 易于实现
  - 可以高效地被计算机分析（数据的长度放在数据正文)
  - 可以很容易地被人类读懂 (非二进制)

- 命令调用

- 返回结果

```c
connSetReadHandler(conn, readQueryFromClient); // -> 绑定命令处理器
void readQueryFromClient(connection *conn) {
		// 缓冲区一堆操作...
		
		// 从查询缓存读取内容，创建参数，并执行命令
		void processInputBuffer(client *c) {
				...
				// 判断请求类型   内联命令 和 其他普通
				// 将缓冲区中的内容转换成命令，以及命令参数
				int processMultibulkBuffer(redisClient *c) {
					// 参数协议解析
				}
				// 执行命令
				int processCommandAndResetClient(client *c) {
						int processCommand(client *c) {
								c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr); // 查找命令，并进行命令合法性检查，以及命令参数个数检查

								/* Exec the command */
							    if (c->flags & CLIENT_MULTI &&
							        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
							        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
							    {
							        queueMultiCommand(c);   // 事务命令处理
							        addReply(c,shared.queued);
							    } else {
							         // 执行命令
											void call(client *c, int flags) {
										    c->cmd->proc(c);   // 执行实现函数
											}
							        c->woff = server.master_repl_offset;
							        if (listLength(server.ready_keys))
							            handleClientsBlockedOnKeys();
							    }
						}
				}
		}

}
```

## 事件

Redis 是 事件驱动程序 ，有两类事件

- 文件事件

  套接字与客户端连接，Server和Client通信，产生相应的文件事件，服务器通过监听处理这些事件来完成通信操作

- 时间事件

  服务器中维护的一些操作，在给定时间点执行，抽象为时间事件。如清理过期键值对、持久化、同步等等。

### 事件的调度执行

```c
// 事件处理器的主循环

void aeMain(aeEventLoop *eventLoop) {

    eventLoop->stop = 0;

    while (!eventLoop->stop) {

        // 如果有需要在事件处理前执行的函数，那么运行它
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}

// 处理所有已到达的时间事件，以及所有已就绪的文件事件。
int aeProcessEvents(aeEventLoop *eventLoop, int flags){
	// 计算最快要执行的时间事件的等待时间
	aeSearchNearestTimer(eventLoop);
	// 阻塞等待文件事件
	aeApiPoll(eventLoop, tvp);
	// 处理文件事件
	for (j = 0; j < numevents; j++) {
    // 读事件
    fe->rfileProc(eventLoop,fd,fe->clientData,mask);            
	  // 写事件
    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
		processed++;
        
	}
	// 处理时间事件
	processTimeEvents(aeEventLoop *eventLoop)
}
```

### 文件事件

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f7fba0eb-8679-4098-95c2-f912b8fbe5ea/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f7fba0eb-8679-4098-95c2-f912b8fbe5ea/Untitled.png)

- I/O 多路复用监听套接字，根据套接字的任务不同，分配不同的任务处理器
- 当套接字准备好执行操作时，与操作相应的文件事件就会产生，调用关联好的事件处理器

多种文件处理器：

- 连接应答处理器 `acceptTcpHandler`
- 命令请求处理器 `readQueryFromClient`
- 命令回复处理器 `sendReplyToClient`

命令在文件事件阶段的执行过程

```c
// 客户端申请连接，socket 将产生 `AE_READABLE` 类型事件，交给连接处理器处理
aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler,NULL) == AE_ERR)
// 客户端发送命令请求，产生一个 `AE_READABLE` 类型事件，交给命令请求处理器...
aeCreateFileEvent(server.el,fd,AE_READABLE, readQueryFromClient, c) == AE_ERR
// 客户端尝试读取回复，产生一个 `AE_WRITEABLE` 类型事件，交给回复处理器...
```

### 时间事件

- 定时事件
- 周期事件

**serverCron 函数**

- 更新服务器信息，时间、内存
- 过期
- 持久化
- 同步

### 线程模型的进化

**单线程**

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ce2276e1-c449-4081-9512-c26c1569d787/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ce2276e1-c449-4081-9512-c26c1569d787/Untitled.png)

单线程模型的好处

- 方便维护、开发、调试
- 使用单线程模型也能并发的处理客户端的请求，I/O多路复用
- 大多数操作在内存中完成，性能瓶颈不在CPU，在于网络I/O

**不仅仅单线程**

redis 在4.0 版本引入一些支持异步处理的删除命令， `UNLINK` `FLUSHDB ASYNC` ...

对于超大键值对，Redis 可能会需要在释放内存空间上消耗较多的时间，这些操作就会阻塞待处理的任务

```c
void unlinkCommand(client *c) {
   /* This command implements DEL and LAZYDEL. */
	void delGenericCommand(client *c, int lazy) {
	    int numdel = 0, j;
	
	    for (j = 1; j < c->argc; j++) {
	        expireIfNeeded(c->db,c->argv[j]);
	        int deleted  = lazy ? dbAsyncDelete(c->db,c->argv[j]) :
	                              dbSyncDelete(c->db,c->argv[j]);
	        if (deleted) {
	            signalModifiedKey(c,c->db,c->argv[j]);
	            notifyKeyspaceEvent(NOTIFY_GENERIC,
	                "del",c->argv[j],c->db->id);
	            server.dirty++;
	            numdel++;
	        }
	    }
	    addReplyLongLong(c,numdel);
	}
}
```

**网络IO处理多线程**

> I/O 多路复用的主要作用是让我们可以使用一个线程来监控多个连接是否可读或者可写，但是从网络另一头发的数据包需要先解序列化成 Redis 内部其他模块可以理解的命令，这个过程就是 Redis 6.0 引入多线程来并发处理的。 I/O 多路复用模块收到数据包之后将其丢给后面多个 I/O 线程进行解析，I/O 线程处理结束后，主线程会负责串行的执行这些命令，由于向客户端发回数据包的过程也是比较耗时的，所以执行之后的结果也会交给多个 I/O 线程发送回客户端。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5dda5268-afa7-4607-8d18-306f71db15fa/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5dda5268-afa7-4607-8d18-306f71db15fa/Untitled.png)

实现了I/O读写的多线程，而执行命令依旧是单线程。

```c
// 在多线程I/O开启时，将读事件放入队列中，等待主线程将读事件分配给I/O线程
// 这里只是把读事件添加到clients_pending_read队列中而已
int postponeClientRead(client *c) {}

// 执行
int handleClientsWithPendingReadsUsingThreads(void) {
	// 将读事件根据RR，分配给所有的I/O线程（包括主线程自己）
	while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }
	// 主线程读取并解析客户端请求（和I/O线程一起，但不执行命令）
	listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        // 这里对应3.1.a中postponeClientRead会返回0, 所以数据会被读取和解析
        readQueryFromClient(c->conn);
    }
	// 等待I/O线程读取完毕
		while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }
	// 遍历所有的事件（通过遍历clients_pending_read上的客户端），解析和执行命令
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read,ln);

        if (c->flags & CLIENT_PENDING_COMMAND) {
            // b) 若CLIENT_PENDING_COMMAND被标记，说明有命令被解析出来
            // 所以要执行命令
            c->flags &= ~CLIENT_PENDING_COMMAND;
            if (processCommandAndResetClient(c) == C_ERR) {
                /* If the client is no longer valid, we avoid
                 * processing the client later. So we just go
                 * to the next. */
                continue;
            }
        }
        // c) 命令解析和执行
        processInputBufferAndReplicate(c);
    }
}
```

I/O线程的任务

```c
void *IOThreadMain(void *myid) {
	// 轮询，判断是否有任务要做
	// 根据任务类型（读/写），执行任务
	long id = (unsigned long)myid;
    while(1) {
        // 轮询，判断是否有任务过来
        for (int j = 0; j < 1000000; j++) {
            if (io_threads_pending[id] != 0) break;
        }
		// 遍历任务列表，根据任务类型，执行I/O任务
        listIter li;
        listNode *ln;
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0); // 写，返回响应
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn); // 读, 上面提过，不会执行命令
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        // 清空任务列表，并设置未完成任务数为0
        listEmpty(io_threads_list[id]);
        io_threads_pending[id] = 0;
        // ...
    }
}
```

## Todo

- 持久化 - -
- 事务  - -

## Reference

[通信协议（protocol） - Redis 命令参考](http://redisdoc.com/topic/protocol.html)

[为什么 Redis 选择单线程模型 - 面向信仰编程](https://draveness.me/whys-the-design-redis-single-thread/)

[Redis 中的事件循环 - 面向信仰编程](https://draveness.me/redis-eventloop/)
