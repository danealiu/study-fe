# mongodb复制集安装全流程

安装mongodb
---
* mkdir -p /apps/mongodb/
* 去官网下载 https://docs.mongodb.com/manual/installation/
下完后直接解压缩 到 /apps/mongodb/
* mkdir -p /data/mongodb/log   /data/mongodb/db
* 在 /apps/mongodb/bin/ 下创建mongodb.conf文件
* 使用yaml格式，示例如下
```
systemLog:
  destination: file
  path: "/data/mongodb/log/mdb.log"
  logAppend: true

storage:
  dbPath: "/data/mongodb/db"
  indexBuildRetry: true
  journal:
    enabled: true

processManagement:
  fork: true

net:
  bindIp: "1.2.3.4"  // 这只是一个示例
  port: 27017
  maxIncomingConnections: 65536

#snpm:

# master: true
security:
  # enabled or disabled
  authorization: "enabled"
  keyFile: /apps/mongodb/mongodb-linux-x86_64-3.6.12-rc1/bin/mongodb-keyfile

replication:
  replSetName: rs_api_manager

```

刚开始还没有创建用户，需要把security隐藏，replication隐藏

* 在bin目录下启动mongod  ， ./mongod -f mongodb.conf
* 打开数据库  ./mongo 1.2.3.4:27017 (这个ip只是一个示例)
* use admin
* 创建用户
```angnular2
 db.createUser({user: "admin",pwd: "***", roles: [{role:"root”,db:"admin"}]})
```

root是最高权限，具体可以参照 https://docs.mongodb.com/manual/reference/built-in-roles/index.html
* 然后退出来，重启Mongodb，记住不能Kill -9，需要kill -2，因为它在关闭前还要进行其它操作
。这时候可以打开 security.authorization
* 然后use admin;  db.auth("admin", "****"); 然后再创建一个数据库test; use test; 然后再创建这个数据库的用户。
* 按照上面的方法在另外一台机器上创建一个Mongo，接下来就是复制集。记得修改config文件，设置replication，然后重启
* 首先要保证数据库有cluster权限，可以updateUser增加权限，如clusterAdmin 或 clusterManager
* 然后在主机器上或有数据的机器上执行  rs.status()  
* 如果没有就
```angular2
rs.initiate(
   {
      _id: "myReplSet",
      version: 1,
      members: [
         { _id: 0, host : "mongodb0.example.net:27017" }
      ]
   }
)
``` 

_id 是你的复制集名称
host 是你对应的ip  port
* 然后添加新的node上去
```angular2
rs.add( "mongodbd4.example.net:27017" )
```

如果报错，可能是防火墙没关闭，iptables stop。

如果还报错 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426193, 1) } with id
这是因为mongodb需要keyfile，在副本集和分片集群上启用认证后，其内部成员之间的访问就必须提供凭证以进行认证。

这时你可以打开config中的 keyfile了，设置它
```angular2
openssl rand -base64 741 > /srv/mongodb/mongodb-keyfile
chmod 600 mongodb-keyfile
```
记住：2个机器上需要是同样的keyfile文件。然后重启mongo
* 这时再rs.add就成功了，成功后可以登陆到另一个服务上看是否同步过来
* 另一个不是主结点，需要rs.slaveOk()  主结点有读写权限，从结点只有读的权限
