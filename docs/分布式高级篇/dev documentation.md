---
typora-root-url: images
typora-copy-images-to: images
---


# 谷粒商城高级篇

> 笔记中仍有很多不足，如有错误还请包涵(●ˇ∀ˇ●)

# 1、Elasticsearch - 全文检索

### 简介

通常，我们用mysql做持久化存储，ES用作检索

https://www.elastic.co/cn/what-is/elasticsearch/

全文搜索属于最常见的需求，开源的 Elasticsearch 是目前全文搜索引擎的首选。

他可以快速地存储、搜索和分析海量数据。维基百科、Stack Overflow、Github 都采用他

![image-20201026084916698](/image-20201026084916698.png)

Elastic 的底层是开源库Lucene。但是，你没法直接用，必须自己写代码调用它的接口，Elastic 是 Lucene 的封装，提供了 REST API 的操作接口，开箱即用

REST API：天然的跨平台

官网文档：https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

官网中文：https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html

社区中文：

http://doc.codingdict.com/elasticsearch/

### 1.1、基本概念

`index库`>`type表`>`document文档`

#### 1.1.1 index(索引)

动词，相当于 MySQL 中的 insert；

名词，相当于MySQL 中的 DataBase

#### 1.1.2 Type(类型)

在 Index（索引）中，可以定义一个或多个类型

类似于 MySQL 中的 Table，每一种类型的数据放在一起

#### 1.1.3 Document(文档)

保存在某个索引（index）下，某种类型（Type）的一个数据（Document）,文档是 JSON 格式的，Document 就像是 MySQL 中某个 Table 里面的内容

#### 1.1.4 倒排索引机制-ES搜索快的秘密

> 倒排索引，英文原名Inverted index，大概因为 Invert 有颠倒的意思，就被翻译成了倒排。
> 但是倒排这个名称很容易让人误解为从A-Z颠倒成Z-A。
>
> 个人认为翻译成转置索引可能比较合适。
> 一个未经处理的数据库中，一般是以文档ID作为索引，以文档内容作为记录。
> 而Inverted index 指的是将单词或记录作为索引，将文档ID作为记录，这样便可以方便地通过单词或记录查找到其所在的文档。

保存的记录

- 红海行动
- 探索红海行动
- 红海特别行动
- 红海记录片
- 特工红海特别探索

将内容分词，创建倒排索引。

| 词     | 记录      |
| ------ | --------- |
| 红海   | 1,2,3,4,5 |
| 行动   | 1,2,3     |
| 探索   | 2,5       |
| 特别   | 3,5       |
| 纪录片 | 4,        |
| 特工   | 5         |

检索过程：

1）、红海特工行动？查出后计算相关性得分：3号记录命中了2次，且3号本身才有3个单词，2/3，所以3号最匹配
2）、红海行动？

> 思考：学过MySql等常见数据库的知道，模糊检索%xxx%不能使用索引，执行效率极低。
>
> es通过分词并创建倒排索引的机制克服了模糊检索效率低的问题。
>
> 用一个字符串str1在es里检索，先用分词器将str1拆分成几个词。每个词通过倒排索引找出匹配的记录。哪个记录匹配的次数多，就得分最高，展示在最前面。
>
> 比如我们想在字典查出包含“车”的所有字。现有字典没有这种索引，只能创建一种新索引，才能满足这种需求。

> 消失的type参数
>
> 关系型数据库中两个数据表示是独立的，即使他们里面有相同名称的列也不影响使用，但ES中不是这样的。elasticsearch是基于Lucene开发的搜索引擎，而ES中不同type下名称相同的filed最终在Lucene中的处理方式是一样的。
>
> 两个不同type下的两个user_name，在ES同一个索引下其实被认为是同一个filed，你必须在两个不同的type中定义相同的filed映射。否则，不同type中的相同字段名称就会在处理中出现冲突的情况，导致Lucene处理效率下降。
> 去掉type就是为了提高ES处理数据的效率。
> Elasticsearch 7.xURL中的type参数为可选。比如，索引一个文档不再要求提供文档类型。
> Elasticsearch 8.x不再支持URL中的type参数。
> 解决：将索引从多类型迁移到单类型，每种类型文档一个独立索引。

### 1.2 Docker 安装 ES

#### 1.2.1 下载镜像

docker pull elasticsearch:7.4.2  存储和检索数据

docker pull kibana:7.4.2 可视化检索数据	

#### 1.2.2 创建实例

##### 1、ElasticSearch

配置

```bash
mkdir -p /mydata/elasticsearch/config # 用来存放配置文件
mkdir -p /mydata/elasticsearch/data  # 数据
echo "http.host: 0.0.0.0" >/mydata/elasticsearch/config/elasticsearch.yml # es可以被远程任何机器访问
chmod -R 777 /mydata/elasticsearch/ ## 设置elasticsearch文件可读写权限
```

启动

```bash
# 9200是用户交互端口 9300是集群心跳端口
# -e指定是单阶段运行
# -e指定占用的内存大小，生产时可以设置32G
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e  "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx512m" \
-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-v  /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.4.2 
```

开机启动 elasticsearch

```bash
docker update elasticsearch --restart=always
```

以后在外面装好插件重启就可

> 因为容器里的文件映射到了外面，所以删除容器和新建容器数据还在

> 第一次查docker ps启动了，第二次查的时候发现关闭了，docker logs elasticsearch

##### 2、Kibana

```bash
docker run --name kibana -e ELASTICSEARCH_HOSTS=http://192.168.56.10:9200 -p 5601:5601 -d kibana:7.4.2

http://192.168.56.10:9200 改成自己Elasticsearch上的地址
```

访问Kibana： http://#:5601/app/kibana 

![image-20200501192629304](image-20200501192629304.png)

> 遇到了更新阿里源也下载不下来kibana镜像的情况，先在别的网络下载下来后传到vagrant中
>
> ```shell
> docker save -o kibana.tar kibana:7.4.2 
> 
> docker load -i kibana.tar 
> 
> # 如何通过其他工具链接ssh
> 
> 修改/etc/ssh/sshd_config
> 修改 PasswordAuthentication yes
> 
> systemctl restart sshd.service  或 service sshd restart
> 
> # 连接192.168.56.10:22端口成功，用户名root，密码vagrant
> 
> 也可以通过vagrant ssh-config查看ip和端口，此时是127.0.0.1:2222
> 
> 
> ```

> 在安装离线docker镜像的时候还提示内存不足，看了下是因为外部挂载的内存也算在了vagrant中，即使外部删了很多文件，vagrant中df -h硬盘占用率也不下降。我在外部删完文件后在内部又rm -rf XXX 强行解除占用



##### 3、安装nginx

随便启动一个 nginx 实例，只是为了复制出配置

```
docker run -p80:80 --name nginx -d nginx:1.10   
```

将容器内的配置文件拷贝到当前目录 （注意后面有个小点）

```bash
docker container cp nginx:/etc/nginx .  
```

创建nginx文件夹

```shell
mkdir -p /mydata/nginx/html
mkdir -p /mydata/nginx/logs
 #由于拷贝完成后会在config中存在一个nginx文件夹，所以需要将它的内容移动到conf中
 #conf 文件夹下就是原先nginx的配置
mv /mydata/nginx/conf/nginx/* /mydata/nginx/conf/
rm -rf /mydata/nginx/conf/nginx
```

别忘了后面的点

修改文件名称：mv nginx.conf 把这个conf 移动到 /mydata/nginx 下

终止原容器， docker stop nginx

执行命令删除容器：docker rm $Containerid

创建新的 nginx 执行以下命令

```bash
 docker run -p 80:80 --name nginx \
 -v /mydata/nginx/html:/usr/share/nginx/html \
 -v /mydata/nginx/logs:/var/log/nginx \
 -v /mydata/nginx/conf/:/etc/nginx \
 -d nginx:1.10
```



### 1.3 初步检索

#### 1.3.1、_cat

（1）GET/_cat/nodes：查看所有节点_

 如：http://#:9200/_cat/nodes :

```
127.0.0.1 61 91 11 0.08 0.49 0.87 dilm * 0adeb7852e00
```

注：*表示集群中的主节点

（2）GET/_cat/health：查看es健康状况_

如： http://#:9200/_cat/health 

```
1588332616 11:30:16 elasticsearch green 1 1 3 3 0 0 0 0 - 100.0%
```

注：green表示健康值正常

（3）GET/_cat/master：查看主节点_

如： http://#:9200/_cat/master 

```
vfpgxbusTC6-W3C2Np31EQ 127.0.0.1 127.0.0.1 0adeb7852e00
```

（4）GET/_cat/indicies：查看所有索引 ，等价于mysql数据库的show databases;

如： http://#:9200/_cat/indices 

```json
green open .kibana_task_manager_1   KWLtjcKRRuaV9so_v15WYg 1 0 2 0 39.8kb 39.8kb
green open .apm-agent-configuration cuwCpJ5ER0OYsSgAJ7bVYA 1 0 0 0   283b   283b
green open .kibana_1                PqK_LdUYRpWMy4fK0tMSPw 1 0 7 0 31.2kb 31.2kb
```

####  

#### 1.3.2 索引一个文档（保存）

保存一个数据，保存在哪个索引的哪个类型下，指定用哪个唯一标识

PUT customer/external/1; 在 customer 索引下的 external 类型下保存 1号数据为

```http
PUT customer/external/1
```

#### 1.3.3、查询文档

```http
GET custome/external/1
```

结果：

```json
{
    "_index": "customer", // 在那个索引
    "_type": "external", // 在那个类型
    "_id": "1", // 记录id
    "_version": 1, 。// 版本号
    "_seq_no": 0, // 并发控制字段，每次更新就会+1，用来做乐观锁
    "_primary_term": 1, //同上，主分片重新分配，如重启，就会变化
    "found": true, 
    "_source": {
        "name": "John Doe" // 真正的内容
    }
}
```

```
在使用乐观锁更新时，就可以通过携带：?if_seq_no=4&if_primary_term=1  实现了乐观锁更新
```



#### 1.3.4 更新文档

```http
POST customer/external/1/_update
{
	"doc":{
		"name":"John Doew"
	}
}
或者
POST customer/external/1
{
	"name":"John Doe2"
}
或者
PUT customer/external/1
{
	"name":"jack"
}
不同：Post操作会对比源文档数据，如果相同不会有什么操作，文档 version 不增加 PUT操作总会将数据重新保存并增加 version 版本
带 _update 对比元数据如果一样就不进行任何操作
看场景
对于大并发更新，不带update
对于大并发查询并偶尔更新，带update 对比更新，重新计算分配规则
更新同时增加属性
POS customer/external/1/_update
{
	"doc":{"name":"Jane Doe","age":20}
}
PUT 和 POST 不带_update也可以
```

#### 1.3.5 PUT与POST的比较

保存一个数据，保存在哪个索引的哪个类型下，指定用那个唯一标识
PUT customer/external/1;在customer索引下的external类型下保存1号数据为

```
PUT customer/external/1
```



```json
{
 "name":"John Doe"
}
```

PUT和POST都可以
POST新增。如果不指定id，会自动生成id。指定id就会修改这个数据，并新增版本号；
PUT可以新增也可以修改。PUT必须指定id；由于PUT需要指定id，我们一般用来做修改操作，不指定id会报错。



下面是在postman中的测试数据：
![image-20200501194449944](image-20200501194449944.png)

创建数据成功后，显示201 created表示插入记录成功。

```json
{
    "_index": "customer",
    "_type": "external",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

这些返回的JSON串的含义；这些带有下划线开头的，称为元数据，反映了当前的基本信息。

"_index": "customer" 表明该数据在哪个数据库下；

"_type": "external"     表明该数据在哪个类型下；

"_id": "1"                    表明被保存数据的id；

 "_version": 1,            被保存数据的版本

"result": "created"      这里是创建了一条数据，如果重新put一条数据，则该状态会变为updated，并且版本号也会发生变化。



下面选用POST方式：

添加数据的时候，不指定ID，会自动的生成id，并且类型是新增：

<img src="/image-20200501195619925.png" alt="image-20200501195619925" style="zoom: 52%;" />

再次使用POST插入数据，仍然是新增的：

<img src="/image-20200501195732492.png" alt="image-20200501195732492" style="zoom: 80%;" />



添加数据的时候，指定ID，会使用该id，并且类型是新增：

<img src="/image-20200501200048361.png" alt="image-20200501200048361" style="zoom: 66%;" />

再次使用POST插入数据，类型为updated

<img src="/image-20200501200132199.png" alt="image-20200501200132199" style="zoom:67%;" />

#### 1.3.5 乐观锁更新

GET /customer/external/1

 http://#:9200/customer/external/1 

```json
{
    "_index": "customer",//在哪个索引
    "_type": "external",//在哪个类型
    "_id": "1",//记录id
    "_version": 3,//版本号
    "_seq_no": 6,//并发控制字段，每次更新都会+1，用来做乐观锁
    "_primary_term": 1,//同上，主分片重新分配，如重启，就会变化
    "found": true,
    "_source": {
        "name": "John Doe"
    }
}
```

 



通过“if_seq_no=1&if_primary_term=1 ”，当序列号匹配的时候，才进行修改，否则不修改。

实例：将id=1的数据更新为name=1，然后再次更新为name=2，起始_seq_no=6，_primary_term=1

（1）将name更新为1

 http://#:9200/customer/external/1?if_seq_no=6&if_primary_term=1 

<img src="/image-20200501212224983-1622618846259.png" alt="image-20200501212224983" style="zoom: 61%;" />

 （2）将name更新为2，更新过程中使用seq_no=6

http://#:9200/customer/external/1?if_seq_no=6&if_primary_term=1 

<img src="/image-20200501213047499-1622618846260.png" alt="image-20200501213047499" style="zoom: 60%;" />

出现更新错误。



（3）查询新的数据

 http://#:9200/customer/external/1 

![image-20200501212924094](/image-20200501212924094-1622618846260.png)

能够看到_seq_no变为7。

（4）再次更新，更新成功

 http://#:9200/customer/external/1?if_seq_no=7&if_primary_term=1 

<img src="/image-20200501213130001-1622618846260.png" alt="image-20200501213130001" style="zoom:75%;" />

#### 4）更新文档

![image-20200501214522818](/image-20200501214522818-1622618846260.png)

 ![image-20200501215746139](/image-20200501215746139-1622618846260.png)

（1）POST更新文档，带有_update

http://#:9200/customer/external/1/_update 

![image-20200501214810741](/image-20200501214810741-1622618846260.png)

如果再次执行更新，则不执行任何操作，序列号也不发生变化

![image-20200501214912607](/image-20200501214912607-1622618846260.png)

POST更新方式，会对比原来的数据，和原来的相同，则不执行任何操作（version和_seq_no也不会变化）。

 （2）POST更新文档，不带_update

![image-20200501215358666](/image-20200501215358666-1622618846260.png)

在更新过程中，重复执行更新操作，数据也能够更新成功，不会和原来的数据进行对比。

#### 1.3.6 删除文档&索引

#### 

```
DELETE customer/external/1
DELETE customer
```

注：elasticsearch并没有提供删除类型的操作，只提供了删除索引和文档的操作。



实例：删除id=1的数据，删除后继续查询

<img src="/image-20200501220559094.png" alt="image-20200501220559094" style="zoom:67%;" />

实例：删除整个costomer索引数据

删除前，所有的索引

```
green  open .kibana_task_manager_1   KWLtjcKRRuaV9so_v15WYg 1 0 2 0 39.8kb 39.8kb
green  open .apm-agent-configuration cuwCpJ5ER0OYsSgAJ7bVYA 1 0 0 0   283b   283b
green  open .kibana_1                PqK_LdUYRpWMy4fK0tMSPw 1 0 7 0 31.2kb 31.2kb
yellow open customer                 nzDYCdnvQjSsapJrAIT8Zw 1 1 4 0  4.4kb  4.4kb
```

删除“ customer ”索引

![image-20200501221105476](/image-20200501221105476.png)

删除后，所有的索引

```
green  open .kibana_task_manager_1   KWLtjcKRRuaV9so_v15WYg 1 0 2 0 39.8kb 39.8kb
green  open .apm-agent-configuration cuwCpJ5ER0OYsSgAJ7bVYA 1 0 0 0   283b   283b
green  open .kibana_1                PqK_LdUYRpWMy4fK0tMSPw 1 0 7 0 31.2kb 31.2kb
```



#### 1.3.7 eleasticsearch的批量操作——bulk

语法格式：

```json
{action:{metadata}}\n
{request body  }\n

{action:{metadata}}\n
{request body  }\n
```

这里的批量操作，当发生某一条执行发生失败时，其他的数据仍然能够接着执行，也就是说彼此之间是独立的。

bulk api以此按顺序执行所有的action（动作）。如果一个单个的动作因任何原因失败，它将继续处理它后面剩余的动作。当bulk api返回时，它将提供每个动作的状态（与发送的顺序相同），所以您可以检查是否一个指定的动作是否失败了。

实例1: 执行多条数据


```json
POST customer/external/_bulk
{"index":{"_id":"1"}}
{"name":"John Doe"}
{"index":{"_id":"2"}}
{"name":"John Doe"}
```

执行结果

```json
#! Deprecation: [types removal] Specifying types in bulk requests is deprecated.
{
  "took" : 491,
  "errors" : false,
  "items" : [
    {
      "index" : {
        "_index" : "customer",
        "_type" : "external",
        "_id" : "1",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "customer",
        "_type" : "external",
        "_id" : "2",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}

```



实例2：对于整个索引执行批量操作

```json
POST /_bulk
{"delete":{"_index":"website","_type":"blog","_id":"123"}}
{"create":{"_index":"website","_type":"blog","_id":"123"}}
{"title":"my first blog post"}
{"index":{"_index":"website","_type":"blog"}}
{"title":"my second blog post"}
{"update":{"_index":"website","_type":"blog","_id":"123"}}
{"doc":{"title":"my updated blog post"}}
```

运行结果：

```json
#! Deprecation: [types removal] Specifying types in bulk requests is deprecated.
{
  "took" : 608,
  "errors" : false,
  "items" : [
    {
      "delete" : {
        "_index" : "website",
        "_type" : "blog",
        "_id" : "123",
        "_version" : 1,
        "result" : "not_found",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 1,
        "status" : 404
      }
    },
    {
      "create" : {
        "_index" : "website",
        "_type" : "blog",
        "_id" : "123",
        "_version" : 2,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "website",
        "_type" : "blog",
        "_id" : "MCOs0HEBHYK_MJXUyYIz",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 2,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "update" : {
        "_index" : "website",
        "_type" : "blog",
        "_id" : "123",
        "_version" : 3,
        "result" : "updated",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 3,
        "_primary_term" : 1,
        "status" : 200
      }
    }
  ]
}

```



#### 7）样本测试数据

准备了一份顾客银行账户信息的虚构的JSON文档样本。每个文档都有下列的schema（模式）。

```json
{
	"account_number": 1,
	"balance": 39225,
	"firstname": "Amber",
	"lastname": "Duke",
	"age": 32,
	"gender": "M",
	"address": "880 Holmes Lane",
	"employer": "Pyrami",
	"email": "amberduke@pyrami.com",
	"city": "Brogan",
	"state": "IL"
}
```

 https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json ，导入测试数据，

POST bank/account/_bulk

#### 1.3.8 样本测试数据

我准备了一份顾客银行账户信息虚构的 JSON 文档样本，每个用户都有下列的 schema （模式）：

```json
{
	"account_number": 1,
	"balance": 39225,
	"firstname": "Amber",
	"lastname": "Duke",
	"age": 32,
	"gender": "M",
	"address": "880 Holmes Lane",
	"employer": "Pyrami",
	"email": "amberduke@pyrami.com",
	"city": "Brogan",
	"state": "IL"
}
```

https://github.com/elastic/elasticsearch/edit/master/docs/src/test/resources/accounts.json

导入测试数据

POST bank/account/_bulk

测试数据

![image-20201026114903942](image-20201026114903942.png)



### 1.4 进阶检索

#### 1.4.1 SearchAPI

ES 支持两种基本方式检索:

- 一个是通过使用 REST request URL,发送搜索参数，(uri + 检索参数)
- 另一个是通过使用 REST request bod 来发送他们，(uri + 请求体)

一切检索从_search开始

1. uri+检索参数

> GET /bank/_search 检索 bank 下的所有信息，包括 type 和 docs
>
> GET /bank/_search?q=*&sort=account_number:asc 请求参数方式检索

响应结果解释
> took - Elasticearch 执行搜索的时间(毫秒)
>
> time_ out - 告诉我们搜索是否超时
>
> _shards - 告诉我们多少个分片被搜索了，以及统计了成功/失败的搜索分片
> hit - 搜索结果
> hits.total - 搜索结果
> hits.hits - 实际的搜索结果数组(默认为前10的文档)
> sort - 结果的排序key (键) (没有则按 score 排序)
> score 和 max score - 相关性得分和最高得分(全文检索用)

2. uri + 请求体

```java
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

HTTP 客户端工具（POSTMAN）,get请求不能携带请求体，我们用 post也是一样的 我们 POST 一个 JSON风格的查询请求体到 _search API

需要了解，一旦搜索结果被返回，ES 就完成了这次请求的搜索，并且不会维护任何服务端的资源或者结果的 cursor（游标）。

```
GET bank/_search?q=*&sort=account_number:asc
```

返回结果：

```json
{
  "took" : 235,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "0",
        "_score" : null,
        "_source" : {
          "account_number" : 0,
          "balance" : 16623,
          "firstname" : "Bradshaw",
          "lastname" : "Mckenzie",
          "age" : 29,
          "gender" : "F",
          "address" : "244 Columbus Place",
          "employer" : "Euron",
          "email" : "bradshawmckenzie@euron.com",
          "city" : "Hobucken",
          "state" : "CO"
        },
        "sort" : [
          0
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "account_number" : 1,
          "balance" : 39225,
          "firstname" : "Amber",
          "lastname" : "Duke",
          "age" : 32,
          "gender" : "M",
          "address" : "880 Holmes Lane",
          "employer" : "Pyrami",
          "email" : "amberduke@pyrami.com",
          "city" : "Brogan",
          "state" : "IL"
        },
        "sort" : [
          1
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "2",
        "_score" : null,
        "_source" : {
          "account_number" : 2,
          "balance" : 28838,
          "firstname" : "Roberta",
          "lastname" : "Bender",
          "age" : 22,
          "gender" : "F",
          "address" : "560 Kingsway Place",
          "employer" : "Chillium",
          "email" : "robertabender@chillium.com",
          "city" : "Bennett",
          "state" : "LA"
        },
        "sort" : [
          2
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "3",
        "_score" : null,
        "_source" : {
          "account_number" : 3,
          "balance" : 44947,
          "firstname" : "Levine",
          "lastname" : "Burks",
          "age" : 26,
          "gender" : "F",
          "address" : "328 Wilson Avenue",
          "employer" : "Amtap",
          "email" : "levineburks@amtap.com",
          "city" : "Cochranville",
          "state" : "HI"
        },
        "sort" : [
          3
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "4",
        "_score" : null,
        "_source" : {
          "account_number" : 4,
          "balance" : 27658,
          "firstname" : "Rodriquez",
          "lastname" : "Flores",
          "age" : 31,
          "gender" : "F",
          "address" : "986 Wyckoff Avenue",
          "employer" : "Tourmania",
          "email" : "rodriquezflores@tourmania.com",
          "city" : "Eastvale",
          "state" : "HI"
        },
        "sort" : [
          4
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "5",
        "_score" : null,
        "_source" : {
          "account_number" : 5,
          "balance" : 29342,
          "firstname" : "Leola",
          "lastname" : "Stewart",
          "age" : 30,
          "gender" : "F",
          "address" : "311 Elm Place",
          "employer" : "Diginetic",
          "email" : "leolastewart@diginetic.com",
          "city" : "Fairview",
          "state" : "NJ"
        },
        "sort" : [
          5
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "6",
        "_score" : null,
        "_source" : {
          "account_number" : 6,
          "balance" : 5686,
          "firstname" : "Hattie",
          "lastname" : "Bond",
          "age" : 36,
          "gender" : "M",
          "address" : "671 Bristol Street",
          "employer" : "Netagy",
          "email" : "hattiebond@netagy.com",
          "city" : "Dante",
          "state" : "TN"
        },
        "sort" : [
          6
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "7",
        "_score" : null,
        "_source" : {
          "account_number" : 7,
          "balance" : 39121,
          "firstname" : "Levy",
          "lastname" : "Richard",
          "age" : 22,
          "gender" : "M",
          "address" : "820 Logan Street",
          "employer" : "Teraprene",
          "email" : "levyrichard@teraprene.com",
          "city" : "Shrewsbury",
          "state" : "MO"
        },
        "sort" : [
          7
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "8",
        "_score" : null,
        "_source" : {
          "account_number" : 8,
          "balance" : 48868,
          "firstname" : "Jan",
          "lastname" : "Burns",
          "age" : 35,
          "gender" : "M",
          "address" : "699 Visitation Place",
          "employer" : "Glasstep",
          "email" : "janburns@glasstep.com",
          "city" : "Wakulla",
          "state" : "AZ"
        },
        "sort" : [
          8
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "9",
        "_score" : null,
        "_source" : {
          "account_number" : 9,
          "balance" : 24776,
          "firstname" : "Opal",
          "lastname" : "Meadows",
          "age" : 39,
          "gender" : "M",
          "address" : "963 Neptune Avenue",
          "employer" : "Cedward",
          "email" : "opalmeadows@cedward.com",
          "city" : "Olney",
          "state" : "OH"
        },
        "sort" : [
          9
        ]
      }
    ]
  }
}

```

1）只有6条数据，这是因为存在分页查询；

使用`from`和`size`可以指定查询

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" },
    {"balance":"desc"}
  ],
  "from": 20,
  "size": 10
}
```



（2）详细的字段信息，参照： https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-search.html 

>
>
>The response also provides the following information about the search request:
>
>- `took` – how long it took Elasticsearch to run the query, in milliseconds
>- `timed_out` – whether or not the search request timed out
>- `_shards` – how many shards were searched and a breakdown of how many shards succeeded, failed, or were skipped.
>- `max_score` – the score of the most relevant document found
>- `hits.total.value` - how many matching documents were found
>- `hits.sort` - the document’s sort position (when not sorting by relevance score)
>- `hits._score` - the document’s relevance score (not applicable when using `match_all`)

#### 1.4.2、QueryDSL

##### 1、基本语法格式

ES 提供了一个可以执行查询的 Json 风格的 DSL （domain-specifig langurage 领域特定语言），这个被成为 Query DSL ，该查询语言非常全面。

一个查询语句 的典型结构

```json
{
    QUERY_NAME:{
        ARGUMENT:VALUE,
        ARGUMENT:VALUE.....
    }
}
```

如果针对某个字段，那么它的结构如下

```json
{
    QUERY_NAME:{
        FIELD_NAME:{
            ARGUMENT:VALUE,
            ARGUMENT:VALUE.....
        }
    }
}
```

```http
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
```

query 定义如何查询

- match_all 查询类型【代表查询所有的所有】，es中可以在 query中 组合非常多的查询类型完成复杂查询

- 除了 query 参数之外，我们也可以传递其他的参数改变查询结构，如 sort，size

- from + size 限定，完成分页功能

- sort排序，多字段排序，会在前序字段相等时后续字段内部排序，否则以前序为准


##### 2、返回部分字段

```http
 GET bank/_search
 {
   "query":{
     "match_all": {}
   },
   "sort": [
     {
       "balance": {
         "order": "desc"
       }
     }
   ],
   "from": 5,
   "size": 5,
   "_source": ["firstname","lastname"]
 }
```

查询结果：

```json
{
  "took" : 18,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "999",
        "_score" : null,
        "_source" : {
          "firstname" : "Dorothy",
          "balance" : 6087
        },
        "sort" : [
          999
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "998",
        "_score" : null,
        "_source" : {
          "firstname" : "Letha",
          "balance" : 16869
        },
        "sort" : [
          998
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "997",
        "_score" : null,
        "_source" : {
          "firstname" : "Combs",
          "balance" : 25311
        },
        "sort" : [
          997
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "996",
        "_score" : null,
        "_source" : {
          "firstname" : "Andrews",
          "balance" : 17541
        },
        "sort" : [
          996
        ]
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "995",
        "_score" : null,
        "_source" : {
          "firstname" : "Phelps",
          "balance" : 21153
        },
        "sort" : [
          995
        ]
      }
    ]
  }
}

```

##### 3、match【匹配查询】

* 基本类型（非字符串），精确控制

```json
GET bank/_search
{
  "query": {
    "match": {
      "account_number": "20"
    }
  }
}

```

match返回account_number=20的数据。

查询结果：

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "20",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 20,
          "balance" : 16418,
          "firstname" : "Elinor",
          "lastname" : "Ratliff",
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "email" : "elinorratliff@scentric.com",
          "city" : "Ribera",
          "state" : "WA"
        }
      }
    ]
  }
}

```







* 字符串，全文检索

```json
GET bank/_search
{
  "query": {
    "match": {
      "address": "kings"
    }
  }
}
```

全文检索，最终会按照评分进行排序，会对检索条件进行分词匹配。

查询结果：

```json
{
  "took" : 30,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 5.990829,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "20",
        "_score" : 5.990829,
        "_source" : {
          "account_number" : 20,
          "balance" : 16418,
          "firstname" : "Elinor",
          "lastname" : "Ratliff",
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "email" : "elinorratliff@scentric.com",
          "city" : "Ribera",
          "state" : "WA"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "722",
        "_score" : 5.990829,
        "_source" : {
          "account_number" : 722,
          "balance" : 27256,
          "firstname" : "Roberts",
          "lastname" : "Beasley",
          "age" : 34,
          "gender" : "F",
          "address" : "305 Kings Hwy",
          "employer" : "Quintity",
          "email" : "robertsbeasley@quintity.com",
          "city" : "Hayden",
          "state" : "PA"
        }
      }
    ]
  }
}

```

##### 4、match_phrase【短语匹配】

将需要匹配的值当成一整个单词（不分词）进行检索

```json
GET bank/_search
{
  "query": {
    "match_phrase": {
      "address": "mill road"
    }
  }
}
```

查处address中包含mill_road的所有记录，并给出相关性得分

查看结果：

```json
{
  "took" : 32,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 8.926605,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 8.926605,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```



match_phrase和Match的区别，观察如下实例：

```json
GET bank/_search
{
  "query": {
    "match_phrase": {
      "address": "990 Mill"
    }
  }
}
```

查询结果：

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 10.806405,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 10.806405,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```



使用match的keyword

```json
GET bank/_search
{
  "query": {
    "match": {
      "address.keyword": "990 Mill"
    }
  }
}
```

查询结果，一条也未匹配到

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

```



修改匹配条件为“990 Mill Road”

```json
GET bank/_search
{
  "query": {
    "match": {
      "address.keyword": "990 Mill Road"
    }
  }
}
```

查询出一条数据

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 6.5032897,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 6.5032897,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```



文本字段的匹配，使用keyword，匹配的条件就是要显示字段的全部值，要进行精确匹配的。

match_phrase是做短语匹配，只要文本中包含匹配条件，就能匹配到。

> 思考：猜测，短语匹配和使用match.key的精确匹配虽然看似没有用到分词匹配（倒排索引）。但实际执行时，也可以先用分词匹配筛选出可能的记录，在这些已查出的记录中，再进行短语匹配或精确匹配。

##### 5、multi_match【多字段匹配】

```http
GET bank/_search
{
  "query":{
    "multi_match": {
      "query": "mill",
      "fields": ["address","city"]
    }
  }
}
```

state或者address中包含mill，并且在查询过程中，会对于查询条件进行分词。

查询结果：

```json
{
  "took" : 28,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 5.4032025,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "472",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 472,
          "balance" : 25571,
          "firstname" : "Lee",
          "lastname" : "Long",
          "age" : 32,
          "gender" : "F",
          "address" : "288 Mill Street",
          "employer" : "Comverges",
          "email" : "leelong@comverges.com",
          "city" : "Movico",
          "state" : "MT"
        }
      }
    ]
  }
}

```

##### 6、bool 【复合查询】

bool 用来做复合查询

复合语句可以合并 任何 其他嵌套语句，包括复合语句，了解这一点是很重要的，这就意味着，复合语句之间可以互相嵌套，可以表达式非常复杂的逻辑

- must：必须达到 must 列举的所有条件

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "gender": "M"
          }
        },
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "must_not": [
        {"match":{
          "age":"18"
        }}
      ],
      "should": [
        {"match": {
          "lastname": "Wallace"
        }}
      ]
    }
  }
}
```
- must_not 必须不是指定的情况

```json
"must_not": [
        {"match":{
          "age":"18"
        }}
      ],
```
- should:应该达到 should 列举的条件，如果达到会增加相关文档的评分，并不会改变查询的结果，如果 query 中只有 should 且只有一种匹配规则，那么 should的条件就会被作为默认匹配条件而区改变查询结果

```json
 "should": [
        {"match": {
          "lastname": "Wallace"
        }}
      ]
```



![image-20201026052225903](image-20201026052225903.png)

实例：查询gender=m，并且address=mill的数据

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "gender": "M"
          }
        },
        {
          "match": {
            "address": "mill"
          }
        }
      ]
    }
  }
}
```

查询结果：

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 6.0824604,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      }
    ]
  }
}

```

**must_not：必须不是指定的情况**

实例：查询gender=m，并且address=mill的数据，但是age不等于38的

```json

GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "gender": "M"
          }
        },
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "age": "38"
          }
        }
      ]
    }
  }

```

查询结果：

```json
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 6.0824604,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```





**should：应该达到should列举的条件，如果到达会增加相关文档的评分，并不会改变查询的结果。如果query中只有should且只有一种匹配规则，那么should的条件就会被作为默认匹配条件二区改变查询结果。**

实例：匹配lastName应该等于Wallace的数据

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "gender": "M"
          }
        },
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "age": "18"
          }
        }
      ],
      "should": [
        {
          "match": {
            "lastname": "Wallace"
          }
        }
      ]
    }
  }
}
```



查询结果：

```json
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 12.585751,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 12.585751,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      }
    ]
  }
}

```

能够看到相关度越高，得分也越高。



##### 7、filter【结果过滤】

并不是所有的查询都需要产生分数，特别是那些仅仅用于 filtering（过滤）的文档，为了不计算分数 ES 会自动检查场景并且优化查询的执行

```http
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 10000,
            "lte": 20000
          }
        }
      }
    }
  }
}
```

这里先是查询所有匹配address=mill的文档，然后再根据10000<=balance<=20000进行过滤查询结果

查询结果：

```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 5.4032025,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      }
    ]
  }
}

```



Each `must`, `should`, and `must_not` element in a Boolean query is referred to as a query clause. How well a document meets the criteria in each `must` or `should` clause contributes to the document’s *relevance score*. The higher the score, the better the document matches your search criteria. By default, Elasticsearch returns documents ranked by these relevance scores.

 在boolean查询中，`must`, `should` 和`must_not` 元素都被称为查询子句 。 文档是否符合每个“must”或“should”子句中的标准，决定了文档的“相关性得分”。  得分越高，文档越符合您的搜索条件。  默认情况下，Elasticsearch返回根据这些相关性得分排序的文档。 

The criteria in a `must_not` clause is treated as a *filter*. It affects whether or not the document is included in the results, but does not contribute to how documents are scored. You can also explicitly specify arbitrary filters to include or exclude documents based on structured data.

`“must_not”子句中的条件被视为“过滤器”。` 它影响文档是否包含在结果中，  但不影响文档的评分方式。  还可以显式地指定任意过滤器来包含或排除基于结构化数据的文档。 



filter在使用过程中，并不会计算相关性得分：

```json
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "filter": {
        "range": {
          "balance": {
            "gte": "10000",
            "lte": "20000"
          }
        }
      }
    }
  }
}
```

查询结果：

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 213,
      "relation" : "eq"
    },
    "max_score" : 0.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "20",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 20,
          "balance" : 16418,
          "firstname" : "Elinor",
          "lastname" : "Ratliff",
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "email" : "elinorratliff@scentric.com",
          "city" : "Ribera",
          "state" : "WA"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "37",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 37,
          "balance" : 18612,
          "firstname" : "Mcgee",
          "lastname" : "Mooney",
          "age" : 39,
          "gender" : "M",
          "address" : "826 Fillmore Place",
          "employer" : "Reversus",
          "email" : "mcgeemooney@reversus.com",
          "city" : "Tooleville",
          "state" : "OK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "51",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 51,
          "balance" : 14097,
          "firstname" : "Burton",
          "lastname" : "Meyers",
          "age" : 31,
          "gender" : "F",
          "address" : "334 River Street",
          "employer" : "Bezal",
          "email" : "burtonmeyers@bezal.com",
          "city" : "Jacksonburg",
          "state" : "MO"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "56",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 56,
          "balance" : 14992,
          "firstname" : "Josie",
          "lastname" : "Nelson",
          "age" : 32,
          "gender" : "M",
          "address" : "857 Tabor Court",
          "employer" : "Emtrac",
          "email" : "josienelson@emtrac.com",
          "city" : "Sunnyside",
          "state" : "UT"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "121",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 121,
          "balance" : 19594,
          "firstname" : "Acevedo",
          "lastname" : "Dorsey",
          "age" : 32,
          "gender" : "M",
          "address" : "479 Nova Court",
          "employer" : "Netropic",
          "email" : "acevedodorsey@netropic.com",
          "city" : "Islandia",
          "state" : "CT"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "176",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 176,
          "balance" : 18607,
          "firstname" : "Kemp",
          "lastname" : "Walters",
          "age" : 28,
          "gender" : "F",
          "address" : "906 Howard Avenue",
          "employer" : "Eyewax",
          "email" : "kempwalters@eyewax.com",
          "city" : "Why",
          "state" : "KY"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "183",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 183,
          "balance" : 14223,
          "firstname" : "Hudson",
          "lastname" : "English",
          "age" : 26,
          "gender" : "F",
          "address" : "823 Herkimer Place",
          "employer" : "Xinware",
          "email" : "hudsonenglish@xinware.com",
          "city" : "Robbins",
          "state" : "ND"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "222",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 222,
          "balance" : 14764,
          "firstname" : "Rachelle",
          "lastname" : "Rice",
          "age" : 36,
          "gender" : "M",
          "address" : "333 Narrows Avenue",
          "employer" : "Enaut",
          "email" : "rachellerice@enaut.com",
          "city" : "Wright",
          "state" : "AZ"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "227",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 227,
          "balance" : 19780,
          "firstname" : "Coleman",
          "lastname" : "Berg",
          "age" : 22,
          "gender" : "M",
          "address" : "776 Little Street",
          "employer" : "Exoteric",
          "email" : "colemanberg@exoteric.com",
          "city" : "Eagleville",
          "state" : "WV"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "272",
        "_score" : 0.0,
        "_source" : {
          "account_number" : 272,
          "balance" : 19253,
          "firstname" : "Lilly",
          "lastname" : "Morgan",
          "age" : 25,
          "gender" : "F",
          "address" : "689 Fleet Street",
          "employer" : "Biolive",
          "email" : "lillymorgan@biolive.com",
          "city" : "Sunbury",
          "state" : "OH"
        }
      }
    ]
  }
}

```

**能看到所有文档的 "_score" : 0.0。**

##### 8、term

和match一样。匹配某个属性的值。全文检索字段用match，其他非text字段匹配用term。



>
>
>Avoid using the `term` query for [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/text.html) fields.
>
>避免对文本字段使用“term”查询
>
>By default, Elasticsearch changes the values of `text` fields as part of [analysis](). This can make finding exact matches for `text` field values difficult.
>
>默认情况下，Elasticsearch作为[analysis]()的一部分更改' text '字段的值。这使得为“text”字段值寻找精确匹配变得困难。 
>
>To search `text` field values, use the match.
>
>要搜索“text”字段值，请使用匹配。
>
>https://www.elastic.co/guide/en/elasticsearch/reference/7.6/query-dsl-term-query.html 

使用term匹配查询

```json
GET bank/_search
{
  "query": {
    "term": {
      "address": "mill Road"
    }
  }
}
```

查询结果：

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

```

一条也没有匹配到



而更换为match匹配时，能够匹配到32个文档

![image-20200502120921830](/image-20200502120921830.png)

也就是说，**全文检索字段用match，其他非text字段匹配用term**。

##### 9、aggregations（执行聚合）

聚合提供了从数据分组和提取数据的能力，最简单的聚合方法大致等于 SQL GROUP BY 和 SQL 聚合函数，在 ES 中，你有执行搜索返回 hits （命中结果） 并且同时返回聚合结果，把一个响应中的所有 hits（命中结果）分隔开的能力，这是非常强大有效的，你可以执行查询和多个聚合，并且在一个使用中得到各自的（任何一个的）返回结果，使用一次简洁简化的 API 来避免网络往返。


aggs：执行聚合。聚合语法如下：

```json
"aggs":{
    "aggs_name这次聚合的名字，方便展示在结果集中":{
        "AGG_TYPE聚合的类型(avg,term,terms)":{}
     }
}，
```



**address中包含mill的所有人的年龄分布以及平均年龄**

size:0不显示搜索数据

```http
GET bank/_search
{
  "query": {
    "match": {
      "address": "mill"
    }
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 10
      }
    },
    "ageAvg":{
      "avg": {
        "field": "age"
      }
    },
    "balanceAvg":{
      "avg": {
        "field": "balance"
      }
    }
    
  },
  "size":0
}
```

查询结果：

```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 38,
          "doc_count" : 2
        },
        {
          "key" : 28,
          "doc_count" : 1
        },
        {
          "key" : 32,
          "doc_count" : 1
        }
      ]
    },
    "ageAvg" : {
      "value" : 34.0
    },
    "balanceAvg" : {
      "value" : 25208.0
    }
  }
}

```

**按照年龄聚合，并且请求这些年龄段的这些人的平均薪资**

```json
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 100
      },
      "aggs": {
        "ageAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  },
  "size": 0
}
```

输出结果：

```json
{
  "took" : 49,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 31,
          "doc_count" : 61,
          "ageAvg" : {
            "value" : 28312.918032786885
          }
        },
        {
          "key" : 39,
          "doc_count" : 60,
          "ageAvg" : {
            "value" : 25269.583333333332
          }
        },
        {
          "key" : 26,
          "doc_count" : 59,
          "ageAvg" : {
            "value" : 23194.813559322032
          }
        },
        {
          "key" : 32,
          "doc_count" : 52,
          "ageAvg" : {
            "value" : 23951.346153846152
          }
        },
        {
          "key" : 35,
          "doc_count" : 52,
          "ageAvg" : {
            "value" : 22136.69230769231
          }
        },
        {
          "key" : 36,
          "doc_count" : 52,
          "ageAvg" : {
            "value" : 22174.71153846154
          }
        },
        {
          "key" : 22,
          "doc_count" : 51,
          "ageAvg" : {
            "value" : 24731.07843137255
          }
        },
        {
          "key" : 28,
          "doc_count" : 51,
          "ageAvg" : {
            "value" : 28273.882352941175
          }
        },
        {
          "key" : 33,
          "doc_count" : 50,
          "ageAvg" : {
            "value" : 25093.94
          }
        },
        {
          "key" : 34,
          "doc_count" : 49,
          "ageAvg" : {
            "value" : 26809.95918367347
          }
        },
        {
          "key" : 30,
          "doc_count" : 47,
          "ageAvg" : {
            "value" : 22841.106382978724
          }
        },
        {
          "key" : 21,
          "doc_count" : 46,
          "ageAvg" : {
            "value" : 26981.434782608696
          }
        },
        {
          "key" : 40,
          "doc_count" : 45,
          "ageAvg" : {
            "value" : 27183.17777777778
          }
        },
        {
          "key" : 20,
          "doc_count" : 44,
          "ageAvg" : {
            "value" : 27741.227272727272
          }
        },
        {
          "key" : 23,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 27314.214285714286
          }
        },
        {
          "key" : 24,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 28519.04761904762
          }
        },
        {
          "key" : 25,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 27445.214285714286
          }
        },
        {
          "key" : 37,
          "doc_count" : 42,
          "ageAvg" : {
            "value" : 27022.261904761905
          }
        },
        {
          "key" : 27,
          "doc_count" : 39,
          "ageAvg" : {
            "value" : 21471.871794871793
          }
        },
        {
          "key" : 38,
          "doc_count" : 39,
          "ageAvg" : {
            "value" : 26187.17948717949
          }
        },
        {
          "key" : 29,
          "doc_count" : 35,
          "ageAvg" : {
            "value" : 29483.14285714286
          }
        }
      ]
    }
  }
}
```
**查出所有年龄分布，并且这些年龄段中M的平均薪资和 F 的平均薪资以及这个年龄段的总体平均薪资**

```json
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 100
      },
      "aggs": {
        "genderAgg": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "balanceAvg": {
              "avg": {
                "field": "balance"
              }
            }
          }
        },
        "ageBalanceAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  },
  "size": 0
}
```

输出结果：

```json
{
  "took" : 119,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 31,
          "doc_count" : 61,
          "genderAgg" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "M",
                "doc_count" : 35,
                "balanceAvg" : {
                  "value" : 29565.628571428573
                }
              },
              {
                "key" : "F",
                "doc_count" : 26,
                "balanceAvg" : {
                  "value" : 26626.576923076922
                }
              }
            ]
          },
          "ageBalanceAvg" : {
            "value" : 28312.918032786885
          }
        }
      ]
        .......//省略其他
    }
  }
}

```

##### 10、nested

嵌套查询

数据类型概览

![image-20201225063939652](image-20201225063939652.png)

参考博客：https://elastic.blog.csdn.net/article/details/82950393

#### 1.4.3 Mapping

##### 1、字段类型

![image-20201026074813810](image-20201026074813810.png)

![image-20201026074841875](image-20201026074841875.png)

##### 2、映射

Mapping（映射）

Mapping 是用来定义一个文档（document）,以及他所包含的属性（field）是如何存储索引的，比如使用 mapping来定义的：

- 哪些字符串属性应该被看做全文本属性（full text fields）
- 那些属性包含数字，日期或者地理位置
- 文档的属性是否能被索引（_all 配置）
- 日期的格式
- 自定义映射规则来执行动态添加属性

查看 mapping 信息

GET bank/_mapping

  ```json
  {
    "bank" : {
      "mappings" : {
        "properties" : {
          "account_number" : {
            "type" : "long"
          },
          "address" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "age" : {
            "type" : "long"
          },
          "balance" : {
            "type" : "long"
          },
          "city" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "email" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "employer" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "firstname" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "gender" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "lastname" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "state" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          }
        }
      }
    }
  }
  ```

修改 mapping 信息

https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping-types.html

自动猜测的映射类型

![image-20201026075424198](image-20201026075424198.png)

##### 3、新版本改变

- 关系型数据库中两个数据表示是独立的，即使他们里面有相同名称的列也不影响使用，但ES中不是这样的。elasticsearch 是基于Lucene开发的搜索引擎，而ES中不同type下名称相同的filed 最终在Lucene,中的处理方式是一样的。

- 两个不同 type下的两个user_ name, 在ES同-个索引下其实被认为是同一一个filed,你必须在两个不同的type中定义相同的filed映射。否则，不同typpe中的相同字段称就会在处理中出现神突的情况，导致Lucene处理效率下降。
- 去掉type就是为了提高ES处理数据的效率。

ES 7.x 

URL 中的 type 参数 可选，比如索引一个文档不再要求提供文档类型

ES 8.X

不在支持 URL 中的 type 参数

解决：

1、将索引从多类型迁移到单类型，每种类型文档一个独立的索引

2、将已存在的索引下的类型数据，全部迁移到指定位置即可，详见数据迁移



> **Elasticsearch 7.x**
>
> - Specifying types in requests is deprecated. For instance, indexing a document no longer requires a document `type`. The new index APIs are `PUT {index}/_doc/{id}` in case of explicit ids and `POST {index}/_doc` for auto-generated ids. Note that in 7.0, `_doc` is a permanent part of the path, and represents the endpoint name rather than the document type.
> - The `include_type_name` parameter in the index creation, index template, and mapping APIs will default to `false`. Setting the parameter at all will result in a deprecation warning.
> - The `_default_` mapping type is removed.
>
> **Elasticsearch 8.x**
>
> - Specifying types in requests is no longer supported.
> - The `include_type_name` parameter is removed.

**1、创建映射**

创建索引并指定映射

```http
PUT /my_index
{
  "mappings":{
    "properties": {
      "age":{"type":"integer"},
      "email":{"type":"keyword"}
    }
  }
}
```

 输出：

```json
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "my_index"
}

```



###### 查看映射

```json
GET /my_index
```

输出结果：

```json
{
  "my_index" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "integer"
        },
        "email" : {
          "type" : "keyword"
        },
        "employee-id" : {
          "type" : "keyword",
          "index" : false
        },
        "name" : {
          "type" : "text"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1588410780774",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "ua0lXhtkQCOmn7Kh3iUu0w",
        "version" : {
          "created" : "7060299"
        },
        "provided_name" : "my_index"
      }
    }
  }
}
```

**2、添加新的字段映射**

```http
PUT /my_index/_mapping
{
  "properties":{
    "employeeid":{
      "type":"keyword",
      "index":false
    }
  }
}
```

这里的 "index": false，表明新增的字段不能被检索，只是一个冗余字段。

##### 3、更新映射

对于已经存在的映射字段，我们不能更新。更新必须创建新的索引，进行数据迁移

**4、数据迁移**

先创 new_twitter 的正确映射，然后使用如下方式进行数据迁移

```http
POST _reindex [固定写法]
{
  "source":{
    "index":"twitter"
  },
  "dest":{
    "index":"new_twitter"
  }
}
## 将旧索引的 type 下的数据进行迁移
POST _reindex
{
  "source": {
    "index":"twitter",
    "type":"tweet"
  },
  "dest":{
    "index":"new_twitter"
  }
}
```

参考官网：https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping-types.html

参数映射规则：https://www.elastic.co/guide/en/elasticsearch/reference/7.10/mapping-params.html#mapping-params

GET /bank/_search

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",//类型为account
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 1,
          "balance" : 39225,
          "firstname" : "Amber",
          "lastname" : "Duke",
          "age" : 32,
          "gender" : "M",
          "address" : "880 Holmes Lane",
          "employer" : "Pyrami",
          "email" : "amberduke@pyrami.com",
          "city" : "Brogan",
          "state" : "IL"
        }
      },
      ...
```



```
GET /bank/_search
```

![image-20200502174825233](/image-20200502174825233.png)

想要将年龄修改为integer

```json
PUT /newbank
{
  "mappings": {
    "properties": {
      "account_number": {
        "type": "long"
      },
      "address": {
        "type": "text"
      },
      "age": {
        "type": "integer"
      },
      "balance": {
        "type": "long"
      },
      "city": {
        "type": "keyword"
      },
      "email": {
        "type": "keyword"
      },
      "employer": {
        "type": "keyword"
      },
      "firstname": {
        "type": "text"
      },
      "gender": {
        "type": "keyword"
      },
      "lastname": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "state": {
        "type": "keyword"
      }
    }
  }
}
```

查看“newbank”的映射：

GET /newbank/_mapping

![image-20200502175901959](/image-20200502175901959.png)

能够看到age的映射类型被修改为了integer.



将bank中的数据迁移到newbank中

```json
POST _reindex
{
  "source": {
    "index": "bank",
    "type": "account"
  },
  "dest": {
    "index": "newbank"
  }
}
```

运行输出：

```json
#! Deprecation: [types removal] Specifying types in reindex requests is deprecated.
{
  "took" : 768,
  "timed_out" : false,
  "total" : 1000,
  "updated" : 0,
  "created" : 1000,
  "deleted" : 0,
  "batches" : 1,
  "version_conflicts" : 0,
  "noops" : 0,
  "retries" : {
    "bulk" : 0,
    "search" : 0
  },
  "throttled_millis" : 0,
  "requests_per_second" : -1.0,
  "throttled_until_millis" : 0,
  "failures" : [ ]
}
```



查看newbank中的数据

![image-20200502181432745](/image-20200502181432745.png)

删除重装es，扩充最大内存。

> free -m可查看虚拟机空闲内存。不足可关闭虚拟机后，在virtualbox中扩充虚拟机内存。

可以看到根据container id 停止 删除 容器时，不必把id输入完整。

![image-20210602204830403](/image-20210602204830403.png)

#### 1.4.4 分词

一个 **tokenizer**（分词器）接收一个字符流，将之分割为独立的 **tokens**（词元，通常是独立的单词），然后输出 **token** 流

列如，witespace tokenizer 遇到的空白字符时分割文本，它会将文本 “Quick brown fox” 分割为 【Quick brown fox】

该 **tokenizer** (分词器)还负责记录各个**term** (词条)的顺序或 **position** 位置(用于**phrase**短语和**word** proximity词近邻查询)，以及

**term** (词条)所代表的原始 **word** (单词)的start(起始)和end (结束)的 **character** **offsets** (字符偏移量) (用于 高亮显示搜索的内容)。

**Elasticsearch** 提供了很多内置的分词器，可以用来构建custom analyzers(自定义分词器)

关于分词器： https://www.elastic.co/guide/en/elasticsearch/reference/7.6/analysis.html 

使用指定分词器输出token流

```json
POST _analyze
{
  "analyzer": "standard",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

执行结果：

```json
{
  "tokens" : [
    {
      "token" : "the",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "2",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "<NUM>",
      "position" : 1
    },
    {
      "token" : "quick",
      "start_offset" : 6,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "brown",
      "start_offset" : 12,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
    {
      "token" : "foxes",
      "start_offset" : 18,
      "end_offset" : 23,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "jumped",
      "start_offset" : 24,
      "end_offset" : 30,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "over",
      "start_offset" : 31,
      "end_offset" : 35,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "the",
      "start_offset" : 36,
      "end_offset" : 39,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "lazy",
      "start_offset" : 40,
      "end_offset" : 44,
      "type" : "<ALPHANUM>",
      "position" : 8
    },
    {
      "token" : "dog's",
      "start_offset" : 45,
      "end_offset" : 50,
      "type" : "<ALPHANUM>",
      "position" : 9
    },
    {
      "token" : "bone",
      "start_offset" : 51,
      "end_offset" : 55,
      "type" : "<ALPHANUM>",
      "position" : 10
    }
  ]
}

```



##### 1、安装 ik 分词器



![image-20200502182929583](/image-20200502182929583.png)

所有的语言分词，默认使用的都是“Standard Analyzer”，但是这些分词器针对于中文的分词，并不友好。为此需要安装中文的分词器。



注意：不能用默认elasticsearch-plugin install xxx.zip 进行自动安装

https://github.com/medcl/elasticsearch-analysis-ik/releases 下载与 es对应的版本

安装后拷贝到 plugins 目录下

在前面安装的elasticsearch时，我们已经将elasticsearch容器的“/usr/share/elasticsearch/plugins”目录，映射到宿主机的“ /mydata/elasticsearch/plugins”目录下，所以比较方便的做法就是下载“/elasticsearch-analysis-ik-7.6.2.zip”文件，然后解压到该文件夹下即可。安装完毕后，需要重启elasticsearch容器。

 

如果不嫌麻烦，还可以采用如下的方式。

###### （1）查看elasticsearch版本号：

```shell
[root@hadoop-104 ~]# curl http://localhost:9200
{
  "name" : "0adeb7852e00",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "9gglpP0HTfyOTRAaSe2rIg",
  "version" : {
    "number" : "7.6.2",      #版本号为7.6.2
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ef48eb35cf30adf4db14086e8aabd07ef6fb113f",
    "build_date" : "2020-03-26T06:34:37.794943Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
[root@hadoop-104 ~]# 
```



###### （2）进入es容器内部plugin目录

* docker exec -it 容器id /bin/bash

```shell
[root@hadoop-104 ~]# docker exec -it elasticsearch /bin/bash
[root@0adeb7852e00 elasticsearch]# 
```

* wget  https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip

```shell
[root@0adeb7852e00 elasticsearch]# pwd
/usr/share/elasticsearch
#下载ik7.6.2
[root@0adeb7852e00 elasticsearch]# wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
```

* unzip 下载的文件

```shell
[root@0adeb7852e00 elasticsearch]# unzip elasticsearch-analysis-ik-7.6.2.zip -d ik
Archive:  elasticsearch-analysis-ik-7.6.2.zip
   creating: ik/config/
  inflating: ik/config/main.dic      
  inflating: ik/config/quantifier.dic  
  inflating: ik/config/extra_single_word_full.dic  
  inflating: ik/config/IKAnalyzer.cfg.xml  
  inflating: ik/config/surname.dic   
  inflating: ik/config/suffix.dic    
  inflating: ik/config/stopword.dic  
  inflating: ik/config/extra_main.dic  
  inflating: ik/config/extra_stopword.dic  
  inflating: ik/config/preposition.dic  
  inflating: ik/config/extra_single_word_low_freq.dic  
  inflating: ik/config/extra_single_word.dic  
  inflating: ik/elasticsearch-analysis-ik-7.6.2.jar  
  inflating: ik/httpclient-4.5.2.jar  
  inflating: ik/httpcore-4.4.4.jar   
  inflating: ik/commons-logging-1.2.jar  
  inflating: ik/commons-codec-1.9.jar  
  inflating: ik/plugin-descriptor.properties  
  inflating: ik/plugin-security.policy  
[root@0adeb7852e00 elasticsearch]#
#移动到plugins目录下
[root@0adeb7852e00 elasticsearch]# mv ik plugins/
```

* rm -rf *.zip

```
[root@0adeb7852e00 elasticsearch]# rm -rf elasticsearch-analysis-ik-7.6.2.zip 
```


确认是否安装好了分词器

##### 2、测试分词器

使用默认

```json
GET my_index/_analyze
{
   "text":"我是中国人"
}
```

请观察执行结果：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<IDEOGRAPHIC>",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "中",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "国",
      "start_offset" : 3,
      "end_offset" : 4,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    },
    {
      "token" : "人",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "<IDEOGRAPHIC>",
      "position" : 4
    }
  ]
}
```



```json
GET my_index/_analyze
{
   "analyzer": "ik_smart", 
   "text":"我是中国人"
}
```

输出结果：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "中国人",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    }
  ]
}

```



```json
GET my_index/_analyze
{
   "analyzer": "ik_max_word", 
   "text":"我是中国人"
}
```



输出结果：

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "中国人",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "中国",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "国人",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 4
    }
  ]
}

```

 

分词器

![image-20201026092255250](image-20201026092255250.png)

**3、自定义词库**

修改 /usr/share/elasticsearch/plugins/ik/config/中的 IKAnalyzer.cfg.xml

/usr/share/elasticsearch/plugins/ik/config

```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict"></entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
         <entry key="remote_ext_dict">http://192.168.56.10/es/fenci.txt</entry>
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

```

原来的xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict"></entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<!-- <entry key="remote_ext_dict">words_location</entry> -->
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

```

修改完成后，需要重启elasticsearch容器，否则修改不生效。

更新完成后，es只会对于新增的数据用更新分词。历史数据是不会重新分词的。如果想要历史数据重新分词，需要执行：

```shell
POST my_index/_update_by_query?conflicts=proceed
```





http://192.168.56.10/es/fenci.txt，这个是我们虚拟机nginx上资源的访问路径

在运行下面实例之前，需要安装nginx（安装方法见安装nginx），然后创建“fenci.txt”文件，内容如下：

使用vim或者echo配合>。输入尚硅谷，乔碧罗。

![image-20210602210558523](/image-20210602210558523.png)



```shell
echo "..." > /mydata/nginx/html/fenci.txt 
```

测试效果：

添加扩展分词前，无法识别乔碧罗。现在可以了。

![image-20210602210842931](/image-20210602210842931.png)



### 5、附录：安装Nginx

* 随便启动一个nginx实例，只是为了复制出配置

  ```shell
  docker run -p80:80 --name nginx -d nginx:1.10   
  ```

* 将容器内的配置文件拷贝到/mydata/nginx/conf/ 下

  ```shell
  mkdir -p /mydata/nginx/html
  mkdir -p /mydata/nginx/logs
  mkdir -p /mydata/nginx/conf
  docker container cp nginx:/etc/nginx/*  /mydata/nginx/conf/ 
  #由于拷贝完成后会在config中存在一个nginx文件夹，所以需要将它的内容移动到conf中
  mv /mydata/nginx/conf/nginx/* /mydata/nginx/conf/
  rm -rf /mydata/nginx/conf/nginx
  ```

* 终止原容器：

  ```shell
  docker stop nginx
  ```

* 执行命令删除原容器：

  ```shell
  docker rm nginx
  ```

* 创建新的Nginx，执行以下命令

  ```shell
  docker run -p 80:80 --name nginx \
   -v /mydata/nginx/html:/usr/share/nginx/html \
   -v /mydata/nginx/logs:/var/log/nginx \
   -v /mydata/nginx/conf/:/etc/nginx \
   -d nginx:1.10
  ```

* 设置开机启动nginx

  ```
  docker update nginx --restart=always
  ```

  

* 创建“/mydata/nginx/html/index.html”文件，测试是否能够正常访问

  ```
  echo '<h2>hello nginx!</h2>' >index.html
  ```

  访问：http://ngix所在主机的IP:80/index.html

### 1.5 java集成ES的工具  Elasticsearch - Rest - client

1、9300：TCP

Spring-data-elasticsearch:transport-api.jar

SpringBoot版本不同，`transport-api.jar` 不同，不能适配 es 版本

7.x 已经不在适合使用，8 以后就要废弃

**2、9200：HTTP**

JestClient 非官方，更新慢

RestTemplate:默认发送 HTTP 请求，ES很多操作都需要自己封装、麻烦

HttpClient：同上

Elasticsearch - Rest - Client：官方RestClient，封装了 ES 操作，API层次分明

最终选择 Elasticsearch - Rest - Client （elasticsearch - rest - high - level - client）



#### 1.5.1 SpringBoot 整合

1、Pom.xml

```xml
<!-- 导入es的 rest-high-level-client -->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.4.2</version>
</dependency>
```

为什么要导入这个？这个配置那里来的？

官网：https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-getting-started-maven.html

在spring-boot-dependencies中所依赖的ELK版本位6.8.7

```
    <elasticsearch.version>6.8.7</elasticsearch.version>
```

![image-20200511074437763](/image-20200511074437763.png)



需要在项目中将它改为7.6.2

```xml
    <properties>
        ...
        <elasticsearch.version>7.6.2</elasticsearch.version>
    </properties>
```

 

#### 1.5.2 Config配置

```java
/**
 * @author gcq
 * @Create 2020-10-26
 *
 * 1、导入配置
 * 2、编写配置，给容器注入一个RestHighLevelClient
 * 3、参照API 官网进行开发
 */
@Configuration
public class GulimallElasticsearchConfig {


    public static final RequestOptions COMMON_OPTIONS;
    static {
        RequestOptions.Builder builder = RequestOptions.DEFAULT.toBuilder();
//        builder.addHeader("Authorization", "Bearer " + TOKEN);
//        builder.setHttpAsyncResponseConsumerFactory(
//                new HttpAsyncResponseConsumerFactory
//                        .HeapBufferedResponseConsumerFactory(30 * 1024 * 1024 * 1024));
        COMMON_OPTIONS = builder.build();
    }



    @Bean
    public RestHighLevelClient esRestClient() {
        RestClientBuilder builder = null;
        builder = RestClient.builder(new HttpHost("192.168.56.10", 9200, "http"));

        RestHighLevelClient client = new RestHighLevelClient(builder);
//        RestHighLevelClient client = new RestHighLevelClient(
//                RestClient.builder(
//                        new HttpHost("localhost", 9200, "http"),
//                        new HttpHost("localhost", 9201, "http")));
        return client;
    }

}
```

官网：https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-getting-started-initialization.html

#### 1.5.3 使用

> 测试是否注入成功

https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-document-index.html 
```java
@Autowired
private RestHighLevelClient client;

@Test
public void contextLoads() {
    System.out.println(client);
}
```

> 测试是否能 添加 或更新数据

```java
/**
 * 添加或者更新
 * @throws IOException
 */
@Test
public void indexData() throws IOException {
    IndexRequest indexRequest = new IndexRequest("users");
    User user = new User();
    user.setAge(19);
    user.setGender("男");
    user.setUserName("张三");
    String jsonString = JSON.toJSONString(user);
    indexRequest.source(jsonString,XContentType.JSON);

    // 执行操作
    IndexResponse index = client.index(indexRequest, GulimallElasticsearchConfig.COMMON_OPTIONS);

    // 提取有用的响应数据
    System.out.println(index);
}
```

测试前：

![image-20200511111618183](/image-20200511111618183.png)

测试后：

![image-20200511112025327](/image-20200511112025327.png)

#### 2）测试获取数据

 https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-search.html 

```java
    @Test
    public void searchData() throws IOException {
        GetRequest getRequest = new GetRequest(
                "users",
                "_-2vAHIB0nzmLJLkxKWk");

        GetResponse getResponse = client.get(getRequest, RequestOptions.DEFAULT);
        System.out.println(getResponse);
        String index = getResponse.getIndex();
        System.out.println(index);
        String id = getResponse.getId();
        System.out.println(id);
        if (getResponse.isExists()) {
            long version = getResponse.getVersion();
            System.out.println(version);
            String sourceAsString = getResponse.getSourceAsString();
            System.out.println(sourceAsString);
            Map<String, Object> sourceAsMap = getResponse.getSourceAsMap();
            System.out.println(sourceAsMap);
            byte[] sourceAsBytes = getResponse.getSourceAsBytes();
        } else {

        }
    }
```



**测试复杂检索**

**搜索address中包含mill的所有人的年龄分布以及平均年龄，平均薪资**


```json
GET bank/_search
{
  "query": {
    "match": {
      "address": "Mill"
    }
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 10
      }
    },
    "ageAvg": {
      "avg": {
        "field": "age"
      }
    },
    "balanceAvg": {
      "avg": {
        "field": "balance"
      }
    }
  }
}
```



```java
 @Test
 /**
     * 复杂检索:在bank中搜索address中包含mill的所有人的年龄分布以及平均年龄，平均薪资
     * @throws IOException
     */
    public void searchTest() throws IOException {
        // 1、创建检索请求
        SearchRequest searchRequest = new SearchRequest();
        // 指定索引
        searchRequest.indices("bank");
        // 指定 DSL，检索条件
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        sourceBuilder.query(QueryBuilders.matchQuery("address", "mill"));

        //1、2 按照年龄值分布进行聚合
        TermsAggregationBuilder aggAvg = AggregationBuilders.terms("ageAgg").field("age").size(10);
        sourceBuilder.aggregation(aggAvg);

        //1、3 计算平均薪资
        AvgAggregationBuilder balanceAvg = AggregationBuilders.avg("balanceAvg").field("balance");
        sourceBuilder.aggregation(balanceAvg);


        System.out.println("检索条件" + sourceBuilder.toString());

        searchRequest.source(sourceBuilder);

        // 2、执行检索
        SearchResponse searchResponse = client.search(searchRequest, GulimallElasticsearchConfig.COMMON_OPTIONS);

        // 3、分析结果
        System.out.println(searchResponse.toString());

        // 4、拿到命中得结果
        SearchHits hits = searchResponse.getHits();
        // 5、搜索请求的匹配
        SearchHit[] searchHits = hits.getHits();
        // 6、进行遍历
        for (SearchHit hit : searchHits) {
            // 7、拿到完整结果字符串
            String sourceAsString = hit.getSourceAsString();
            // 8、转换成实体类
            Accout accout = JSON.parseObject(sourceAsString, Accout.class);
            System.out.println("account:" + accout );
        }

        // 9、拿到聚合
        Aggregations aggregations = searchResponse.getAggregations();
//        for (Aggregation aggregation : aggregations) {
//
//        }
        // 10、通过先前名字拿到对应聚合
        Terms ageAgg1 = aggregations.get("ageAgg");
        for (Terms.Bucket bucket : ageAgg1.getBuckets()) {
            // 11、拿到结果
            String keyAsString = bucket.getKeyAsString();
            System.out.println("年龄:" + keyAsString);
            long docCount = bucket.getDocCount();
            System.out.println("个数：" + docCount);
        }
        Avg balanceAvg1 = aggregations.get("balanceAvg");
        System.out.println("平均薪资：" + balanceAvg1.getValue());
        System.out.println(searchResponse.toString());
    }
```

结果：

```bash
Account(accountNumber=970, balance=19648, firstname=Forbes, lastname=Wallace, age=28, gender=M, address=990 Mill Road, employer=Pheast, email=forbeswallace@pheast.com, city=Lopezo, state=AK)
Account(accountNumber=136, balance=45801, firstname=Winnie, lastname=Holland, age=38, gender=M, address=198 Mill Lane, employer=Neteria, email=winnieholland@neteria.com, city=Urie, state=IL)
Account(accountNumber=345, balance=9812, firstname=Parker, lastname=Hines, age=38, gender=M, address=715 Mill Avenue, employer=Baluba, email=parkerhines@baluba.com, city=Blackgum, state=KY)
Account(accountNumber=472, balance=25571, firstname=Lee, lastname=Long, age=32, gender=F, address=288 Mill Street, employer=Comverges, email=leelong@comverges.com, city=Movico, state=MT)
年龄:38
个数:2
年龄:28
个数:1
年龄:32
个数:1
平均薪水:25208.0
```

总结：参考官网的API 和对应在 kibana 中发送的请求，在代码中通过调用对应API实现效果

官网：https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-search.html#java-rest-high-search-request-optional









# 2、商城业务 & 商品上架

检索技术栈评估：

以日志分析常用的ELK检索框架为例。

ELK  

Elasticsearch 用于存储检索数据。相对于Mysql的优势，基于内存检索，请求处理速度快。对全文检索支持更好。天生支持集群部署，可将海量数据分片存储。

logstach：收集数据

Kibana:视图化查看数据

![image-20201027120603889](image-20201027120603889.png)

上架的商品才可以在网站展示

上架的商品需要可以检索

### 2.1 商品的ES数据数据模型设计 -sku与spu信息的存储

ES是基于内存数据检索的，内存相对硬盘要昂贵许多。所以设计检索数据时要考虑下存储成本。非必要的字段或信息，不做存储或简化存储。比如，sku的库存情况，我们并不在ES里存储每个sku的数量，而是只设计一个bool字段，标记有无库存。不用于检索的属性，就在mapping里设计成非检索字段。

在检索界面展示的spu规格参数，是当前检索出的所有sku，分析出来的，是根据检索结果动态变化的。换句话说，检索位置的规格参数及数值，一定匹配了当前检索出的某些sku。

总之，我们检索sku时，一定也要查相应的spu规格参数。

而一个spu通常对应多个sku。在ES存储时有两种方案。

**方案一  单索引设计。**在每个sku信息里存储对应的spu规格参数等信息。优点是检索方便，检索sku可以直接拿到对应的spu规格参数信息。缺点是spu信息冗余，占用空间多。

```json
{
    skuId:1
    spuId:11
    skyTitile:华为xx
    price:999
    saleCount:99
    attr:[
        {尺寸:5},
        {CPU:高通945},
        {分辨率:全高清}
	]
}
缺点：如果每个sku都存储规格参数(如尺寸)，会有冗余存储，因为每个spu对应的sku的规格参数都一样
```

**方案二 双索引设计。**sku和spu分别建里索引。

```json
sku索引
{
    spuId:1
    skuId:11
}
spu索引
{
    spuId:11
    attrs:[
        {尺寸:5},
        {CPU:高通945},
        {分辨率:全高清}
	]
}
搜索 小米；可能有粮食，手机，电器
10000个sku，可能对应4000个spu，
分步，再根据4000个spu查询对应的属性；
只考虑spuid，通过esClient 传输了4000个id，long 8B*4000=32000B=32KB
10000个人检索，就是320MB
百万并发来了，检索一下，就是32GB。别的不说，只是网络传输就需要很长时间。


结论：如果将规格参数单独建立索引，以后可能会出现检索时出现大量数据网络传输的问题，会引起网络阻塞。

```

总之，两种方案都可以，在当前高并发场景下，我们采取空间换时间的策略。使用方案一。



### 2.1 商品检索数据的Mapping设计

1、检索的时候输入名字，是需要按照 sku 的 title进行全文检索的

2、有些字段不需要做为检索条件，所以设计为keyword，index为false，doc_values为false

2、nested类型的使用



```json
PUT product
{
    "mappings":{
        "properties": {
            "skuId":{
                "type": "long"
            },
            "spuId":{
                "type": "keyword" # 不可分词
            },
            "skuTitle": {
                "type": "text",
                "analyzer": "ik_smart" # 指定该属性使用中文分词器
            },
            "skuPrice": {
                "type": "keyword" # 保证精度问题
            },
            "skuImg":{
                "type": "keyword", # 图片地址不需要分词
                "index": false, # 不需要检索，这样保存数据时不用建索引，节省时间空间
                "doc_values": false # 不需要做分组（aggr）等分析 节省时间空间
            },
            "saleCount":{
                "type":"long"
            },
            "hasStock": {
                "type": "boolean"
            },
            "hotScore": {
                "type": "long"
            },
            "brandId": {
                "type": "long"
            },
            "catalogId": {
                "type": "long"
            },
            "brandName": {
                "type": "keyword",
                "index": false,
                "doc_values": false
            },
            "brandImg":{
                "type": "keyword",
                 "index": false,
                "doc_values": false
            },
            "catalogName": {
                "type": "keyword",
                "index": false,
                "doc_values": false
            },
            "attrs": {
                "type": "nested", # attrs
                "properties": {
                    "attrId": {
                        "type": "long"
                    },
                    "attrName": {
                        "type": "keyword",
                        "index": false,
                        "doc_values": false
                    },
                    "attrValue": {
                        "type": "keyword"
                    }
                }
            }
        }
    }
}
```

### 2.1 避免嵌套子对象扁平化的nested类型

 文档子对象默认为object类型，会被扁平化处理。由于扁平化处理，有多个子对象的话，子对象内属性的关联性被抹除了。

**object类型的扁平化**

存储有多个子对象的文档

如果我们依赖[字段自动映射](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dynamic-mapping.html),那么 `comments` 字段会自动映射为 `object` 类型。

由于所有的信息都在一个文档中,当我们查询时就没有必要去联合文章和评论文档,查询效率就很高。

但是当我们使用如下查询时,上面的文档也会被当做是符合条件的结果：

```bash
PUT /my_index/blogpost/1
{
  "title": "Nest eggs",
  "body":  "Making your money work...",
  "tags":  [ "cash", "shares" ],
  "comments": [ 
    {
      "name":    "John Smith",
      "comment": "Great article",
      "age":     28,
      "stars":   4,
      "date":    "2014-09-01"
    },
    {
      "name":    "Alice White",
      "comment": "More like this please",
      "age":     31,
      "stars":   5,
      "date":    "2014-10-22"
    }
  ]
}
```

由于所有的信息都在一个文档中,当我们查询时就没有必要去联合文章和评论文档,查询效率就很高。

但是当我们使用如下查询时,上面的文档也会被当做是符合条件的结果：

```json
GET /_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Alice" }},
        { "match": { "age":  28      }} 
      ]
    }
  }
}
```

正如我们在 [对象数组](https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#object-arrays) 中讨论的一样,出现上面这种问题的原因是 JSON 格式的文档被处理成如下的扁平式键值对的结构。

```json
{
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ],
  "comments.name":    [ alice, john, smith, white ],
  "comments.comment": [ article, great, like, more, please, this ],
  "comments.age":     [ 28, 31 ],
  "comments.stars":   [ 4, 5 ],
  "comments.date":    [ 2014-09-01, 2014-10-22 ]
}
```

`Alice` 和 31 、 `John` 和 `2014-09-01` 之间的相关性信息不再存在。虽然 `object` 类型 (参见 [内部对象](https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#inner-objects)) 在存储 *单一对象* 时非常有用,但对于对象数组的搜索而言,毫无用处。



**嵌套对象类型nested**

*嵌套对象* 就是来解决这个问题的。将 `comments` 字段类型设置为 `nested` 而不是 `object` 后,每一个嵌套对象都会被索引为一个 *隐藏的独立文档* ,举例如下:

```json
{ # 第一个 嵌套文档
  "comments.name":    [ john, smith ],
  "comments.comment": [ article, great ],
  "comments.age":     [ 28 ],
  "comments.stars":   [ 4 ],
  "comments.date":    [ 2014-09-01 ]
}
{ # 第二个 嵌套文档
  "comments.name":    [ alice, white ],
  "comments.comment": [ like, more, please, this ],
  "comments.age":     [ 31 ],
  "comments.stars":   [ 5 ],
  "comments.date":    [ 2014-10-22 ]
}
{ # 根文档 或者也可称为父文档
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ]
}
```

在独立索引每一个嵌套对象后,对象中每个字段的相关性得以保留。我们查询时,也仅仅返回那些真正符合条件的文档。

不仅如此,由于嵌套文档直接存储在文档内部,查询时嵌套文档和根文档联合成本很低,速度和单独存储几乎一样。

嵌套文档是隐藏存储的,我们不能直接获取。如果要增删改一个嵌套对象,我们必须把整个文档重新索引才可以。值得注意的是,查询的时候返回的是整个文档,而不是嵌套文档本身。

> 总结：object类型中，一个子对象数组，被扁平化。变成了一个个父对象的，数组类型的属性。
>
> nested类型，保证了子对象之间的独立性。用户可以对某个子对象进行检索。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html

### 2.2 商品上架接口编写

**需求分析：**

上架商品、将该商品相关属性上传到 Es中 为搜索服务做铺垫

<img src="image-20201028152104489.png" alt="image-20201028152104489" style="zoom:200%;" />

Controller

```java
/**
 * 商品上架
 * @param spuId
 * @return
 */
@RequestMapping("/{spuId}/up")
public R list(@PathVariable("spuId") Long spuId){
    spuInfoService.up(spuId);

    return R.ok();
}
```

Service

```java
@Override
public void up(Long spuId) {

    // 1、查出当前 spuid 对应的所有skuid信息，品牌的名字
    List<SkuInfoEntity> skus = skuInfoService.getSkuBySpuId(spuId);
    // 取出 Skuid 组成集合
    List<Long> skuIds = skus.stream().map(SkuInfoEntity::getSkuId).collect(Collectors.toList());

    //TODO 2、查询当前 spu 的所有可以用来被检索的规格参数属性
    List<ProductAttrValueEntity> attrValueEntities = attrValueService.baseAttrListforspu(spuId);

    // 返回所有 attrId
    List<Long> attrIds = attrValueEntities.stream().map(attr -> {
        return attr.getAttrId();
    }).collect(Collectors.toList());

    // 3、找出spu的attrIds中，哪些属于检索属性（pms_attr表中 search_type为1则是检索属性）
    List<Long> searchAttrIds = attrService.selectSearchAttrs(attrIds);

    // 找出spu的属性值中，哪些属于可检索的属性值（将查询出来的 attr_id 放到 set 集合中 用来判断 attrValueEntities 是否包含 attrId）
    Set<Long> idSet = new HashSet<>(searchAttrIds);

    List<SkuEsModel.Attrs> attrList = attrValueEntities.stream().filter(item -> {
        // 过滤掉不包含在 searchAttrIds集合中的元素
        return idSet.contains(item.getAttrId());
    }).map(item -> {
        SkuEsModel.Attrs attrs1 = new SkuEsModel.Attrs();
        // 属性对拷 将 ProductAttrValueEntity对象与 SkuEsModel.Attrs相同的属性进行拷贝
        BeanUtils.copyProperties(item, attrs1);
        return attrs1;
    }).collect(Collectors.toList());


    // 4、发送远程调用，库存系统查询是否有库存
    Map<Long, Boolean> stockMap = null;
    try {
        R r = wareFeignService.getSkuHasStock(skuIds);
        // key 是 SkuHasStockVo的 skuid value是 item的hasStock 是否拥有库存
        TypeReference<List<SkuHasStockVo>> typeReference = new TypeReference<List<SkuHasStockVo>>() {
        };
        stockMap = r.getData(typeReference).stream().collect(Collectors.toMap(SkuHasStockVo::getSkuId, item -> item.getHasStock()));
    } catch (Exception e) {
        log.error("库存服务异常：原因：{}",e);
        e.printStackTrace();
    }

    // 5、封装每个 sku 的信息
    Map<Long, Boolean> finalStockMap = stockMap;
    List<SkuEsModel> upProducts = skus.stream().map(sku -> {
        // 组装需要查询的数据
        SkuEsModel skuEsModel = new SkuEsModel();
        BeanUtils.copyProperties(sku, skuEsModel);
        skuEsModel.setSkuPrice(sku.getPrice());
        skuEsModel.setSkuImg(sku.getSkuDefaultImg());

        // 设置对应spu的规格参数
        skuEsModel.setAttrs(attrList);

        // 设置库存信息
        if (finalStockMap == null) {
            // 远程服务出现问题，任然设置为null
            skuEsModel.setHasStock(true);
        } else {
            skuEsModel.setHasStock(finalStockMap.get(sku.getSkuId()));
        }
        //TODO 2、热度频分 0
        skuEsModel.setHotScore(0L);

        //3、查询品牌名字和分类的信息
        BrandEntity brandEntity = brandService.getById(skuEsModel.getBrandId());
        skuEsModel.setBrandName(brandEntity.getName());
        skuEsModel.setBrandImg(brandEntity.getLogo());
        CategoryEntity categoryEntity = categoryService.getById(skuEsModel.getCatalogId());
        skuEsModel.setCatalogName(categoryEntity.getName());

        return skuEsModel;
    }).collect(Collectors.toList());

    // 5、将数据发给es进行保存，直接发送给 search服务
    R r = esFeignClient.productStatusUp(upProducts);
    if (r.getCode() == 0) {
        // 远程调用成功
        // TODO 6、修改当前 spu 的状态
        baseMapper.updateSpuStatus(spuId, ProductConstant.StatusEnum.SPU_UP.getCode());
    } else {
        // 远程调用失败
        //TODO 7、重复调用？ 接口冥等性 重试机制

        /**
         * Feign调用流程
         * 1、构造请求数据，将对象转成json
         * RequestTemplate template = buildTemplateFromArgs.create(argv);
         * 2、发送请求进行执行(执行成功会进行解码响应数据)
         *  executeAndDecode(template);
         * 3、执行请求会有重试机制
         *  while (true) {
         *       try {
         *         return executeAndDecode(template);
         *       } catch (RetryableException e) {
         *         try {
         *           retryer.continueOrPropagate(e);
         *         } catch (RetryableException th) {
         *              throw cause;
         *         }
         *         continute
         *	}
         */
    }
}
```

#### 2.2.1、发送远程调用，库存系统查询是否有库存

```java
/**
 * 根据 skuIds 查询是否有库存
 * @param skuIds
 * @return
 */
@PostMapping("/hasstock")
public R getSkuHasStock(@RequestBody List<Long> skuIds) {
    List<SkuHasStockVo> vos = wareSkuService.getSkusStock(skuIds);

    R ok = R.ok();
    ok.setData(vos);
    return ok;
}
```

Service

```java
@Override
public List<SkuHasStockVo> getSkusStock(List<Long> skuIds) {
    List<SkuHasStockVo> collect = skuIds.stream().map(skuId -> {
        SkuHasStockVo vo = new SkuHasStockVo();
        // 查询当前 sku 的总库存良
        // SELECT SUM(stock-stock_locked) FROM `wms_ware_sku` WHERE sku_id = 1
        Long count = baseMapper.getSkuStock(skuId);

        vo.setSkuId(skuId);
        vo.setHasStock(count == null?false:count>0);

        return vo;
    }).collect(Collectors.toList());
    return collect;
}
```

#### 2.2.2 将数据发送给 es保存 ，直接发送给 search服务

controller

```java
/**
     * 上架商品
     * @param skuEsModelList
     * @return
     */
    @PostMapping("/product")
    public R productStatusUp(@RequestBody List<SkuEsModel> skuEsModelList){
        boolean b = false;
        try {
            b = productSaveService.productStatusUp(skuEsModelList);
        } catch (Exception e) {
            log.error("ElasticSaveController商品上架错误：{}",e);
            return R.error(BizCodeEnume.PRODUCT_UP_EXCEPTION.getCode(),BizCodeEnume.PRODUCT_UP_EXCEPTION.getMsg());
        }
        if (!b) {
            return R.ok();
        } else {
            return R.error(BizCodeEnume.PRODUCT_UP_EXCEPTION.getCode(),BizCodeEnume.PRODUCT_UP_EXCEPTION.getMsg());
        }
    }

```

service

```java
@Override
public boolean productStatusUp(List<SkuEsModel> skuEsModelList) throws IOException {
    
    // 保存到es
    // 批量保存
    BulkRequest bulkRequest = new BulkRequest();
    for (SkuEsModel model : skuEsModelList) {
        // 1、构造请求 指定es索引
        IndexRequest indexRequest = new IndexRequest(EsConstant.PRODUCT_INDEX);
        // 1.1 指定id
        indexRequest.id(model.getSkuId().toString());
        // 1.2 将对象esmodel对象转换成 json 进行存储
        String s = JSON.toJSONString(model);
        // 1.3 设置文档源
        indexRequest.source(s, XContentType.JSON);

        bulkRequest.add(indexRequest);
    }
    
    BulkResponse bulk = restHighLevelClient.bulk(bulkRequest, GulimallElasticsearchConfig.COMMON_OPTIONS);

    // TODO 1、 如果批量错误
    boolean b = bulk.hasFailures();
    List<String> collect = Arrays.stream(bulk.getItems()).map(item -> {
        return item.getId();
    }).collect(Collectors.toList());
    log.error("商品上架完成：{}",collect);

    return b;
}
```



**业务流程总结：**

前端点击上架后，发送请求并带上参数 spuid

- 根据 `spuid` 查询 `pms_sku_info` 表得到商品相关属性

- ![image-20201028153205471](image-20201028153205471.png)

- 根据 `spuid` 查询 `pms_product_attr_value` 表得到可以用来检索的规格属性

- ![image-20201028153038796](image-20201028153038796.png)

  

- 从 `ProductAttrValueEntity` 中拿到所有的 attrId，根据 attrId 查询 `pms_attr` 查询检索的属性

- ![image-20201028155204710](image-20201028155204710.png)

- x根据 `pms_attr`  查询到检索属性后，用检索属性和 原先根据 `spuid` 查询 `pms_sku_info` 表得到商品相关属性进行比较，`pms_sku_info` 包含 从 `pms_attr` 字段attr_id 则数据保存否则过滤

- 根据 `skuIds` 去查询远程仓库中是否有库存 SELECT SUM(stock-stock_locked) FROM `wms_ware_sku` WHERE sku_id = 1

- 组装数据 设置 SkuEsModel 

- 发送给 es 保存





# 3、商城业务 & 首页整合

### 3.1 整合 thymeleaf 渲染首页

**需求分析：**

开发传统Java WEB工程时，我们可以使用JSP页面模板语言，但是在SpringBoot中已经不推荐使用了。SpringBoot支持如下页面模板语言

- Thymeleaf
- FreeMarker
- Velocity
- Groovy
- JSP

thymeleaf 官网：https://www.thymeleaf.org/

官网文档给出了，语法、相关标签如何使用的步骤，由于官网文档都是英文，英文文档阅读能力好的同学可以选择阅读，英文不好的同学可以选择中文文档进行学习，为此我在网上找到了相关的中文文档：文档：thymeleaf
链接：http://note.youdao.com/noteshare?id=7771a96e9031b30b91ed55c50528e918

#### 3.1.2 SpringBoot 整合 thymeleaf

Pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
    <!-- 版本后由SpringBoot进行管理-->
</dependency>
```

application.yml

```yaml
Spring:
  thymeleaf:
    cache: false # 开发过程建议关闭缓存
```

resource 目录介绍：

![image-20201029102830164](image-20201029102830164.png)



index.html中使用

```html
<!DOCTYPE html>
<!--使用thymeleaf中必须声明加上该行代码-->
<html lang="en"  xmlns:th="http://www.thymeleaf.org">
```

相关语法使用

```html
<!--和jsp相关表达式有点相似 具体使用过程参考文档-->
<ul>
  <li th:each="category : ${categorys}">
    <a href="#" class="header_main_left_a"
       th:attr="ctg-data=${category.catId}">
      <b th:text="${category.name}">家用电器</b></a>
  </li>
```

默认SpringBoot会直接去找 templates 下的 index.html 

![image-20201029103230550](image-20201029103230550.png)

### 3.2 整合dev-tools 渲染分类数据

**需求分析：**

我们需要在页面的侧边查询出分类的数据，并且选中一级分类数据后显示二级和三级分类数据

![image-20201029103544553](image-20201029103544553.png)

- 先获取一级分类数据
- 用户选中后在查询二级分类数据

Controller

```java
/**
 * 查询所有一级分类
 * @param model
 * @return
 */
@GetMapping({"/","/index.html"})
public String indexPage(Model model){

    // select * from category where parent_id = 0
    //TODO 1、查询所有的一级分类
    List<CategoryEntity> categoryEntityList = categoryService.getLevel1Categorys();

    model.addAttribute("categorys",categoryEntityList);
    // 查询所有的一级分类
    return "index";
}
```

Service

```java
@Override
public List<CategoryEntity> getLevel1Categorys() {
    // parent_cid为0则是一级目录
   return baseMapper.selectList(new QueryWrapper<CategoryEntity>().eq("parent_cid",0));

}
```

一级分类数据能显示后，着手处理二级分类数据获取

Controller

```java
/**
 * 查询完整分类数据
 * @return
 */
@ResponseBody
@RequestMapping("/index/catalog.json")
public Map<String, List<Catelog2Vo>> getCatelogJson() {
    Map<String, List<Catelog2Vo>> catelogJson = categoryService.getCatelogJson();
    return catelogJson;
}
```

Service

```java
@Override
public Map<String, List<Catelog2Vo>> getCatelogJson() {

    // 1、查询所有1级分类
    List<CategoryEntity> level1Categorys = getLevel1Categorys();
    // 2、封装数据封装成 map类型  key为 catId,value List<Catelog2Vo>
    Map<String, List<Catelog2Vo>> categoryList = level1Categorys.stream().collect(Collectors.toMap(k -> k.getCatId().toString(), v -> {
        // 1、每一个的一级分类，查询到这个一级分类的二级分类
        List<CategoryEntity> categoryEntities = baseMapper.selectList(new QueryWrapper<CategoryEntity>().eq("parent_cid", v.getCatId()));
        // 2、封装上面的结果
        List<Catelog2Vo> catelog2Vos = null;
        if (categoryEntities != null) {
            catelog2Vos = categoryEntities.stream().map(l2 -> {
                Catelog2Vo catelog2Vo = new Catelog2Vo(v.getCatId().toString(), null, l2.getCatId().toString(), l2.getName());
                // 1、查询当前二级分类的三级分类vo
                List<CategoryEntity> categoryEntities1 = baseMapper.selectList(new QueryWrapper<CategoryEntity>().eq("parent_cid", l2.getCatId()));
                if (categoryEntities1 != null ){
                    // 2、分装成指定格式
                    List<Catelog2Vo.catelog3Vo> catelog3VoList = categoryEntities1.stream().map(l3 -> {
                        Catelog2Vo.catelog3Vo catelog3Vo = new Catelog2Vo.catelog3Vo(l2.getCatId().toString(), l3.getCatId().toString(), l3.getName());
                        return catelog3Vo;
                    }).collect(Collectors.toList());
                    // 3、设置分类数据
                    catelog2Vo.setCatalog3List(catelog3VoList);
                 }
                return catelog2Vo;
            }).collect(Collectors.toList());
        }
        return catelog2Vos;
    }));
    // 2、封装数据
    return categoryList;
}
```

**上方代码具体业务逻辑：**

- 查询到一级分类，根据一级分类查询出二级分类并设置对应Vo对象，以此类推

<img src="Snipaste_2020-09-06_20-38-59.png" style="zoom:60%;" />



# 4、商城业务 & Nginx 域名访问

### 4.1 Nginx 搭建域名环境一（反向代理配置）

什么是 反向代理?

![image-20201029051037570](image-20201029051037570.png)

vi nginx.conf 文件后在底部有该条语句：

- 引入nginx下的 conf.d 下面的conf文件
- 那么我们开始在该目录下增加关于 谷粒商城的 nginx

![image-20201029045921936](image-20201029045921936.png)

拷贝原先默认的 conf

![image-20201029050207857](image-20201029050207857.png)

修改

![image-20201029050136324](image-20201029050136324.png)

> nginx代理过程，请求进来，根据请求url匹配一个location，再根据location匹配指定的server。



### 4.2 Nginx 搭建域名环境二 （负载均衡到网关）

 配置 UpStream

![image-20201029050256216](image-20201029050256216.png)



使用nginx实现负载平衡的最简单配置如下,官网地址：https://nginx.org/en/docs/http/load_balancing.html

```http
http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

同时在 本机上 hosts 文件上那个配置 域名映射

![image-20201029050625740](image-20201029050625740.png)

将请求转接给网关后，需要在网关配置

```yaml
- id: gulimall_host_route
  uri: lb://gulimall-product
  predicates:
    - Host=**.gulimall.com
```

最后放几张图方便理解哈

![image-20201029050828421](image-20201029050828421.png)



![image-20201029050901546](image-20201029050901546.png)



```shell
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

#event块
events {
    worker_connections  1024;
}

#http块
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;
    
    upstream gulimall{
	server 192.168.43.201:88;
    }    

    include /etc/nginx/conf.d/*.conf;
    ############################################################################
    #/etc/nginx/conf.d/default.conf 的server块
    server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}

}

```





# 5、性能压测 & 压力测试

**简介**

压力测试考察当前软硬件环境下系统所能承受住的最大负荷并帮助找出系统的瓶颈所在，压测都是为了系统。

在线上的处理能力和稳定性维持在一个标准范围内，做到心中有数。

使用压力测试，我们有希望找到很多种用其他测试方法更难发现的错误，有两种错误类型是：

内存泄漏、并发与同步。

有些服务在单线程工作模式下没问题，但一出现并发请求。就会出现各种错误，这些错误是我们在做功能测试时很难发现的。

> 有效的压力测试系统将应用以下这些关键条件：重复、并发、量级、随机变化

* jvisualvm

  用于检测java应用资源占用和垃圾回收的情况，cmd输入jvisualvm即可打开

* Jmeter

  下载后bin目录`jmeter.bat`打开

* 测试环境

  系统win10，CPU I710710U，内存16G





### 3. 三级分类(优化业务)

**优化前**

对二级菜单的每次遍历都需要查询数据库，浪费大量资源

```java
 //查询数据库：二级菜单
List<CategoryEntity> categoryEntities = this.list(new QueryWrapper<CategoryEntity>().eq("cat_level", 2));

        List<Catalog2Vo> catalog2Vos = categoryEntities.stream().map(categoryEntity -> {
            //查询数据库：三级菜单
            List<CategoryEntity> level3 = this.list(new QueryWrapper<CategoryEntity>().eq("parent_cid", categoryEntity.getCatId()));
            List<Catalog2Vo.Catalog3Vo> catalog3Vos = level3.stream().map(cat -> {
                return new Catalog2Vo.Catalog3Vo(cat.getParentCid().toString(), cat.getCatId().toString(), cat.getName());
            }).collect(Collectors.toList());
            Catalog2Vo catalog2Vo = new Catalog2Vo(categoryEntity.getParentCid().toString(), categoryEntity.getCatId().toString(), categoryEntity.getName(), catalog3Vos);
            return catalog2Vo;
        }).collect(Collectors.toList());
        Map<String, List<Catalog2Vo>> catalogMap = new HashMap<>();
        for (Catalog2Vo catalog2Vo : catalog2Vos) {
            List<Catalog2Vo> list = catalogMap.getOrDefault(catalog2Vo.getCatalog1Id(), new LinkedList<>());
            list.add(catalog2Vo);
            catalogMap.put(catalog2Vo.getCatalog1Id(),list);
        }
        return catalogMap;
```

**优化后**

仅查询一次数据库，剩下的数据通过遍历得到并封装

```java
  //优化业务逻辑，仅查询一次数据库
        List<CategoryEntity> categoryEntities = this.list();
        //查出所有一级分类
        List<CategoryEntity> level1Categories = getCategoryByParentCid(categoryEntities, 0L);
        Map<String, List<Catalog2Vo>> listMap = level1Categories.stream().collect(Collectors.toMap(k->k.getCatId().toString(), v -> {
            //遍历查找出二级分类
            List<CategoryEntity> level2Categories = getCategoryByParentCid(categoryEntities, v.getCatId());
            List<Catalog2Vo> catalog2Vos=null;
            if (level2Categories!=null){
                //封装二级分类到vo并且查出其中的三级分类
                catalog2Vos = level2Categories.stream().map(cat -> {
                    //遍历查出三级分类并封装
                    List<CategoryEntity> level3Catagories = getCategoryByParentCid(categoryEntities, cat.getCatId());
                    List<Catalog2Vo.Catalog3Vo> catalog3Vos = null;
                    if (level3Catagories != null) {
                        catalog3Vos = level3Catagories.stream()
                                .map(level3 -> new Catalog2Vo.Catalog3Vo(level3.getParentCid().toString(), level3.getCatId().toString(), level3.getName()))
                                .collect(Collectors.toList());
                    }
                    Catalog2Vo catalog2Vo = new Catalog2Vo(v.getCatId().toString(), cat.getCatId().toString(), cat.getName(), catalog3Vos);
                    return catalog2Vo;
                }).collect(Collectors.toList());
            }
            return catalog2Vos;
        }));
        return listMap;
```

### 4. Nginx动静分类

### 5.1 性能指标

#### 5.1.1 Jvm 内存模型

1、Jvm内存模型

![image-20201029112517466](image-20201029112517466.png)



![image-20201029113956043](image-20201029113956043.png)



#### 5.1.2 堆

所有的对象实例以及数组都要在堆上分配，堆时垃圾收集器管理的主要区域，也被称为 "GC堆"，也是我们优化最多考虑的地方

堆可以细分为：

- 新生代
  - Eden空间
  - From Survivor 空间
  - To Survivor 空间

- 老年代

- 永久代/原空间
  - Java8 以前永久代、受 JVM 管理、Java8 以后原空间，直接使用物理内存，因此默认情况下，原空间的大小仅受本地内存限制

垃圾回收

![image-20201029114153244](image-20201029114153244.png)

从 Java8 开始,HotSpot 已经完全将永久代（Permanent Generation）移除，取而代之的是一个新的区域 - 元空间（MetaSpace)

<img src="image-20201029114652885.png" alt="image-20201029114652885"  />



![image-20201029114218716](image-20201029114218716.png)

#### 5.1.3 jconsole 与 jvisualvm

jdk 的两个小工具 jconsole、jvisualvm（升级版本的 jconsole）。通过命令行启动、可监控本地和远程应用、远程应用需要配置

1、jvisualvm 能干什么

监控内存泄漏、跟踪垃圾回收、执行时内存、cpu分析、线程分析.....

![image-20201029120502383](image-20201029120502383.png)

运行：正在运行的线程

休眠：sleep

等待：wait

驻留：线程池里面的空闲线程

监视：组赛的线程、正在等待锁

2、安装插件方便查看 gc

cmd 启动 jvisualvm

工具->插件

![image-20201029121108492](image-20201029121108492.png)



如果503 错误解决

打开网址： https://visualvm.github.io/pluginscenters.html

cmd 查看自己的jdk版本，找到对应的

docker stats 查看相关命令

#### 5.1.4 监控指标

SQL 耗时越小越好、一般情况下微妙级别

命中率越高越好、一般情况下不能低于95%

锁等待次数越低越好、等待时间越短越好

##### 1、中间件指标

毫秒 

注：简单业务仅返回一个字符串

|                  压测内容                  | 压测线程数 | 吞吐量/ms              | 90%响应时间/ms | 99%响应时间/ms |
| :----------------------------------------: | ---------- | ---------------------- | -------------- | -------------- |
|                 **Nginx**                  | 50         | 1834                   | 11             | 40             |
|                **GateWay**                 | 50b        | 16577                  | 5              | 19             |
|                **简单服务**                | 50         | 17965                  | 5              | 10             |
|            **首页一级菜单渲染**            | 50         | 400（db,thymeleaf）    | 149            | 230            |
|            **首页渲染(开缓存)**            | 50         | 467                    | 128            | 209            |
| **首页渲染（开缓存、优化数据库，关日志）** | 50         | 1300                   | 60             | 107            |
|            **三级分类数据获取**            | 50         | 4.4（db）/18（优化后） | 16622          | 16832          |
|      **三级分类数据获取（优化业务）**      | 50         | 163                    | 410            | 585            |
|     **三级分类数据获取（redis缓存）**      | 50         | 553                    | 113            | 182            |
|            **首页全量数据获取**            | 50         | 10（静态资源）/12      | 6183           | 12358          |
|               Nginx+Gateway                | 50         |                        |                |                |
|              Gateway+简单服务              | 50         | 2685                   | 7              | 10             |
|                   全链路                   | 50         | 900                    | 73             | 971            |

中间件越多，性能损失越大，大多都损失在了网络交互

##### 2、性能指标

- **响应时间**（Response Time:RT）
- 响应时间指用户从客户端发起一个请求开始，到客户端接收到服务器端返回的响应结束，整个过程所耗费的时间
- **HPS**（Hits Per Second） ：每秒点击次数，单位是次/秒
- **TPS**（Transaction per Second）：系统每秒处理交易数，单位是笔/秒
- **QPS** (Query perSecond) :系统每秒处理查询次数，单位是次/秒。对于互联网业务中，如果某些业务有且仅有一个请求连接，那么TPS=QPS=HPS，一般情况下用TPS来衡量整个业务流程，用QPS来衡量接口查询次数，用HPS来表示对服务器单击请求。
- 无论TPS、QPS、HPS,此指标是衡量系统处理能力非常重要的指标，越大越好，根据经验，一般情况下:
  - 金融行业: 1000TPS~50000TPS, 不包括互联网化的活动
  - 保险行业: 1007P-00000PS， 不包括互联网化的活动
  - 制造行业: 10TPS~5000TPS
  - 互联网电子商务: 10000TPS~-100000TPS
  - 互联网中型网站: 1000TPS~50000TPS
  - 互联网小型网站: 5007PS~10000TPS

- **最大响应时间**(Max Response Time) 指用户发出请求或者指令到系统做出反应(响应)的最大时间。
- **最少响应时间** （Mininum ResponseTime）指用户发出请求或者指令到系统做出反应（响应）的最少时间
- **90%响应时间**（90% Response Time） 是指所有用户的响应时间进行排序、第90%的响应时间
- 从外部看、性能测试主要关注如下三个指
  - 吞吐量：每秒钟系统能够处理的请求数、任务数
  - 响应时间：服务处理一个请求或一个任务的耗时
  - 错误率：一批请求中结果出错的请求所占比例



吞吐量大:系统支持高并发，

响应时间：越短说明接口性能越好



### 5.2 JMeter

#### 5.2.1 JMeter 安装

jmeter官网：https://jmeter.apache.org/

#### 5.2.2 JMeter 压测示例

##### 1、添加线程组

![image-20201029084634498](image-20201029084634498.png)

##### 2、添加 HTTP 请求

![image-20201029085843220](image-20201029085843220.png)

##### 3、添加监听器

![image-20201029085942442](image-20201029085942442.png)

##### 4、启动压测&查看

汇总图

![image-20201029092357910](image-20201029092357910.png)

察看结果树

![image-20201029092436633](image-20201029092436633.png)

汇总报告

![image-20201029092454376](image-20201029092454376.png)



聚合报告

![image-20201029092542876](image-20201029092542876.png)



#### 5.2.3 JMeter Address Already in use 错误解决

windows本身提供的端口访问机制的问题。
Windows提供给TCP/IP 链接的端口为1024-5000，并且要四分钟来循环回收他们。就导致
我们在短时间内跑大量的请求时将端口占满了。

1.cmd中，用regedit命令打开注册表

2.在HKEY_ LOCAL MACHINE\SYSTEMCurrentControlSet\Services Tcpip\Parameters下，

​	1.右击parameters,添加一个新的DWORD,名字为MaxUserPort
​	2.然后双击 MaxUserPort,输入数值数据为65534,基数选择十进制(如果是分布式运行的话，控制机器和负载机器都需要这样操作哦)

3.修改配置完毕之后记得重启机器才会生效

TCPTimedWaitDelay:30



# 6、缓存与分布式锁

### 6.1 缓存

#### 6.1.2 缓存使用

为了系统性能的提升，我们一般都会将部分数据放入缓存中，加速访问，而 db 承担数据落盘工作

**哪些数据适合放入缓存？**

- **即时性、数据一致性要求不高的**
- **访问量大且更新频率不高的数据（读多、写少）**

举例：电商类应用、商品分类，商品列表等适合缓存并加一个失效时间（根据数据更新频率来定）后台如果发布一个商品、买家需要 5 分钟才能看到新商品一般还是可以接受的

![image-20201030190425556](image-20201030190425556.png)

伪代码

```java
data = cche.load(b); //从缓存中加载数据
if(data == null) {
    data = db.load(id); // 从数据库加载数据
    cache.put(id,data); // 保存到 cache中
}
return data;
```

注意：在开发中，凡是放到缓存中的数据我们都应该制定过期时间，使其可以在系统即使没有主动更新数据也能自动触发数据加载的流程，避免业务奔溃导致的数据永久不一致的问题



### 6.1.2 本地缓存

#### 

<img src="/Snipaste_2020-09-06_23-44-20.png" style="zoom:33%;" />

#### 1)  使用hashmap本地缓存

```java
    //测试本地缓存，通过hashmap
    private Map<String,Object> cache=new HashMap<>();

    public Map<String, List<Catalog2Vo>> getCategoryMap() {
          Map<String, List<Catalog2Vo>> catalogMap = (Map<String, List<Catalog2Vo>>) cache.get("catalogMap");
        //如果没有缓存，则从数据库中查询并放入缓存中
        if (catalogMap == null) {
            catalogMap = getCategoriesDb();
            cache.put("catalogMap",catalogMap);
        }
        return catalogMap;
    }
```

#### 1) 本地缓存面临问题

当有多个服务存在时，每个服务的缓存仅能够为本服务使用，这样每个服务都要查询一次数据库，并且当数据更新时只会更新单个服务的缓存数据，就会造成数据不一致的问题

![](/Snipaste_2020-09-06_23-54-28-1623061147164.png)

所有的服务都到同一个redis进行获取数据，就可以避免这个问题

![](/Snipaste_2020-09-07_18-03-17-1623061147165.png)

#### 

#### 

#### 6.1.3 整合 redis 作为缓存

##### 1、引入依赖

SpringBoot 整合 redis，查看SpringBoot提供的 starter

![image-20201031154148722](image-20201031154148722.png)

官网：https://docs.spring.io/spring-boot/docs/2.1.18.RELEASE/reference/html/using-boot-build-systems.html#using-boot-starter

**内存泄漏及解决办法**

当进行压力测试时后期后出现堆外内存溢出OutOfDirectMemoryError

产生原因：

1)、springboot2.0以后默认使用lettuce操作redis的客户端，它使用通信

2)、lettuce的bug导致netty堆外内存溢出

/**
 * TODO 产生堆外内存溢出 OutOfDirectMemoryError
 * 1、SpringBoot2.0以后默认使用 Lettuce作为操作redis的客户端，它使用 netty进行网络通信
 * 2、lettuce 的bug导致netty堆外内存溢出，-Xmx300m netty 如果没有指定堆内存移除，默认使用 -Xmx300m
 *      可以通过-Dio.netty.maxDirectMemory 进行设置
 *   解决方案 不能使用 -Dio.netty.maxDirectMemory调大内存
 *   1、升级 lettuce客户端，2、 切换使用jedis
 *   redisTemplate:
 *   lettuce、jedis 操作redis的底层客户端，Spring再次封装
 * @return
 */

解决方案：由于是lettuce的bug造成，不能直接使用-Dio.netty.maxDirectMemory去调大虚拟机堆外内存

1)、升级lettuce客户端。   2）、切换使用jedis

pom.xml

```xml
 <!--引入redis-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
            <!--不加载自身的 lettuce-->
            <exclusions>
                <exclusion>
                    <groupId>io.lettuce</groupId>
                    <artifactId>lettuce-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!--jedis-->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
```



##### 2、配置

application.yaml

```yaml
Spring:
  redis:
    host: 192.168.56.10
    port: 6379
```

RedisAutoConfig.java

![image-20201031154710108](image-20201031154710108.png)

##### 3、测试

```java
@Autowired
StringRedisTemplate stringRedisTemplate;

@Test
public void testStringRedisTemplate() {

    stringRedisTemplate.opsForValue().set("hello","world_" + UUID.randomUUID().toString());
    String hello = stringRedisTemplate.opsForValue().get("hello");
    System.out.println("之前保存的数据是：" + hello);
}
```

#### 3) 高并发下缓存失效问题

**缓存穿透**

指查询一个一定不存在的数据，由于缓存是不命中，将去查询数据库，但是数据库也无此记录，我们没有将这次查询的null写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义

风险：
利用不存在的数据进行攻击，数据库瞬时压力增大，最终导致崩溃

解决：
null结果缓存，并加入短暂过期时间

**缓存雪崩**

缓存雪崩是指在我们设置缓存时key采用了相同的过期时间，导致缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时
压力过重雪崩。

解决：
原有的失效时间基础上增加一个随机值，比如1-5分钟随机，这样每一个缓存的过期时间的重复率就会降低，就很难引发集体失效的事件。

**缓存击穿**

* 对于一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。

* 如果这个key在大量请求同时进来前正好失效，那么所有对这个key的数据查询都落到db，我们称为缓存击穿。

解决：
加锁。大量并发只让一个去查，其他人等待，查到以后释放锁，其他人获取到锁，先查缓存，就会有数据，不用去db

#### 高并发下缓存失效问题 

##### 缓存穿透

![](image-20201031163704355.png)

##### 缓存雪崩

![image-20201031163949881](image-20201031163949881.png)

##### 缓存击穿

![image-20201031164021131](image-20201031164021131.png)

#### 4) 加锁解决缓存击穿问题 阶段一

将查询db的方法加锁，这样在同一时间只有一个方法能查询数据库，就能解决缓存击穿的问题了

```java
public Map<String, List<Catalog2Vo>> getCategoryMap() {
    // 给缓存中放 json 字符串、拿出的是 json 字符串，还要逆转为能用的对象类型【序列化和反序列化】

    // 1、加入缓存逻辑，缓存中放的数据是 json 字符串
    // JSON 跨语言，跨平台兼容
        ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
        String catalogJson = ops.get("catalogJson");
        if (StringUtils.isEmpty(catalogJson)) {
            // 2、缓存没有，从数据库中查询
            System.out.println("缓存不命中，准备查询数据库。。。");
            Map<String, List<Catalog2Vo>> categoriesDb = getCategoriesDb();
            // 3、查询到数据，将数据转成 JSON 后放入缓存中
            String toJSONString = JSON.toJSONString(categoriesDb);
            ops.set("catalogJson",toJSONString);
            return categoriesDb;
        }
        System.out.println("缓存命中。。。。");
        Map<String, List<Catalog2Vo>> listMap = JSON.parseObject(catalogJson, new TypeReference<Map<String, List<Catalog2Vo>>>() {});
        return listMap;
    }

 private synchronized Map<String, List<Catalog2Vo>> getCategoriesDb() {
        String catalogJson = stringRedisTemplate.opsForValue().get("catalogJson");
        if (StringUtils.isEmpty(catalogJson)) {
            System.out.println("查询了数据库");
      		。。。。。
            return listMap;
        }else {
            Map<String, List<Catalog2Vo>> listMap = JSON.parseObject(catalogJson, new TypeReference<Map<String, List<Catalog2Vo>>>() {});
            return listMap;
        }
    }
```

#### 5)  锁时序问题

在上述方法中，我们将业务逻辑中的`确认缓存没有`和`查数据库`放到了锁里，但是最终控制台却打印了两次查询了数据库。这是因为在释放锁后，将结果放入缓存的这段时间（可能因为网络波动等原因变长）里，有其他线程拿到锁确认缓存没有，又再次查询了数据库，因此我们要将`结果放入缓存`也进行加锁。

> 在这里，将数据放入redis，就是影响双重检查锁判断的数据状态的操作。不把更新双重检查锁数据状态的代码，放入同步代码，就会导致锁失效。
>
> 这提示我们，过早释放锁（尤其是没有更新双重检查锁使用的数据状态），很可能导致锁失效，导致多个线程重复执行业务代码。

<img src="/Snipaste_2020-09-07_17-23-41.png" style="zoom:38%;" />

优化代码逻辑后

```java
public Map<String, List<Catalog2Vo>> getCategoryMap() {
    ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
        String catalogJson = ops.get("catalogJson");
        if (StringUtils.isEmpty(catalogJson)) {
            System.out.println("缓存不命中，准备查询数据库。。。");
            synchronized (this) {
                String synCatalogJson = stringRedisTemplate.opsForValue().get("catalogJson");
                if (StringUtils.isEmpty(synCatalogJson)) {
                    Map<String, List<Catalog2Vo>> categoriesDb= getCategoriesDb();
                    String toJSONString = JSON.toJSONString(categoriesDb);
                    ops.set("catalogJson", toJSONString);
                    return categoriesDb;
                }else {
                    Map<String, List<Catalog2Vo>> listMap = JSON.parseObject(synCatalogJson, new TypeReference<Map<String, List<Catalog2Vo>>>() {});
                    return listMap;
                }
            }
        }
        System.out.println("缓存命中。。。。");
        Map<String, List<Catalog2Vo>> listMap = JSON.parseObject(catalogJson, new TypeReference<Map<String, List<Catalog2Vo>>>() {});
        return listMap;
}
```

优化后多线程访问时仅查询一次数据库





### 6.4 分布式锁

##### 分布式锁

当分布式项目在高并发下也需要加锁，但本地锁只能锁住当前服务，这个时候就需要分布式锁

![image-20201031112235353](image-20201031112235353.png)

#### 分布式锁原理与应用

##### 分布式锁基本原理

![image-20201031122557660](image-20201031122557660.png)

**理解：**就先当1000个人去占一个厕所，厕所只能有一个人占到这个坑，占到这个坑其他人就只能在外面等待，等待一段时间后可以再次来占坑，业务执行后，释放锁，那么其他人就可以来占这个坑

**阶段一**

<img src="/Snipaste_2020-09-08_18-51-36.png" style="zoom: 33%;" />

```java
	public Map<String, List<Catalog2Vo>> getCatalogJsonDbWithRedisLock() {
        //阶段一
        Boolean lock = stringRedisTemplate.opsForValue().setIfAbsent("lock", "111");
        //获取到锁，执行业务
        if (lock) {
            Map<String, List<Catalog2Vo>> categoriesDb = getCategoryMap();
            //删除锁，如果在此之前报错或宕机会造成死锁
            stringRedisTemplate.delete("lock");
            return categoriesDb;
        }else {
            //没获取到锁，等待100ms重试
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return getCatalogJsonDbWithRedisLock();
        }
    }

public Map<String, List<Catalog2Vo>> getCategoryMap() {
        ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
        String catalogJson = ops.get("catalogJson");
        if (StringUtils.isEmpty(catalogJson)) {
            System.out.println("缓存不命中，准备查询数据库。。。");
            Map<String, List<Catalog2Vo>> categoriesDb= getCategoriesDb();
            String toJSONString = JSON.toJSONString(categoriesDb);
            ops.set("catalogJson", toJSONString);
            return categoriesDb;
        }
        System.out.println("缓存命中。。。。");
        Map<String, List<Catalog2Vo>> listMap = JSON.parseObject(catalogJson, new TypeReference<Map<String, List<Catalog2Vo>>>() {});
        return listMap;
    }
```

问题：
1、setnx占好了位，业务代码异常或者程序在页面过程中宕机。没有执行删除锁逻辑，这就造成了死锁

解决：设置锁的自动过期，即使没有删除，会自动删除

> 如果死锁，等待线程就会递归调用的不断重试，函数栈帧持续累计，无法释放，就会导致OOM



**阶段二**

<img src="Snipaste_2020-09-08_18-56-06.png" style="zoom:33%;" />

```java
   public Map<String, List<Catalog2Vo>> getCatalogJsonDbWithRedisLock() {
        Boolean lock = stringRedisTemplate.opsForValue().setIfAbsent("lock", "111");
        if (lock) {
            // 加锁成功..执行业务
            // 设置过期时间
            stringRedisTemplate.expire("lock", 30, TimeUnit.SECONDS);
            Map<String, List<Catalog2Vo>> categoriesDb = getCategoryMap();
            stringRedisTemplate.delete("lock");
            return categoriesDb;
        }else {
            // 加锁失败，重试 synchronized()
            // 休眠100ms重试
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return getCatalogJsonDbWithRedisLock();
        }
    }
```

问题：
1、setnx设置好，正要去设置过期时间，宕机。又死锁了。
解决：
设置过期时间和占位必须是原子的。redis支持使用setnx ex命令



##### 分布式锁演进 - 阶段三

<img src="/Snipaste_2020-09-08_19-03-23.png" style="zoom:33%;" />

```java
public Map<String, List<Catalog2Vo>> getCatalogJsonDbWithRedisLock() {
    //加锁的同时设置过期时间，二者是原子性操作
    Boolean lock = stringRedisTemplate.opsForValue().setIfAbsent("lock", "1111",5, TimeUnit.SECONDS);
    if (lock) {
        // 加锁成功..执行业务
        Map<String, List<Catalog2Vo>> categoriesDb = getCategoryMap();
        //模拟超长的业务执行时间
        try {
            Thread.sleep(6000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //删除锁
        stringRedisTemplate.delete("lock");
        return categoriesDb;
    }else {
      // 休眠100ms重试
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return getCatalogJsonDbWithRedisLock();
    }
}
```

问题：
1、删除锁直接删除？？？
如果由于业务时间很长，锁自己过期了，我们直接删除，有可能把别人正在持有的锁删除了。
解决：
占锁的时候，值指定为uuid，每个人匹配是自己的锁才删除。

##### 分布式锁演进 - 阶段四

<img src="/Snipaste_2020-09-08_19-20-43.png" style="zoom: 33%;" />

图解：校验锁后，去删除锁时，锁恰好自己已过期，可能会删掉其他线程的锁

![image-20201031130547173](image-20201031130547173.png)



```java
 public Map<String, List<Catalog2Vo>> getCatalogJsonDbWithRedisLock() {
        String uuid = UUID.randomUUID().toString();
        ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
     	//为当前锁设置唯一的uuid，只有当uuid相同时才会进行删除锁的操作
        Boolean lock = ops.setIfAbsent("lock", uuid,5, TimeUnit.SECONDS);
        if (lock) {
            Map<String, List<Catalog2Vo>> categoriesDb = getCategoryMap();
            String lockValue = ops.get("lock");
            if (lockValue.equals(uuid)) {
                //模拟校验锁后，去删除锁时，锁恰好自己已过期，可能会删掉其他线程的锁
                //try {
                //    Thread.sleep(6000);
                //} catch (InterruptedException e) {
                //    e.printStackTrace();
                //}
                // 删除我自己的锁
                stringRedisTemplate.delete("lock");
            }
            return categoriesDb;
        }else {
            // 休眠100ms重试
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return getCatalogJsonDbWithRedisLock();
        }
    }
```

问题：
1、如果正好判断是当前值，正要删除锁的时候，锁已经过期，别人已经设置到了新的值。那么我们删除的是别人的锁
解决：
删除锁必须保证原子性。使用redis+Lua脚本完成


##### 分布式锁演进 - 阶段五 最终形态



<img src="Snipaste_2020-09-08_19-35-01.png" style="zoom:33%;" />

代码：

```java
 String uuid = UUID.randomUUID().toString();
        // 加锁+设置过期时间=原子操作。必须是同步的。
        Boolean lock=redisTemplate.opsForValue().setIfAbsent("lock",uuid,300,TimeUnit.SECONDS);
        if (lock) {
            System.out.println("获取分布式锁成功");
            // 加锁成功..执行业务
            Map<String,List<Catelog2Vo>> dataFromDb;
            try {
                dataFromDb = getDataFromDB();
            } finally {
                // 获取值对比+对比成功删除=原子操作 lua脚本解锁
                String script = "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
                //删除锁
                Long lock1 = redisTemplate.execute(new DefaultRedisScript<Long>(script, Long.class), Arrays.asList("lock"), uuid);
            }
            return dataFromDb;
        } else {
            // 休眠200ms重试
            System.out.println("获取分布式锁失败，等待重试");
            try { TimeUnit.MILLISECONDS.sleep(200); } catch (InterruptedException e) { e.printStackTrace(); }
            return getCatelogJsonFromDbWithRedisLock();
        }
```

总结：

1. 两个核心操作，原子加锁，原子解锁。通过上面的代码，保证加锁【占位+过期时间】和删除锁【判断+删除】操作的原子性。

2. 更难的事情，锁的自动续期

问题：

- 分布式加锁解锁都是这段代码，我们可以封装成工具类，在其他业务复用。
- 分布式锁有更专业的框架Redisson。
- 上面的递归调用阻塞方式容易出现OOM问题，为了方便演示OOM。把递归前的sleep去掉，用压测工具压这个服务接口。很容易出现OOM。



#### 更专业的分布式锁实现框架 - Redisson

##### 1、简介&整合

官网文档上详细说明了 不推荐使用 setnx来实现分布式锁，应该参考 the Redlock algorithm 的实现

![image-20201101050725534](image-20201101050725534.png)

 the Redlock algorithm：https://redis.io/topics/distlock

在Java 语言环境下使用 Redisson

![image-20201101050924914](image-20201101050924914.png)

github：https://github.com/redisson/redisson

有对应的 中文文档

在 Maven 仓库中搜索也能搜索出 Redisson

![image-20201101051157803](image-20201101051157803.png)

Pom

```xml
<!--以后使用 redisson 作为分布锁，分布式对象等功能-->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.12.0</version>
</dependency>
```
开启配置https://github.com/redisson/redisson/wiki/2.-%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95

```java
@Configuration
public class MyRedisConfig {

    @Value("${ipAddr}")
    private String ipAddr;

    // redission通过redissonClient对象使用 // 如果是多个redis集群，可以配置
    @Bean(destroyMethod = "shutdown")
    public RedissonClient redisson() {
        Config config = new Config();
        // 创建单例模式的配置
        config.useSingleServer().setAddress("redis://" + ipAddr + ":6379");
        return Redisson.create(config);
    }
}

```



##### 2、 可重入锁

避免死锁

1. 通过自动设置默认过期时间，避免意外宕机等情况导致死锁
2. 如果业务执行时间长，通过自动续期，避免锁自动过期后。（看门狗）

```java
@RequestMapping("/hello")
@ResponseBody
public String hello(){
    // 1、获取一把锁，只要锁得名字一样，就是同一把锁
    RLock lock = redission.getLock("my-lock");

    // 2、加锁
    lock.lock(); // 阻塞式等待，默认加的锁都是30s时间
    // 1、锁的自动续期（看门狗），如果业务超长，运行期间自动给锁续上新的30s，不用担心业务时间长，锁自动过期后被删掉
    // 2、加锁的业务只要运行完成，就不会给当前锁续期，即使不手动解锁，锁默认会在30s以后自动删除

    lock.lock(10, TimeUnit.SECONDS); //10s 后自动删除
    //问题 lock.lock(10, TimeUnit.SECONDS) 在锁时间到了后，不会自动续期
    // 1、如果我们传递了锁的超时时间，就发送给 redis 执行脚本，进行占锁，默认超时就是我们指定的时间
    // 2、如果我们未指定锁的超时时间，就是用 30 * 1000（LockWatchchdogTimeout看门狗的默认时间）、
    //      只要占锁成功，就会启动一个定时任务，【重新给锁设置过期时间，新的过期时间就是看门狗的默认时间】,每隔10s就自动续期
    //      internalLockLeaseTime【看门狗时间】/3,10s

    //最佳实践
    // 1、lock.lock(30, TimeUnit.SECONDS);省掉了整个续期操作，手动解锁

    try {
        System.out.println("加锁成功，执行业务..." + Thread.currentThread().getId());
        Thread.sleep(3000);
    } catch (Exception e) {

    } finally {
        // 解锁 将设解锁代码没有运行，reidsson会不会出现死锁
        System.out.println("释放锁...." + Thread.currentThread().getId());
        lock.unlock();
    }

    return "hello";
}
```

进入到 `Redisson` Lock 源码

1、进入 `Lock` 的实现 发现 他调用的也是 `lock` 方法参数  时间为 -1

![image-20201101051659465](image-20201101051659465.png)

2、再次进入 `lock` 方法

发现他调用了 tryAcquire，而且是通过while true死循环实现自旋，不用担心OOM。

![image-20201101051925487](image-20201101051925487.png)

3、进入 tryAcquire

![image-20201101052008724](image-20201101052008724.png)

4、里头调用了 tryAcquireAsync

这里就是看门狗，所以看门狗也是在while 死循环里面持续判断的。这里判断 laseTime != -1 就与刚刚的第一步传入的值有关系

![image-20201101052037959](image-20201101052037959.png)

5、进入到 `tryLockInnerAsync` 方法

![image-20201101052158592](image-20201101052158592.png)



6、`internalLockLeaseTime` 这个变量是锁的默认时间

这个变量在构造的时候就赋初始值

![image-20201101052346059](image-20201101052346059.png)

7、最后查看 `lockWatchdogTimeout` 变量

也就是30秒的时间

![image-20201101052428198](image-20201101052428198.png)

8、自动续期是通过一个定时器实现的。

一个方法在定时器里的回调里，重新调用自己。

![image-20210608205950098](/image-20210608205950098.png)

![image-20210608205454637](/image-20210608205454637.png)

> 总结：专业框架相对于我们自己实现的锁的几点不同
>
> 1. 避免死锁的方式，通过设置过期时间和自动续期
> 2. 阻塞等待获取锁的时候，是用while死循环方式实现自旋。不用担心递归调用带来的OOM风险。
> 3. 实现的都是可重入锁。避免两个要求锁的函数，在嵌套调用时出现死锁问题。

> 思考：当时引入锁是防止缓存击穿问题，如果出现大量请求在请求等待锁，是不是也会导致机器资源耗尽？
>
> 如果真的出现这样大量请求，不引入锁，服务器压力更大，每个请求去请求数据库，处理时间都变长。都去访问数据库，还会直接压垮数据库。
>
> 毕竟只要服务器可以“接住”这些请求，坚持到缓存里有数据，积压的请求很快就处理掉了。
>
> 业务服务器引入锁机制，保护限制对数据库的访问。

##### 最佳实战：

lock.lock(30, TimeUnit.SECONDS);

虽然默认的lock()方法有自动续期。

但我们还是推荐使用指定过期时间的方法，手动解锁。这样我们控制过期时间，而且省掉了整个续期操作，如果锁过期了，业务还没执行完，说明业务有问题了（数据库宕掉了等）。

> 如果线程A的锁过期时间设置的太短，业务完成前过期了，线程B拿到了锁。然后线程A业务完成后去解锁，会抛异常。因为锁里记录了是哪个线程创建的，只允许创建它的线程删除。

![image-20210608211510995](/image-20210608211510995.png)

##### (2) 可重入锁（Reentrant Lock）

分布式锁：github.com/redisson/redisson/wiki/8.-分布式锁和同步器

A调用B。AB都需要同一锁，此时可重入锁就可以重入，A就可以调用B。不可重入锁时，A调用B将死锁

```java
// 参数为锁名字
RLock lock = redissonClient.getLock("CatalogJson-Lock");//该锁实现了JUC.locks.lock接口
lock.lock();//阻塞等待
// 解锁放到finally // 如果这里宕机：有看门狗，不用担心
lock.unlock();

```

基于Redis的Redisson分布式可重入锁RLock Java对象实现了java.util.concurrent.locks.Lock接口。同时还提供了异步（Async）、反射式（Reactive）和RxJava2标准的接口。

锁的续期：大家都知道，如果负责储存这个分布式锁的Redisson节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟（每到20s就会自动续借成30s，是1/3的关系），也可以通过修改Config.lockWatchdogTimeout来另行指定。

```java
// 加锁以后10秒钟自动解锁，看门狗不续命
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}
如果传递了锁的超时时间，就执行脚本，进行占锁;
如果没传递锁时间，使用看门狗的时间，占锁。如果返回占锁成功future，调用future.onComplete();
没异常的话调用scheduleExpirationRenewal(threadId);
重新设置过期时间，定时任务;
看门狗的原理是定时任务：重新给锁设置过期时间，新的过期时间就是看门狗的默认时间;
锁时间/3是定时任务周期;

```

Redisson同时还为分布式锁提供了异步执行的相关方法：

```java
RLock lock = redisson.getLock("anyLock");
lock.lockAsync();
lock.lockAsync(10, TimeUnit.SECONDS);
Future<Boolean> res = lock.tryLockAsync(100, 10, TimeUnit.SECONDS);

```

RLock对象完全符合Java的Lock规范。也就是说只有拥有锁的进程才能解锁，其他进程解锁则会抛出IllegalMonitorStateException错误。但是如果遇到需要其他进程也能解锁的情况，请使用分布式信号量Semaphore 对象.

```java
public Map<String, List<Catalog2Vo>> getCatalogJsonDbWithRedisson() {
    Map<String, List<Catalog2Vo>> categoryMap=null;
    RLock lock = redissonClient.getLock("CatalogJson-Lock");
    lock.lock();
    try {
        Thread.sleep(30000);
        categoryMap = getCategoryMap();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }finally {
        lock.unlock();
        return categoryMap;
    }
}

```

最佳实战：自己指定锁时间，时间长点即可


##### 3、Reidsson - 读写锁

基于Redis的Redisson分布式可重入读写锁RReadWriteLock Java对象实现了java.util.concurrent.locks.ReadWriteLock接口。其中读锁和写锁都继承了RLock接口。

分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。

```java
RReadWriteLock rwlock = redisson.getReadWriteLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();

```



```java
// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

上锁时在redis的状态

HashWrite-Lock
key:mode  value:read
key:sasdsdffsdfsdf... value:1

上锁时在redis的状态

<img src="/Snipaste_2020-09-09_12-53-21.png" style="zoom: 50%;" />



二话不说，上代码！！！

```java
/**
     * 保证一定能读取到最新数据，修改期间，写锁是一个排他锁（互斥锁，独享锁）读锁是一个共享锁
     * 写锁没释放读锁就必须等待
     * 读 + 读 相当于无锁，并发读，只会在 reids中记录好，所有当前的读锁，他们都会同时加锁成功
     * 写 + 读 等待写锁释放
     * 写 + 写 阻塞方式
     * 读 + 写 有读锁，写也需要等待
     * 只要有写的存在，都必须等待
     * @return String
     */
    @RequestMapping("/write")
    @ResponseBody
    public String writeValue() {

        RReadWriteLock lock = redission.getReadWriteLock("rw_lock");
        String s = "";
        RLock rLock = lock.writeLock();
        try {
            // 1、改数据加写锁，读数据加读锁
            rLock.lock();
            System.out.println("写锁加锁成功..." + Thread.currentThread().getId());
            s = UUID.randomUUID().toString();
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            redisTemplate.opsForValue().set("writeValue",s);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rLock.unlock();
            System.out.println("写锁释放..." + Thread.currentThread().getId());
        }
        return s;
    }

    @RequestMapping("/read")
    @ResponseBody
    public String readValue() {
        RReadWriteLock lock = redission.getReadWriteLock("rw_lock");
        RLock rLock = lock.readLock();
        String s = "";
        rLock.lock();
        try {
            System.out.println("读锁加锁成功..." + Thread.currentThread().getId());
            s = (String) redisTemplate.opsForValue().get("writeValue");
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rLock.unlock();
            System.out.println("读锁释放..." + Thread.currentThread().getId());
        }
        return s;
    }

```

##### 4、Redisson - 闭锁测试

基于Redisson的Redisson分布式闭锁（CountDownLatch）Java对象RCountDownLatch采用了与java.util.concurrent.CountDownLatch相似的接口和用法。

以下代码只有offLatch()被调用5次后 setLatch()才能继续执行



```java
 	@GetMapping("/setLatch")
    @ResponseBody
    public String setLatch() {
        RCountDownLatch latch = redissonClient.getCountDownLatch("CountDownLatch");
        try {
            latch.trySetCount(5);
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "门栓被放开";
    }

    @GetMapping("/offLatch")
    @ResponseBody
    public String offLatch() {
        RCountDownLatch latch = redissonClient.getCountDownLatch("CountDownLatch");
        latch.countDown();
        return "门栓被放开1";
    }
```

闭锁在redis的存储状态

<img src="/Snipaste_2020-09-09_13-11-45.png" style="zoom:50%;" />



和 JUC 的 CountDownLatch 一致

await()等待闭锁完成

countDown() 把计数器减掉后 await就会放行

##### 5、Redisson - 信号量测试

信号量为存储在redis中的一个数字，当这个数字大于0时，即可以调用acquire()方法增加数量，也可以调用release()方法减少数量，但是当调用release()之后小于0的话方法就会阻塞，直到数字大于0

基于Redis的Redisson的分布式信号量（Semaphore）Java对象RSemaphore采用了与java.util.concurrent.Semaphore相似的接口和用法。同时还提供了异步（Async）、反射式（Reactive）和RxJava2标准的接口。

```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.acquire();
//或
semaphore.acquireAsync();
semaphore.acquire(23);
semaphore.tryAcquire();
//或
semaphore.tryAcquireAsync();
semaphore.tryAcquire(23, TimeUnit.SECONDS);
//或
semaphore.tryAcquireAsync(23, TimeUnit.SECONDS);
semaphore.release(10);
semaphore.release();
//或
semaphore.releaseAsync();
```



```java
@GetMapping("/park")
@ResponseBody
public String park() {
    RSemaphore park = redissonClient.getSemaphore("park");
    try {
        park.acquire(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "停进2";
}

@GetMapping("/go")
@ResponseBody
public String go() {
    RSemaphore park = redissonClient.getSemaphore("park");
    park.release(2);
    return "开走2";
}
```

类似 JUC 中的 Semaphore 



##### 6、Redission - 缓存一致性解决

### 6.3 缓存数据一致性

#### 缓存数据一致性 - 双写模式

当数据更新时，更新数据库时同时更新缓存

<img src="/Snipaste_2020-09-10_18-29-41.png" style="zoom: 50%;" />

**存在问题**

由于卡顿等原因，导致写缓存2在最前，写缓存1在后面就出现了不一致

![](C:/Users/wangliangliang/Downloads/gulimall-learning-master/docs/images/Snipaste_2020-09-10_18-31-38.png)

这是暂时性的脏数据问题，但是在数据稳定，缓存过期以后，又能得到最新的正确数据



#### 缓存数据一致性 - 失效模式

数据库更新时将缓存删除

<img src="/Snipaste_2020-09-10_18-33-47.png" style="zoom: 50%;" />

**存在问题**

当两个请求同时修改数据库，一个请求已经更新成功并删除缓存时又有读数据的请求进来，这时候发现缓存中无数据就去数据库中查询并**放入缓存，在放入缓存前第二个更新数据库的请求成功**，这时候留在缓存中的数据依然是第一次数据更新的数据

<img src="/Snipaste_2020-09-10_18-34-56.png" style="zoom: 40%;" />

**解决方法**

1、缓存的所有数据都有过期时间，数据过期下一次查询触发主动更新
2、读写数据的时候(并且写的不频繁)，加上分布式的读写锁。



#### 缓存数据一致性解决方案

无论是双写模式还是失效模式，都会到这缓存不一致的问题，即多个实例同时更新会出事，怎么办？

- 1、如果是用户维度数据（订单数据、用户数据），这并发几率很小，几乎不用考虑这个问题，缓存数据加上过期时间，每隔一段时间触发读的主动更新即可
- 2、如果是菜单，商品介绍等基础数据，也可以去使用 canal 订阅，binlog 的方式
- <img src="Snipaste_2020-09-10_20-06-15.png" style="zoom:80%;" />
- 3、缓存数据 + 过期时间也足够解决大部分业务对缓存的要求
- 4、通过加锁保证并发读写，写写的时候按照顺序排好队，读读无所谓，所以适合读写锁，（业务不关心脏数据，允许临时脏数据可忽略）

总结:

- 我们能放入缓存的数据本来就不应该是实时性、一致性要求超高的。所以缓存数据的时候加上过期时间，保证每天拿到当前的最新值即可
- 我们不应该过度设计，增加系统的复杂性
- 遇到实时性、一致性要求高的数据，就应该查数据库，不走缓存,即使慢点

### 6.5 Spring Cache

==这部分的理论内容可以[参考我之前学习springboot记的笔记](https://github.com/NiceSeason/SpringBoot_demo/tree/master/docs)，也是雷丰阳老师的课程，更加详细深入，包含源码分析。==

每次都那样写缓存太麻烦了，spring从3.1开始定义了Cache、CacheManager接口来统一不同的缓存技术。并支持使用JCache(JSR-107)注解简化我们的开发

#### 1、简介

- Spring 从3.1开始定义了 `org.springframework.cache.Cache` 和 `org.sprngframework.cache.CacheManager` 接口睐统一不同的缓存技术
- 并支持使用 `JCache`（JSR-107）注解简化我们的开发
- Cache 接口为缓存的组件规范定义，包含缓存的各种操作集合 `Cache` 接口下 Spring 提供了各种 XXXCache的实现，如 `RedisCache`、`EhCache`,`ConcrrentMapCache`等等，
- 每次调用需要缓存功能实现方法的时候，`Spring` 会检查检查指定参数的目标方法是否已经被调用过，如果有就直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓存结果后返回给用户，下次直接调用从缓存中获取
- 使用 `Sprng` 缓存抽象时我们需要关注的点有以下两点
  - 1、确定方法需要被缓存以及他们的的缓存策略
  - 2、从缓存中读取之前缓存存储的数据

官网地址：https://docs.spring.io/spring-framework/docs/5.2.10.RELEASE/spring-framework-reference/integration.html#cache-strategie

缓存注解配置

![image-20201228171806703](image-20201228171806703.png)

#### 2、基础概念

从3.1版本开始，Spring 框架就支持透明地向现有 Spring 应用程序添加缓存。与事务支持类似，缓存抽象允许在对代码影响最小的情况下一致地使用各种缓存解决方案。从 Spring 4.1 开始，缓存抽象在JSR-107注释和更多定制选项的支持下得到了显著扩展。

```java
 /**
 *  8、整合SpringCache简化缓存开发
 *      1、引入依赖
 *          spring-boot-starter-cache
 *      2、写配置
 *          1、自动配置了那些
 *              CacheAutoConfiguration会导入 RedisCacheConfiguration
 *              自动配置好了缓存管理器，RedisCacheManager
 *          2、配置使用redis作为缓存
 *          Spring.cache.type=redis
 *
 *       4、原理
 *       CacheAutoConfiguration ->RedisCacheConfiguration ->
 *       自动配置了 RedisCacheManager ->初始化所有的缓存 -> 每个缓存决定使用什么配置
 *       ->如果redisCacheConfiguration有就用已有的，没有就用默认的
 *       ->想改缓存的配置，只需要把容器中放一个 RedisCacheConfiguration 即可
 *       ->就会应用到当前 RedisCacheManager管理所有缓存分区中
 */
```

指定缓存类型并在主配置类上加上注解@EnableCaching

```yaml
spring:
  cache:
  	#指定缓存类型为redis
    type: redis
    redis:
指定redis中的过期时间为1h
      time-to-live: 3600000
```


默认使用jdk进行序列化（可读性差），默认ttl为-1永不过期，自定义序列化方式需要编写配置类


​        

```java
@Configuration
public class MyCacheConfig {
    @Bean
    public RedisCacheConfiguration redisCacheConfiguration( CacheProperties cacheProperties) {
        
        CacheProperties.Redis redisProperties = cacheProperties.getRedis();
        org.springframework.data.redis.cache.RedisCacheConfiguration config = org.springframework.data.redis.cache.RedisCacheConfiguration
            .defaultCacheConfig();
        //指定缓存序列化方式为json
        config = config.serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        //设置配置文件中的各项配置，如过期时间
        if (redisProperties.getTimeToLive() != null) {
            config = config.entryTtl(redisProperties.getTimeToLive());
        }

        if (redisProperties.getKeyPrefix() != null) {
            config = config.prefixKeysWith(redisProperties.getKeyPrefix());
        }
        if (!redisProperties.isCacheNullValues()) {
            config = config.disableCachingNullValues();
        }
        if (!redisProperties.isUseKeyPrefix()) {
            config = config.disableKeyPrefix();
        }
        return config;
    }
}

```

2) 缓存自动配置

```java
// 缓存自动配置源码
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(CacheManager.class)
@ConditionalOnBean(CacheAspectSupport.class)
@ConditionalOnMissingBean(value = CacheManager.class, name = "cacheResolver")
@EnableConfigurationProperties(CacheProperties.class)
@AutoConfigureAfter({ CouchbaseAutoConfiguration.class, HazelcastAutoConfiguration.class,
                     HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class })
@Import({ CacheConfigurationImportSelector.class, // 看导入什么CacheConfiguration
         CacheManagerEntityManagerFactoryDependsOnPostProcessor.class })
public class CacheAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CacheManagerCustomizers cacheManagerCustomizers(ObjectProvider<CacheManagerCustomizer<?>> customizers) {
        return new CacheManagerCustomizers(customizers.orderedStream().collect(Collectors.toList()));
    }

    @Bean
    public CacheManagerValidator cacheAutoConfigurationValidator(CacheProperties cacheProperties,
                                                                 ObjectProvider<CacheManager> cacheManager) {
        return new CacheManagerValidator(cacheProperties, cacheManager);
    }

    @ConditionalOnClass(LocalContainerEntityManagerFactoryBean.class)
    @ConditionalOnBean(AbstractEntityManagerFactoryBean.class)
    static class CacheManagerEntityManagerFactoryDependsOnPostProcessor
        extends EntityManagerFactoryDependsOnPostProcessor {

        CacheManagerEntityManagerFactoryDependsOnPostProcessor() {
            super("cacheManager");
        }

    }

```



```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisConnectionFactory.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
@ConditionalOnBean(RedisConnectionFactory.class)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class RedisCacheConfiguration {

    @Bean // 放入缓存管理器
    RedisCacheManager cacheManager(CacheProperties cacheProperties, 
                                   CacheManagerCustomizers cacheManagerCustomizers,
                                   ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
                                   ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
                                   RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
        RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
            determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
        List<String> cacheNames = cacheProperties.getCacheNames();
        if (!cacheNames.isEmpty()) {
            builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
        }
        redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
        return cacheManagerCustomizers.customize(builder.build());
    }

```



#### 3、注解

缓存使用@Cacheable@CacheEvict

第一个方法存放缓存，第二个方法清空缓存

```java
// 调用该方法时会将结果缓存，缓存名为category，key为方法名
// sync表示该方法的缓存被读取时会加锁 // value等同于cacheNames // key如果是字符串"''"
@Cacheable(value = {"category"},key = "#root.methodName",sync = true)
public Map<String, List<Catalog2Vo>> getCatalogJsonDbWithSpringCache() {
    return getCategoriesDb();
}

//调用该方法会删除缓存category下的所有cache，如果要删除某个具体，用key="''"
@Override
@CacheEvict(value = {"category"},allEntries = true)
public void updateCascade(CategoryEntity category) {
    this.updateById(category);
    if (!StringUtils.isEmpty(category.getName())) {
        categoryBrandRelationService.updateCategory(category);
    }
}

如果要清空多个缓存，用@Caching(evict={@CacheEvict(value="")})

```

对于缓存声明，Spring的缓存抽象提供了一组Java注解

```java
/**
@Cacheable: Triggers cache population:触发将数据保存到缓存的操作
@CacheEvict: Triggers cache eviction: 触发将数据从缓存删除的操作
@CachePut: Updates the cache without interfering with the method execution:不影响方法执行更新缓存
@Caching: Regroups multiple cache operations to be applied on a method:组合以上多个操作
@CacheConfig: Shares some common cache-related settings at class-level:在类级别共享缓存的相同配置
**/
```

**注解使用**

```java
/**
     * 1、每一个需要缓存的数据我们都需要指定放到那个名字的缓存【缓存分区的划分【按照业务类型划分】】
     * 2、@Cacheable({"category"})
     *      代表当前方法的结果需要缓存，如果缓存中有，方法不调用
     *      如果缓存中没有，调用方法，最后将方法的结果放入缓存
     * 3、默认行为:
     *      1、如果缓存中有，方法不用调用
     *      2、key默自动生成，缓存的名字:SimpleKey[](自动生成的key值)
     *      3、缓存中value的值，默认使用jdk序列化，将序列化后的数据存到redis
     *      3、默认的过期时间，-1
     *
     *    自定义操作
     *      1、指定缓存使用的key     key属性指定，接收一个SpEl
     *      2、指定缓存数据的存活时间  配置文件中修改ttl
     *      3、将数据保存为json格式
     * @return
     */
	//value 缓存的别名
     // key redis中key的名称，默认是方法名称
    @Cacheable(value = {"category"},key = "#root.method.name")
    @Override
    public List<CategoryEntity> getLevel1Categorys() {
        long l = System.currentTimeMillis();
        // parent_cid为0则是一级目录
        List<CategoryEntity> categoryEntities = baseMapper.selectList(new QueryWrapper<CategoryEntity>().eq("parent_cid", 0));
        System.out.println("耗费时间：" + (System.currentTimeMillis() - l));
        return categoryEntities;
    }
```

#### 4、表达式语法

配置

```java
package com.atguigu.gulimall.product.config;

import org.springframework.boot.autoconfigure.cache.CacheProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * @author gcq
 * @Create 2020-11-01
 */
@EnableConfigurationProperties(CacheProperties.class)
@EnableCaching
@Configuration
public class MyCacheConfig {

    /**
     * 配置文件中的东西没有用上
     * 1、原来的配置吻技安绑定的配置类是这样子的
     *      @ConfigurationProperties(prefix = "Spring.cache")
     * 2、要让他生效
     *      @EnableConfigurationProperties(CacheProperties.class)
     * @param cacheProperties
     * @return
     */
    @Bean
    RedisCacheConfiguration redisCacheConfiguration(CacheProperties cacheProperties) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig();
        // 设置key的序列化
        config = config.serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()));
        // 设置value序列化 ->JackSon
        config = config.serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        CacheProperties.Redis redisProperties = cacheProperties.getRedis();
        if (redisProperties.getTimeToLive() != null) {
            config = config.entryTtl(redisProperties.getTimeToLive());
        }
        if (redisProperties.getKeyPrefix() != null) {
            config = config.prefixKeysWith(redisProperties.getKeyPrefix());
        }
        if (!redisProperties.isCacheNullValues()) {
            config = config.disableCachingNullValues();
        }
        if (!redisProperties.isUseKeyPrefix()) {
            config = config.disableKeyPrefix();
        }
        return config;
    }

}
```

yaml 

```yaml
Spring:
  cache:
    type: redis
    redis:
      time-to-live: 3600000           # 过期时间
      key-prefix: CACHE_              # key前缀
      use-key-prefix: true            # 是否使用写入redis前缀
      cache-null-values: true         # 是否允许缓存空值
```

#### 5、SpringCache原理与不足

1）、读模式

- 缓存穿透：查询一个null数据。解决方案：缓存空数据，可通过spring.cache.redis.cache-null-values=true
- 缓存击穿：大量并发进来同时查询一个正好过期的数据。解决方案：加锁 ? 默认是无加锁的;
使用sync = true来解决击穿问题
- 缓存雪崩：大量的key同时过期。解决：加随机时间。

2)、写模式：（缓存与数据库一致）
- 读写加锁。
- 引入Canal，感知到MySQL的更新去更新Redis
- 读多写多，直接去数据库查询就行
3）、总结：

常规数据（读多写少，即时性，一致性要求不高的数据，完全可以使用Spring-Cache）：

写模式(只要缓存的数据有过期时间就足够了)

特殊数据：特殊设计




```java
/**
 * 1、每一个需要缓存的数据我们都需要指定放到那个名字的缓存【缓存分区的划分【按照业务类型划分】】
 * 2、@Cacheable({"category"})
 *      代表当前方法的结果需要缓存，如果缓存中有，方法不调用
 *      如果缓存中没有，调用方法，最后将方法的结果放入缓存
 * 3、默认行为:
 *      1、如果缓存中有，方法不用调用
 *      2、key默自动生成，缓存的名字:SimpleKey[](自动生成的key值)
 *      3、缓存中value的值，默认使用jdk序列化，将序列化后的数据存到redis
 *      3、默认的过期时间，-1
 *
 *    自定义操作
 *      1、指定缓存使用的key     key属性指定，接收一个SpEl
 *      2、指定缓存数据的存活时间  配置文件中修改ttl
 *      3、将数据保存为json格式
 *      原理：
 *          CacheManager(RedisManager) -> Cache(RedisCache) ->Cache负责缓存的读写
 * @return
 */
@Cacheable(value = {"category"},key = "#root.method.name",sync = true)
@Override
public List<CategoryEntity> getLevel1Categorys() {
    long l = System.currentTimeMillis();
    // parent_cid为0则是一级目录
    List<CategoryEntity> categoryEntities = baseMapper.selectList(new QueryWrapper<CategoryEntity>().eq("parent_cid", 0));
    System.out.println("耗费时间：" + (System.currentTimeMillis() - l));
    return categoryEntities;
}
```

#### 6、缓存更新

```java
/**
     * 级联更新所有的关联数据
     * @CacheEvict 失效模式
     * 1、同时进行多种缓存操作 @Caching
     * 2、指定删除某个分区下的所有数据 @CacheEvict(value = {"category"},allEntries = true)
     * 3、存储同一类型的数据，都可以指定成同一分区，分区名默认就是缓存的前缀
     *
     * @param category
     */
    @Caching(evict = {
            @CacheEvict(value = {"category"},key = "'getLevel1Categorys'"),
            @CacheEvict(value = {"category"},key = "'getCatelogJson'")
    })
//    @CacheEvict(value = {"category"},allEntries = true)
    @Transactional
    @Override
    public void updateCascate(CategoryEntity category) {
        // 更新自己表对象
        this.updateById(category);
        // 更新关联表对象
        categoryBrandRelationService.updateCategory(category.getCatId(), category.getName());
    }
```

总结业务流程：

如果忘了这个技术点看下做的笔记的例子，然后去官网看下文档，温故而知新

流程图

![image-20201228171552816](image-20201228171552816.png)



# 7、商城业务 & 商品检索

### 7.1 检索业务分析

#### 7.1.1 检索查询参数模型分析抽取

打个比例吧 你肯定上过京东、淘宝买过东西吧？ 那么你想要购买什么东西，你需要在搜索框中搜索你想要购买的物品，那么系统就会给你响应	

我在京东搜索 `Iphone`  他会显示出相对应的产品

![image-20201104152544243](image-20201104152544243.png)



那么我们开始对业务条件进行分析，并创建对应的VO类

![image-20201104153118677](image-20201104153118677.png)

好的创建出来了........

```java
/**
 * 封装页面所有可能传递过来的查询条件
 * @author gcq
 * @Create 2020-11-02
 */
@Data
public class SearchParam {

    /**
     * 页面传递过来的全文匹配关键字
     */
    private String keyword;
    /**
     * 三级分类id
     */
    private Long catalog3Id;
    /**
     * sort=saleCout_asc/desc
     * sort=skuPrice_asc/desc
     * sort=hotScore_asc/desc
     * 排序条件
     */
    private String sort;

    /**
     * hasStock(是否有货) skuPrice区间，brandId、catalog3Id、attrs
     */
    /**
     * 是否显示有货
     */
    private Integer hasStock = 0;
    /**
     * 价格区间查询
     */
    private String skuPrice;
    /**
     * 按照品牌进行查询，可以多选
     */
    private List<Long> brandId;
    /**
     * 按照属性进行筛选
     */
    private List<String> attrs;
    /**
     * 页码
     */
    private Integer pageNum = 1;

}
```

#### 7.1.2 检索返回结果模型分析抽取

 那么返回的数据我们是不是也要创建一个 VO 用来返回页面的数据？

借鉴京东的实例来做参考

![image-20201104153950990](image-20201104153950990.png)

那么抽取实体类

```java
/**
 * 查询结果返回
 * @author gcq
 * @Create 2020-11-02
 */
@Data
public class SearchResult {
    /**
     * 查询到所有商品的商品信息
     */
    private List<SkuEsModel> products;
    /**
     * 以下是分页信息
     * 当前页码
     */
    private Integer pageNum;
    /**
     * 总共记录数
     */
    private Long total;
    /**
     * 总页码
     */
    private Integer totalPages;
    /**
     * 当前查询到的结果，所有涉及的品牌
     */
    private List<BrandVo> brands;
    /**
     * 当前查询结果，所有涉及到的分类
     */
    private List<CatalogVo> catalogs;
    /**
     * 当前查询到的结果，所有涉及到的所有属性
     */
    private List<AttrVo> attrs;
    /**
     * 页码
     */
    private List<Integer> pageNavs;

    //==================以上是要返回给页面的所有信息
    @Data
    public static class BrandVo {
        /**
         * 品牌id
         */
        private Long brandId;
        /**
         * 品牌名字
         */
        private String brandName;
        /**
         * 品牌图片
         */
        private String brandImg;
    }
    @Data
    public static class CatalogVo {
        /**
         * 分类id
         */
        private Long catalogId;
        /**
         * 品牌名字
         */
        private String CatalogName;
    }

    @Data
    public static class AttrVo {
        /**
         * 属性id
         */
        private Long attrId;

        /**
         * 属性名字
         */
        private String attrName;
        /**
         * 属性值
         */
        private List<String> attrValue;
    }

}
```

### 7.2 检索语句构建

#### 7.2.1 查询部分 & 聚合部分

那么这个 DSL 编写我们就在 Kibana 中测试

```json

GET gulimall_product/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "skuTitle": "华为" // 按照关键字查询
          }
        }
      ],
      "filter": [
        {
          "term": {
            "catalogId": "225" // 根据分类id过滤
          }
        },
        {
          "terms": {
            "brandId": [ // 品牌id
              "1",
              "5",
              "9"
            ]
          }
        },
        {
          "nested": { // 根据属性id 以及属性值进行过滤
            "path": "attrs",
            "query": {
              "bool": {
                "must": [
                  {
                    "term": {
                      "attrs.attrId": {
                        "value": "8"
                      }
                    }
                  },
                  {
                    "terms": {
                      "attrs.attrValue": [
                        "2019"
                      ]
                    }
                  }
                ]
              }
            }
          }
        },
        {
          "term": { // 是否有库存
            "hasStock": {
              "value": "false"
            }
          }
        },
        {
                  "range": { // 价格区间
                    "skuPrice": {
                      "gte": 0,
                      "lte": 7000
                    }
                  }
                }
      ]
    }
  },
  "sort": [ //排序
    {
      "skuPrice": {
        "order": "desc"
      }
    }
  ],
  "from": 0,
  "size":4,
  "highlight": { // 对搜索田间进行高亮
    "fields": {"skuTitle": {}}, 
    "pre_tags": "<b style=color:red>",
    "post_tags": "</b>"
  },
  "aggs": {
    "brand_agg": { //品牌进行聚合
      "terms": {
        "field": "brandId",
        "size": 10
      },
      "aggs": {
        "brand_name_agg": { // 品牌名字
          "terms": {
            "field": "brandName",
            "size": 10
          }
        },
        "brand_img_agg": { //品牌图片
          "terms": {
            "field": "brandImg",
            "size": 10
          }
        }
      }
    },
    "catalog_agg": { // 分类
      "terms": {
        "field": "catalogId",
        "size": 10
      },
      "aggs": {
        "catalog_name_agg": { //分类名字
          "terms": {
            "field": "catalogName",
            "size": 10
          }
        }
      }
    },
    "attr_agg":{
      "nested": {
        "path": "attrs"
      },
      "aggs": { //属性聚合
        "attr_id_agg": {
          "terms": {
            "field": "attrs.attrId",
            "size": 10
          },
          "aggs": {
            "attr_name_agg": { //属性名字
              "terms": {
                "field": "attrs.attrName",
                "size": 10
              }
            },
            "attr_value_agg":{ //属性的值
              "terms": {
                "field": "attrs.attrValue",
                "size": 10
              }
            }
          }
        }
      }
    }
  }
}

```

用代码实现

```java
 /**
     * 准备检索请求
     * #模糊匹配、过滤（按照属性、分类、品牌、价格区间、库存）、排序、分页、高亮、聚合分析
     *
     * @return
     */
    private SearchRequest buildSearchRequest(SearchParam param) {

        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder(); //构建DSL语句
        /**
         * 模糊匹配 过滤（按照属性、分类、品牌、价格区间、库存）
         */
        // 1、构建bool - query
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        // 1.1 must - 模糊匹配
        if (!StringUtils.isEmpty(param.getKeyword())) {
            boolQuery.must(QueryBuilders.matchQuery("skuTitle", param.getKeyword()));
        }
        // 1.2 bool - filter 按照三级分类id来查询
        if (param.getCatalog3Id() != null) {
            boolQuery.filter(QueryBuilders.termQuery("catalogId", param.getCatalog3Id()));
        }
        // 1.2 bool - filter 按照品牌id来查询
        if (param.getBrandId() != null && param.getBrandId().size() > 0) {
            boolQuery.filter(QueryBuilders.termsQuery("brandId", param.getBrandId()));
        }
        // 1.2 bool - filter 按照所有指定的属性来进行查询 *******不理解这个attr=1_5寸:8寸这样的设计
        if (param.getAttrs() != null && param.getAttrs().size() > 0) {
            for (String attr : param.getAttrs()) {
                // attr=1_5寸:8寸&attrs=2_16G:8G
                BoolQueryBuilder nestedboolQuery = QueryBuilders.boolQuery();
                String[] s = attr.split("_");
                String attrId = s[0];// 检索的属性id
                String[] attrValues = s[1].split(":");
                nestedboolQuery.must(QueryBuilders.termQuery("attrs.attrId", attrId));
                nestedboolQuery.must(QueryBuilders.termsQuery("attrs.attrValue", attrValues));
                // 每一个必须都生成一个nested查询
                NestedQueryBuilder nestedQuery = QueryBuilders.nestedQuery("attrs", nestedboolQuery, ScoreMode.None);
                boolQuery.filter(nestedQuery);
            }
        }
        // 1.2 bool - filter 按照库存是否存在
        boolQuery.filter(QueryBuilders.termQuery("hasStock", param.getHasStock() == 1 ? true : false));
        // 1.2 bool - filter 按照价格区间
        /**
         * 1_500/_500/500_
         */
        if (!StringUtils.isEmpty(param.getSkuPrice())) {
            RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("skuPrice");
            String[] s = param.getSkuPrice().split("_");
            if (s.length == 2) {
                // 区间
                rangeQuery.gte(s[0]).lte(s[1]);
            } else if (s.length == 1) {
                if (param.getSkuPrice().startsWith("_")) {
                    rangeQuery.lte(s[0]);
                }
                if (param.getSkuPrice().endsWith("_")) {
                    rangeQuery.gte(s[0]);
                }
            }
            boolQuery.filter(rangeQuery);
        }
        //把以前所有条件都拿来进行封装
        sourceBuilder.query(boolQuery);

        /**
         * 排序、分页、高亮
         */
        //2.1、排序
        if (!StringUtils.isEmpty(param.getSort())) {
            String sort = param.getSort();
            //sort=hotScore_asc/desc
            String[] s = sort.split("_");
            SortOrder order = s[1].equalsIgnoreCase("asc") ? SortOrder.ASC : SortOrder.DESC;
            sourceBuilder.sort(s[0], order);
        }
        //2.2 分页 pageSize:5
        // pageNum:1 from 0 size:5 [0,1,2,3,4]
        // pageNum:2 from 5 size:5
        // from (pageNum - 1)*size
        sourceBuilder.from((param.getPageNum() - 1) * EsConstant.PRODUCT_PAGESIZE);
        sourceBuilder.size(EsConstant.PRODUCT_PAGESIZE);
        //2.3、高亮
        if (!StringUtils.isEmpty(param.getKeyword())) {
            HighlightBuilder builder = new HighlightBuilder();
            builder.field("skuTitle");
            builder.preTags("<b style='color:red'>");
            builder.postTags("</b>");
            sourceBuilder.highlighter(builder);
        }
        /**
         * 聚合分析
         */
        //1、品牌聚合
        TermsAggregationBuilder brand_agg = AggregationBuilders.terms("brand_agg");
        brand_agg.field("brandId").size(50);
        //品牌聚合的子聚合
        brand_agg.subAggregation(AggregationBuilders.terms("brand_name_agg").field("brandName").size(2));
        brand_agg.subAggregation(AggregationBuilders.terms("brand_img_agg").field("brandImg").size(2));
        // TODO 1、聚合brand
        sourceBuilder.aggregation(brand_agg);
        //2、分类聚合
        TermsAggregationBuilder catalog_agg = AggregationBuilders.terms("catalog_agg").field("catalogId").size(20);
        catalog_agg.subAggregation(AggregationBuilders.terms("catalog_name_agg").field("catalogName").size(1));
        // TODO 2、聚合catalog
        sourceBuilder.aggregation(catalog_agg);
        //3、属性聚合 attr_agg
        NestedAggregationBuilder attr_agg = AggregationBuilders.nested("attr_agg", "attrs");
        // 聚合出当前所有的attrId
        TermsAggregationBuilder attr_id_agg = AggregationBuilders.terms("attr_id_agg").field("attrs.attrId");
        //聚合分析出当前attr_id对应的名字
        attr_id_agg.subAggregation(AggregationBuilders.terms("attr_name_agg").field("attrs.attrName").size(1));
        // 聚合分析出当前attr_id对应的可能的属性值attractValue
        attr_id_agg.subAggregation(AggregationBuilders.terms("attr_value_agg").field("attrs.attrValue").size(50));
        attr_agg.subAggregation(attr_id_agg);
        // TODO 3、聚合attr
        sourceBuilder.aggregation(attr_agg);


        String s = sourceBuilder.toString();
        System.out.println("构建的DSL:" + s);
        SearchRequest searchRequest = new SearchRequest(new String[]{EsConstant.PRODUCT_INDEX}, sourceBuilder);
        return searchRequest;
    }
```

代码实现基本上是 根据 json 来写 调用对应的 API 

term 和 terms 不要调用错误

### 7.3 结果提取封装

Controller

```java
	/**
     * 自动将页面提交过来的所有请求查询参数自动封装成指定的对象
     * @param param
     * @return
     */
    @GetMapping("/list.html")
    public String listPage(SearchParam param, Model model){

        //1、根据传递来的页面参数，去es中检索商品
        SearchResult result = mallSearchService.search(param);
        model.addAttribute("result",result);
        return "list";
    }
```

Service

```java
@Override
public SearchResult search(SearchParam param) {
    // 1、动态构建出查询需要的DSL语句
    SearchResult result = null;

    //1、准备检索请求
    SearchRequest searchRequest = buildSearchRequest(param);
    try {
        // 2、执行检索请求
        SearchResponse response = client.search(searchRequest, GulimallElasticsearchConfig.COMMON_OPTIONS);

        // 3、分析响应数据封装成我们需要的格式
        result = buildSearchResult(response, param);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return result;
}
```

具体业务执行

```java
/**
 * 构建结果数据
 *
 * @param response
 * @return
 */
private SearchResult buildSearchResult(SearchResponse response, SearchParam param) {

    SearchResult result = new SearchResult();
    SearchHits hits = response.getHits();
    List<SkuEsModel> esModels = new ArrayList<>();
    if (hits.getHits() != null && hits.getHits().length > 0) {
        for (SearchHit hit : hits.getHits()) {
            String sourceAsString = hit.getSourceAsString();
            SkuEsModel skuEsModel = JSON.parseObject(sourceAsString, SkuEsModel.class);
            if (!StringUtils.isEmpty(param.getKeyword())) {
                HighlightField skuTitle = hit.getHighlightFields().get("skuTitle");
                String string = skuTitle.getFragments()[0].string();
                skuEsModel.setSkuTitle(string);
            }
            esModels.add(skuEsModel);
        }
    }
    //1、返回所有查询到的商品
    result.setProducts(esModels);
    //2、当前所有商品设计到的所有属性信息
    List<SearchResult.AttrVo> attrVos = new ArrayList<>();
    ParsedNested attr_agg = response.getAggregations().get("attr_agg");
    ParsedLongTerms attr_id_agg = attr_agg.getAggregations().get("attr_id_agg");
    for (Terms.Bucket bucket : attr_id_agg.getBuckets()) {
        SearchResult.AttrVo attrVo = new SearchResult.AttrVo();
        // 1、得到属性的id
        Long attrId = bucket.getKeyAsNumber().longValue();
        // 2、得到属性的名字
        String attrName = ((ParsedStringTerms) bucket.getAggregations().get("attr_name_agg")).getBuckets().get(0).getKeyAsString();
        // 3、得到属性的所有值
        List<String> attrValue = ((ParsedStringTerms) bucket.getAggregations().get("attr_value_agg")).getBuckets().stream().map(item -> {
            String keyAsString = item.getKeyAsString();
            return keyAsString;
        }).collect(Collectors.toList());
        attrVo.setAttrId(attrId);
        attrVo.setAttrName(attrName);
        attrVo.setAttrValue(attrValue);
        attrVos.add(attrVo);
    }
    result.setAttrs(attrVos);
    //3、当前所有商品的分类信息
    ParsedLongTerms Catalog_agg = response.getAggregations().get("catalog_agg");
    List<SearchResult.CatalogVo> catalogVos = new ArrayList<>();
    List<? extends Terms.Bucket> buckets = Catalog_agg.getBuckets();
    for (Terms.Bucket bucket : buckets) {
        SearchResult.CatalogVo catalogVo = new SearchResult.CatalogVo();
        // 得到分类id
        String keyAsString = bucket.getKeyAsString();
        catalogVo.setCatalogId(Long.parseLong(keyAsString));
        // 得到分类名
        ParsedStringTerms catalog_name_agg = bucket.getAggregations().get("catalog_name_agg");
        String catalog_name = catalog_name_agg.getBuckets().get(0).getKeyAsString();
        catalogVo.setCatalogName(catalog_name);
        catalogVos.add(catalogVo);
    }
    result.setCatalogs(catalogVos);

    //4、当前所有商品的品牌信息
    List<SearchResult.BrandVo> brandVos = new ArrayList<>();
    ParsedLongTerms brand_agg = response.getAggregations().get("brand_agg");
    for (Terms.Bucket bucket : brand_agg.getBuckets()) {
        SearchResult.BrandVo brandVo = new SearchResult.BrandVo();
        // 1、得到品牌的id
        long brandId = bucket.getKeyAsNumber().longValue();
        // 2、得到品牌的图片
        String brandImg = ((ParsedStringTerms) bucket.getAggregations().get("brand_img_agg")).getBuckets().get(0).getKeyAsString();
        // 3、得到品牌的姓名
        String brandname = ((ParsedStringTerms) bucket.getAggregations().get("brand_name_agg")).getBuckets().get(0).getKeyAsString();
        brandVo.setBrandName(brandname);
        brandVo.setBrandId(brandId);
        brandVo.setBrandImg(brandImg);
        brandVos.add(brandVo);
    }
    result.setBrands(brandVos);
    //5、分页信息 - 总记录数
    long total = hits.getTotalHits().value;
    result.setTotal(total);
    //6、分页信息 - 页码
    result.setPageNum(param.getPageNum());
    //7、分页信息 - 总页码
    int totalPages = (int) total % EsConstant.PRODUCT_PAGESIZE == 0 ? (int) total / EsConstant.PRODUCT_PAGESIZE : ((int) total / EsConstant.PRODUCT_PAGESIZE + 1);
    result.setTotalPages(totalPages);

    List<Integer> pageNavs = new ArrayList<>();
    for(int i = 1; i <= totalPages; i++) {
        pageNavs.add(i);
    }
    result.setPageNavs(pageNavs);
    return result;
}
```



### 7.4 页面数据渲染

#### 7.4.1基本数据渲染

将商品的基本属性渲染出来

```html
<div class="rig_tab">
    <!-- 遍历各个商品-->
    <div th:each="product : ${result.getProduct()}">
        <div class="ico">
            <i class="iconfont icon-weiguanzhu"></i>
            <a href="/static/search/#">关注</a>
        </div>
        <p class="da">
            <a th:href="|http://item.gulimall.com/${product.skuId}.html|" >
                <!--图片 -->
                <img   class="dim" th:src="${product.skuImg}">
            </a>
        </p>
        <ul class="tab_im">
            <li><a href="/static/search/#" title="黑色">
                <img th:src="${product.skuImg}"></a></li>
        </ul>
        <p class="tab_R">
              <!-- 价格 -->
            <span th:text="'￥' + ${product.skuPrice}">¥5199.00</span>
        </p>
        <p class="tab_JE">
            <!-- 标题 -->
            <!-- 使用utext标签,使检索时高亮不会被转义-->
            <a href="/static/search/#" th:utext="${product.skuTitle}">
                Apple iPhone 7 Plus (A1661) 32G 黑色 移动联通电信4G手机
            </a>
        </p>
        <p class="tab_PI">已有<span>11万+</span>热门评价
            <a href="/static/search/#">二手有售</a>
        </p>
        <p class="tab_CP"><a href="/static/search/#" title="谷粒商城Apple产品专营店">谷粒商城Apple产品...</a>
            <a href='#' title="联系供应商进行咨询">
                <img src="/static/search/img/xcxc.png">
            </a>
        </p>
        <div class="tab_FO">
            <div class="FO_one">
                <p>自营
                    <span>谷粒商城自营,品质保证</span>
                </p>
                <p>满赠
                    <span>该商品参加满赠活动</span>
                </p>
            </div>
        </div>
    </div>
</div>
```



遍历后显示结果

![image-20201105124552833](image-20201105124552833.png)



#### 7.4.3 筛选条件渲染

将用于筛选的品牌、分类、商品属性信息进行遍历显示，并且点击某个属性值时可以通过拼接url进行跳转

```html
<div class="JD_nav_logo">
    <!--品牌筛选-->
    <div class="JD_nav_wrap">
        <div class="sl_key">
            <span>品牌：</span>
        </div>
        <div class="sl_value">
            <div class="sl_value_logo">
                <ul>
                    <li th:each="brand: ${result.getBrands()}">
                        <!--替换url 传入搜索字符串和品牌id,进行查询-->
                        <a href="#"  th:href="${'javascript:searchProducts(&quot;brandId&quot;,'+brand.brandId+')'}">
                            <img src="/static/search/img/598033b4nd6055897.jpg" alt="" th:src="${brand.brandImg}">
                            <div th:text="${brand.brandName}">
                                华为(HUAWEI)
                            </div>
                        </a>
                    </li>
                </ul>
            </div>
        </div>
        <div class="sl_ext">
            <a href="#">
                更多
                <i style='background: url("image/search.ele.png")no-repeat 3px 7px'></i>
                <b style='background: url("image/search.ele.png")no-repeat 3px -44px'></b>
            </a>
            <a href="#">
                多选
                <i>+</i>
                <span>+</span>
            </a>
        </div>
    </div>
    <!--分类信息筛选-->
    <div class="JD_pre" th:each="catalog: ${result.getCatalogs()}">
        <div class="sl_key">
            <span>分类：</span>
        </div>
        <div class="sl_value">
            <ul>
                <li><a href="#" th:text="${catalog.getCatalogName()}" th:href="${'javascript:searchProducts(&quot;catalogId&quot;,'+catalog.catalogId+')'}">0-安卓（Android）</a></li>
            </ul>
        </div>
    </div>
    <!--价格-->
    <div class="JD_pre">
        <div class="sl_key">
            <span>价格：</span>
        </div>
        <div class="sl_value">
            <ul>
                <li><a href="#">0-499</a></li>
                <li><a href="#">500-999</a></li>
                <li><a href="#">1000-1699</a></li>
                <li><a href="#">1700-2799</a></li>
                <li><a href="#">2800-4499</a></li>
                <li><a href="#">4500-11999</a></li>
                <li><a href="#">12000以上</a></li>
                <li class="sl_value_li">
                    <input type="text">
                    <p>-</p>
                    <input type="text">
                    <a href="#">确定</a>
                </li>
            </ul>
        </div>
    </div>
    <!--商品属性筛选-->
    <!-- 注:将前面请求数据时已选择的规格参数排除出去-->
        <div class="JD_pre" th:each="attr : ${result.attrs}" th:if="${!#lists.contains(result.attrIds,attr.attrId)}">
          <div class="sl_key">
            <span th:text="${attr.attrName}">属性名字</span>
          </div>
          <div class="sl_value">
            <ul>
              <li th:each="val : ${attr.attrValue}"><a th:href="${'javascript:searchProducts(&quot;attrs&quot;, &quot;' + attr.attrId + '_' + val + '&quot;)'}" th:text="${val}">显示所有属性值</a></li>
            </ul>
          </div>
        </div>
</div>
```







```javascript
 
function searchProducts(name, value) {
    // 跳转对应地址
        location.href = replaceAndParamVal(location.href,name,value)
    }
   /**
     * 正则表达式替换
     * **/
    function replaceAndParamVal(url, paramName, replaceVal,forceAdd) {
        var oUrl = url.toString();
        // 1、如果没有就添加，有就替换
        if (oUrl.indexOf(paramName) != -1) {
            if(forceAdd) {
                var nUrl = "";
                if (oUrl.indexOf("?") != -1) {
                    nUrl = oUrl + "&" + paramName + "=" + replaceVal;
                } else {
                    nUrl = oUrl + "?" + paramName + "=" + replaceVal;
                }
                return nUrl;
            } else {
                var re = eval('/(' + paramName + '=)([^&]*)/gi');
                var nUrl = oUrl.replace(re, paramName + '=' + replaceVal)
                return nUrl;
            }
        } else {
            var nUrl = "";
            if (oUrl.indexOf("?") != -1) {
                nUrl = oUrl + "&" + paramName + "=" + replaceVal;
            } else {
                nUrl = oUrl + "?" + paramName + "=" + replaceVal;
            }
            return nUrl;
        }

    }

```





#### 7.4.4 分页数据筛选

将页码绑定至属性pn，当点击某页码时，通过获取pn值进行url拼接跳转页面

```html
<div class="filter_page">
    <div class="page_wrap">
        <span class="page_span1">
               <!-- 不是第一页时显示上一页 -->
            <a class="page_a" href="#" th:if="${result.pageNum>1}" th:attr="pn=${result.getPageNum()-1}">
                < 上一页
            </a>
             <!-- 将各个页码遍历显示，并将当前页码绑定至属性pn -->
            <a href="#" class="page_a"
               th:each="page: ${result.pageNavs}"
               th:text="${page}"
               th:style="${page==result.pageNum?'border: 0;color:#ee2222;background: #fff':''}"
               th:attr="pn=${page}"
            >1</a>
              <!-- 不是最后一页时显示下一页 -->
            <a href="#" class="page_a" th:if="${result.pageNum<result.totalPages}" th:attr="pn=${result.getPageNum()+1}">
                下一页 >
            </a>
        </span>
        <span class="page_span2">
            <em>共<b th:text="${result.totalPages}">169</b>页&nbsp;&nbsp;到第</em>
            <input type="number" value="1" class="page_input">
            <em>页</em>
            <a href="#">确定</a>
        </span>
    </div>
</div>
```

```javascript
$(".page_a").click(function () {
    var pn=$(this).attr("pn");
    location.href=replaceParamVal(location.href,"pageNum",pn,false);
    console.log(replaceParamVal(location.href,"pageNum",pn,false))
})
```



#### 7.4.5 页面排序功能

#### 

![](images/Snipaste_2020-09-18_12-47-55.png)

页面排序功能需要保证，点击某个按钮时，样式会变红，并且其他的样式保持最初的样子；

点击某个排序时首先按升序显示，再次点击再变为降序，并且还会显示上升或下降箭头

页面排序跳转的思路是通过点击某个按钮时会向其`class`属性添加/去除`desc`，并根据属性值进行url拼接

```html
<div class="filter_top">
    <div class="filter_top_left" th:with="p = ${param.sort}, priceRange = ${param.skuPrice}">
        <!-- 通过判断当前class是否有desc来进行样式的渲染和箭头的显示-->
        <a sort="hotScore"
           th:class="${(!#strings.isEmpty(p) && #strings.startsWith(p,'hotScore') && #strings.endsWith(p,'desc')) ? 'sort_a desc' : 'sort_a'}"
           th:attr="style=${(#strings.isEmpty(p) || #strings.startsWith(p,'hotScore')) ?
               'color: #fff; border-color: #e4393c; background: #e4393c;':'color: #333; border-color: #ccc; background: #fff;' }">
            综合排序[[${(!#strings.isEmpty(p) && #strings.startsWith(p,'hotScore') &&
            #strings.endsWith(p,'desc')) ?'↓':'↑' }]]</a>
        <a sort="saleCount"
           th:class="${(!#strings.isEmpty(p) && #strings.startsWith(p,'saleCount') && #strings.endsWith(p,'desc')) ? 'sort_a desc' : 'sort_a'}"
           th:attr="style=${(!#strings.isEmpty(p) && #strings.startsWith(p,'saleCount')) ?
               'color: #fff; border-color: #e4393c; background: #e4393c;':'color: #333; border-color: #ccc; background: #fff;' }">
            销量[[${(!#strings.isEmpty(p) && #strings.startsWith(p,'saleCount') &&
            #strings.endsWith(p,'desc'))?'↓':'↑'  }]]</a>
        <a sort="skuPrice"
           th:class="${(!#strings.isEmpty(p) && #strings.startsWith(p,'skuPrice') && #strings.endsWith(p,'desc')) ? 'sort_a desc' : 'sort_a'}"
           th:attr="style=${(!#strings.isEmpty(p) && #strings.startsWith(p,'skuPrice')) ?
               'color: #fff; border-color: #e4393c; background: #e4393c;':'color: #333; border-color: #ccc; background: #fff;' }">
            价格[[${(!#strings.isEmpty(p) && #strings.startsWith(p,'skuPrice') &&
            #strings.endsWith(p,'desc'))?'↓':'↑'  }]]</a>
        <a sort="hotScore" class="sort_a">评论分</a>
        <a sort="hotScore" class="sort_a">上架时间</a>
        <!--价格区间搜索-->
        <input id="skuPriceFrom" type="number"
               th:value="${#strings.isEmpty(priceRange)?'':#strings.substringBefore(priceRange,'_')}"
               style="width: 100px; margin-left: 30px">
        -
        <input id="skuPriceTo" type="number"
               th:value="${#strings.isEmpty(priceRange)?'':#strings.substringAfter(priceRange,'_')}"
               style="width: 100px">
        <button id="skuPriceSearchBtn">确定</button>
    </div>
    <div class="filter_top_right">
        <span class="fp-text">
           <b>1</b><em>/</em><i>169</i>
       </span>
        <a href="#" class="prev"><</a>
        <a href="#" class="next"> > </a>
    </div>
</div>
```

```javascript
$(".sort_a").click(function () {
    	//添加、剔除desc
        $(this).toggleClass("desc");
    	//获取sort属性值并进行url跳转
        let sort = $(this).attr("sort");
        sort = $(this).hasClass("desc") ? sort + "_desc" : sort + "_asc";
        location.href = replaceParamVal(location.href, "sort", sort,false);
        return false;
    });
```

#### 7.4.6 页面价格筛选

![image-20201105131105507](images/image-20201105131105507.png)

JS

```javascript
$("#skuPriceSearchBtn").click(function() {
    // 1、拼上价格区间的查询条件
    var from = $("#skuPriceFrom").val();
    var to = $("#skuPriceTo").val();
    // 2、拼接语句
    var query = from + "_" + to;
    // 3、替换 skuPrice
    location.href = replaceAndParamVal(location.href,"skuPrice",query);
})
```

#### 7.4.7 面包屑导航（遇到问题！）

前端页面：

在封装结果时，将查询的属性值进行封装

```java
   // 6. 构建面包屑导航
        List<String> attrs = searchParam.getAttrs();
        if (attrs != null && attrs.size() > 0) {
            List<SearchResult.NavVo> navVos = attrs.stream().map(attr -> {
                String[] split = attr.split("_");
                SearchResult.NavVo navVo = new SearchResult.NavVo();
                //6.1 设置属性值
                navVo.setNavValue(split[1]);
                //6.2 查询并设置属性名
                try {
                    R r = productFeignService.info(Long.parseLong(split[0]));
                    if (r.getCode() == 0) {
                        AttrResponseVo attrResponseVo = JSON.parseObject(JSON.toJSONString(r.get("attr")), new TypeReference<AttrResponseVo>() {
                        });
                        navVo.setNavName(attrResponseVo.getAttrName());
                    }
                } catch (Exception e) {
                    log.error("远程调用商品服务查询属性失败", e);
                }
                //6.3 设置面包屑跳转链接(当点击该链接时剔除点击属性)
                String queryString = searchParam.get_queryString();
                String replace = queryString.replace("&attrs=" + attr, "").replace("attrs=" + attr+"&", "").replace("attrs=" + attr, "");
                navVo.setLink("http://search.gulimall.com/search.html" + (replace.isEmpty()?"":"?"+replace));
                return navVo;
            }).collect(Collectors.toList());
            result.setNavs(navVos);
        }
```

页面渲染

```html
<div class="JD_ipone_one c">
    <!-- 遍历面包屑功能 -->
    <a th:href="${nav.link}" th:each="nav:${result.navs}">
        <span th:text="${nav.navName}"></span>：
        <span th:text="${nav.navValue}"></span> x</a>
</div>
```

<img src="images/Snipaste_2020-09-18_12-59-52.png" style="zoom: 50%;" />

在返回Vo类中 新增了

![image-20201105131841728](images/image-20201105131841728.png)

Controller中 的解析方法中

```java
    // 8、构建面包屑导航功能
        if (param.getAttrs()!=null && param.getAttrs().size() >0){
            List<SearchResult.NavVo> collect = param.getAttrs().stream().map(attr -> {
                // 1、分析每个 attrs传递过来的值
                SearchResult.NavVo navo = new SearchResult.NavVo();
                // attrs=2_5寸：6寸
                String[] s = attr.split("_");
                navo.setNavValue(s[1]);
                R r = productFeignService.getAttrInfo(Long.parseLong(s[0]));
                result.getAttrIds().add(Long.parseLong(s[0]));
                if (r.getCode() == 0) {
                    AttrResponseVo data = r.getData("attr", new TypeReference<AttrResponseVo>() {
                    });
                    navo.setNavName(data.getAttrName());
                } else {
                    navo.setNavName(s[0]);
                }
                // 取消这个面包屑导航以后，我们要跳转到那个地方，将请求地址的url里面的当前置空
                //拿到所有的查询条件后，去掉当前
                // attrs=15_海思(Hisilicon)

                String replace = replaceQueryString(param, attr,"attrs");
                navo.setLink("http://search.gulimall.com/list.html?" + replace);

                return navo;
            }).collect(Collectors.toList());
            result.setNavs(collect);
        }
        // 品牌、分类
        if(param.getBrandId() != null && param.getBrandId().size() > 0) {
            List<SearchResult.NavVo> navs = result.getNavs();
            SearchResult.NavVo navVo = new SearchResult.NavVo();
            navVo.setNavName("品牌");
            //TODO 远程查询所有品牌
            R info = productFeignService.info(param.getBrandId());
            if (info.getCode() == 0) {
                List<BrandVo> brand = info.getData("brand", new TypeReference<List<BrandVo>>() {
                });
                StringBuffer buffer = new StringBuffer();
                String replace = "";
                for (BrandVo brandVo : brand) {
                    buffer.append(brandVo.getBrandName() + ";");
                    replace = replaceQueryString(param,brandVo.getBrandId() + "","brandId");
                }
                navVo.setNavValue(buffer.toString());
                navVo.setLink("http://search.hanhunmall.com/list.html?" + replace);
            }
            navs.add(navVo);
        }

        //TODO 分类：不需要导航
        return result;
```



# 8、异步 & 线程池

### 8.1 线程回顾

#### 8.1.1 初始化线程的 4 种方式

1、继承 Thread

2、实现 Runnable

3、实现 Callable 接口 + FutureTask（可以拿到返回结果，可以处理异常）

4、线程池

方式一和方式二 主进程无法获取线程的运算结果，不适合当前场景

方式三：主进程可以获取当前线程的运算结果，但是不利于控制服务器种的线程资源，可以导致服务器资源耗尽

方式四：通过如下两种方式初始化线程池

```java
Executors.newFixedThreadPool(3);
//或者
new ThreadPollExecutor(corePoolSize,maximumPoolSize,keepAliveTime,TimeUnit,unit,workQueue,threadFactory,handler);
```

<span style="color:blue;font-">通过线程池性能稳定，也可以获取执行结果，并捕获异常，但是</span>，**在业务复杂情况下，一个异步调用可能会依赖另一个异步调用的执行结果**

#### 8.1.2 线程池的 7 大参数	

![image-20201105154808826](image-20201105154808826.png)

运行流程：

1、线程池创建，准备好 `core` 数量 的核心线程，准备接受任务

2、新的任务进来，用 `core` 准备好的空闲线程执行

- `core` 满了，就将再进来的任务放入阻塞队列中，空闲的 core 就会自己去阻塞队列获取任务执行
- 阻塞队列也满了，就直接开新线程去执行，最大只能开到 `max` 指定的数量
- `max` 都执行好了，`Max-core` 数量空闲的线程会在 `keepAliveTime` 指定的时间后自动销毁，终保持到 `core` 大小
- 如果线程数开到了 `max` 数量，还有新的任务进来，就会使用 reject 指定的拒绝策略进行处理

3、所有的线程创建都是由指定的 `factory` 创建的



面试;

一个线程池 core 7、max 20 ，queue 50 100 并发进来怎么分配的 ?

先有 7 个能直接得到运行，接下来 50 个进入队列排队，再多开 13 个继续执行，线程70个被安排上了，剩下30个默认拒绝策略



#### 8.1.3 常见的 4 种线程池

- `newCacheThreadPool`
  - 创建一个可缓存的线程池，如果线程池长度超过需要，可灵活回收空闲线程，若无可回收，则新建线程
- `newFixedThreadPool`
  - 创建一个指定长度的线程池，可控制线程最大并发数，超出的线程会再队列中等待
- `newScheduleThreadPool`
  - 创建一个定长线程池，支持定时及周期性任务执行
- `newSingleThreadExecutor`
  - 创建一个单线程化的线程池，她只会用唯一的工作线程来执行任务，保证所有任务

#### 8.1.4 开发中为什么使用线程池

- 降低资源的消耗
  - 通过重复利用已创建好的线程降低线程的创建和销毁带来的损耗
- 提高响应速度
  - 因为线程池中的线程没有超过线程池的最大上限时，有的线程处于等待分配任务的状态，当任务来时无需创建新的线程就能执行
- 提高线程的可管理性
  - 线程池会根据当前系统的特点对池内的线程进行优化处理，减少创建和销毁线程带来的系统开销，无限的创建和销毁线程不仅消耗系统资源，还降低系统的稳定性，使用线程池进行统一分配

### 8.2 CompletableFuture 异步编排

业务场景：

查询商品详情页逻辑比较复杂，有些数据还需要远程调用，必然需要花费更多的时间

![image-20201105163535757](image-20201105163535757.png )

假如商品详情页的每个查询，需要如下标注时间才能完成

那么，用户需要5.5s后才能看到商品相详情页的内容，很显然是不能接受的

如果有多个线程同时完成这 6 步操作，也许只需要 1.5s 即可完成响应

#### 8.2.1 创建异步对象

CompletableFuture 提供了四个静态方法来创建一个异步操作

![image-20201105185420349](image-20201105185420349.png)

1、**runXxx 都是没有返回结果的，supplyXxxx都是可以获取返回结果的**

2、可以传入自定义的线程池，否则就是用默认的线程池

3、根据方法的返回类型来判断是否该方法是否有返回类型

代码实现：

```java
  public static void main(String[] args) throws ExecutionException, InterruptedException {
        System.out.println("main....start.....");
        CompletableFuture<Void> completableFuture = CompletableFuture.runAsync(() -> {
            System.out.println("当前线程：" + Thread.currentThread().getId());
            int i = 10 / 2;
            System.out.println("运行结果：" + i);
        }, executor);

        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("当前线程：" + Thread.currentThread().getId());
            int i = 10 / 2;
            System.out.println("运行结果：" + i);
            return i;
        }, executor);
        Integer integer = future.get();

        System.out.println("main....stop....." + integer);
    }
```



#### 8.2.2 计算完成时回调方法

![image-20201105185821263](image-20201105185821263.png)

whenComplete 可以处理正常和异常的计算结果，exceptionally 处理异常情况

whenComplete 和 whenCompleteAsync 的区别

​		whenComplete ：是执行当前任务的线程继续执行 whencomplete 的任务

​		whenCompleteAsync： 是执行把 whenCompleteAsync 这个任务继续提交给线程池来进行执行

**方法不以 Async 结尾，意味着 Action 使用相同的线程执行，而 Async 可能会使用其他线程执行（如果是使用相同的线程池，也可能会被同一个线程选中执行）**

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("当前线程：" + Thread.currentThread().getId());
    int i = 10 / 0;
    System.out.println("运行结果：" + i);
    return i;
}, executor).whenComplete((res,exception) ->{
    // 虽然能得到异常信息，但是没法修改返回的数据
    System.out.println("异步任务成功完成了...结果是：" +res + "异常是：" + exception);
}).exceptionally(throwable -> {
    // 可以感知到异常，同时返回默认值
    return 10;
});
```

#### 8.2.3 handle 方法

![image-20201105194503175](image-20201105194503175.png)

和 complete 一样，可以对结果做最后的处理（可处理异常），可改变返回值

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("当前线程：" + Thread.currentThread().getId());
    int i = 10 / 2;
    System.out.println("运行结果：" + i);
    return i;
}, executor).handle((res,thr) ->{
    if (res != null ) {
        return res * 2;
    }
    if (thr != null) {
        return 0;
    }
    return 0;
});
```

#### 8.2.4 线程串行方法

![image-20201105195632819](image-20201105195632819.png)

thenApply 方法：**当一个线程依赖另一个线程时，获取上一个任务返回的结果，并返回当前任务的返回值**

thenAccept方法：**消费处理结果，接受任务处理结果，并消费处理，无返回结果**

thenRun 方法：**只要上面任务执行完成，就开始执行 thenRun ,只是处理完任务后，执行 thenRun的后续操作**

带有 Async 默认是异步执行的，同之前，

以上都要前置任务完成

```java
   /**
         * 线程串行化，
         * 1、thenRun:不能获取到上一步的执行结果，无返回值
         * .thenRunAsync(() ->{
         *             System.out.println("任务2启动了....");
         *         },executor);
         * 2、能接受上一步结果，但是无返回值 thenAcceptAsync
         * 3、thenApplyAsync 能收受上一步结果，有返回值
         *
         */
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("当前线程：" + Thread.currentThread().getId());
            int i = 10 / 2;
            System.out.println("运行结果：" + i);
            return i;
        }, executor).thenApplyAsync(res -> {
            System.out.println("任务2启动了..." + res);
            return "Hello " + res;
        }, executor);
        String s = future.get();

        System.out.println("main....stop....." + s);
```

#### 8.2.5 两任务组合 - 都要完成

![image-20210102044028142](image-20210102044028142.png)

![image-20210102044044914](image-20210102044044914.png)



两个任务必须都完成，触发该任务

thenCombine: 组合两个 future，获取两个 future的返回结果，并返回当前任务的返回值

thenAccpetBoth: 组合两个 future，获取两个 future 任务的返回结果，然后处理任务，没有返回值

runAfterBoth:组合 两个 future，不需要获取 future 的结果，只需要两个 future处理完成任务后，处理该任务，

```java
   /**
         * 两个都完成
         */
        CompletableFuture<Integer> future01 = CompletableFuture.supplyAsync(() -> {
            System.out.println("任务1当前线程：" + Thread.currentThread().getId());
            int i = 10 / 4;
            System.out.println("任务1结束：" + i);
            return i;
        }, executor);

        CompletableFuture<String> future02 = CompletableFuture.supplyAsync(() -> {
            System.out.println("任务2当前线程：" + Thread.currentThread().getId());
            System.out.println("任务2结束：");
            return "Hello";
        }, executor);

        // f1 和 f2 执行完成后在执行这个
//        future01.runAfterBothAsync(future02,() -> {
//            System.out.println("任务3开始");
//        },executor);

        // 返回f1 和 f2 的运行结果
//        future01.thenAcceptBothAsync(future02,(f1,f2) -> {
//            System.out.println("任务3开始....之前的结果:" + f1 + "==>" + f2);
//        },executor);

        // f1 和 f2 单独定义返回结果
        CompletableFuture<String> future = future01.thenCombineAsync(future02, (f1, f2) -> {
            return f1 + ":" + f2 + "-> Haha";
        }, executor);

        System.out.println("main....end....." + future.get());
```



#### 8.2.6 两任务组合 - 一个完成

![image-20201106101904880](image-20201106101904880.png)

![image-20201106101918013](image-20201106101918013.png)

当两个任务中，任意一个future 任务完成时，执行任务

**applyToEither**;两个任务有一个执行完成，获取它的返回值，处理任务并有新的返回值

**acceptEither**: 两个任务有一个执行完成，获取它的返回值，处理任务，没有新的返回值

**runAfterEither**:两个任务有一个执行完成，不需要获取 future 的结果，处理任务，也没有返回值



```java
/**
         * 两个任务，只要有一个完成，我们就执行任务
         * runAfterEnitherAsync：不感知结果，自己没有返回值
         * acceptEitherAsync：感知结果，自己没有返回值
         *  applyToEitherAsync：感知结果，自己有返回值
         */
//        future01.runAfterEitherAsync(future02,() ->{
//            System.out.println("任务3开始...之前的结果:");
//        },executor);

//        future01.acceptEitherAsync(future02,(res) -> {
//            System.out.println("任务3开始...之前的结果:" + res);
//        },executor);

        CompletableFuture<String> future = future01.applyToEitherAsync(future02, res -> {
            System.out.println("任务3开始...之前的结果：" + res);
            return res.toString() + "->哈哈";
        }, executor);
```



#### 8.2.7 多任务组合

![image-20201106104031315](image-20201106104031315.png)

allOf：**等待所有任务完成**

anyOf:**只要有一个任务完成**

```java
        CompletableFuture<String> futureImg = CompletableFuture.supplyAsync(() -> {
            System.out.println("查询商品的图片信息");
            return "hello.jpg";
        });

        CompletableFuture<String> futureAttr = CompletableFuture.supplyAsync(() -> {
            System.out.println("查询商品的属性");
            return "黑色+256G";
        });

        CompletableFuture<String> futureDesc = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println("查询商品介绍");
            return "华为";
        });

        // 等待全部执行完
//        CompletableFuture<Void> allOf = CompletableFuture.allOf(futureImg, futureAttr, futureDesc);
//        allOf.get();

        // 只需要有一个执行完
        CompletableFuture<Object> anyOf = CompletableFuture.anyOf(futureImg, futureAttr, futureDesc);
        anyOf.get();
        System.out.println("main....end....." + anyOf.get());
```

都是操作 CompletableFuture 类 更多方法还请参考该类



# 9、商品详情

### 9.1 详情数据

![image-20201229121729595](image-20201229121729595.png)

![image-20201109080935340](image-20201109080935340.png)

**需求分析：**通过 `skuId` 查询出商品的相关信息，图片、标题、价格，属性对应版本等等

#### 9.1.1 返回数据模型抽取

```java
/**
 * @author gcq
 * @Create 2020-11-06
 */
@Data
public class SkuItemVo {
    // 1、sku基本获取 pms_sku_info
    SkuInfoEntity skuInfo;
    // 是否有库存
    boolean  hasStock = true;
    // 2、sku的图片信息 pms_sku_images
    List<SkuImagesEntity> images;
    // 3、获取spu的销售属性组
    List<SkuItemSaleAttrVo> saleAttr;
    // 4、获取spu的介绍
    SpuInfoDescEntity desc;
    // 5、获取spu的规格参数信息
    List<SpuItemAttrGroupVo> groupAttrs;
}
```



```java
/**
 * @author gcq
 * @Create 2020-11-08
 */
@Data
public class SkuItemSalAttrVo {
    private Long attrId;
    private String attrName;
    private List<AttrValueWithSkuIdVo> attrValues;
}
```



```java
/**
 * @author gcq
 * @Create 2020-11-08
 */
@Data
public class AttrValueWithSkuIdVo {

    private String attrValue;
    private String skuIds;
    
}
```

```java
@Data
@ToString
public class SpuItemAttrGroupVo {

    private String groupName;

    //attrId,attrName,attrValue
    private List<Attr> attrs;

}
```



### 9.2 查询详情

#### (1) 总体思路

```java
@GetMapping("/{skuId}.html")
public String skuItem(@PathVariable("skuId") Long skuId, Model model) {
    SkuItemVo skuItemVo=skuInfoService.item(skuId);
    model.addAttribute("item", skuItemVo);
    return "item";
}

 	@Override
    public SkuItemVo item(Long skuId) {
        SkuItemVo skuItemVo = new SkuItemVo();
        //1、sku基本信息的获取  pms_sku_info
        SkuInfoEntity skuInfoEntity = this.getById(skuId);
        skuItemVo.setInfo(skuInfoEntity);
        Long spuId = skuInfoEntity.getSpuId();
        Long catalogId = skuInfoEntity.getCatalogId();


        //2、sku的图片信息    pms_sku_images
        List<SkuImagesEntity> skuImagesEntities = skuImagesService.list(new QueryWrapper<SkuImagesEntity>().eq("sku_id", skuId));
        skuItemVo.setImages(skuImagesEntities);

        //3、获取spu的销售属性组合-> 依赖1 获取spuId
        List<SkuItemSaleAttrVo> saleAttrVos=skuSaleAttrValueService.listSaleAttrs(spuId);
        skuItemVo.setSaleAttr(saleAttrVos);

        //4、获取spu的介绍-> 依赖1 获取spuId
        SpuInfoDescEntity byId = spuInfoDescService.getById(spuId);
        skuItemVo.setDesc(byId);

        //5、获取spu的规格参数信息-> 依赖1 获取spuId catalogId
        List<SpuItemAttrGroupVo> spuItemAttrGroupVos=productAttrValueService.getProductGroupAttrsBySpuId(spuId, catalogId);
        skuItemVo.setGroupAttrs(spuItemAttrGroupVos);
        //TODO 6、秒杀商品的优惠信息

        return skuItemVo;
    }
```

#### (2) 获取spu的销售属性

由于我们需要获取该spu下所有sku的销售属性，因此我们需要先从`pms_sku_info`查出该`spuId`对应的`skuId`，

<img src="/Snipaste_2020-09-21_00-08-20.png" style="zoom: 33%;" />

再在`pms_sku_sale_attr_value`表中查出上述`skuId`对应的属性

<img src="/Snipaste_2020-09-21_00-07-08.png" style="zoom:38%;" />

因此我们需要使用连表查询，并且通过分组将单个属性值对应的多个`spuId`组成集合，效果如下

<img src="/Snipaste_2020-09-21_00-11-39.png" style="zoom: 50%;" />

==为什么要设计成这种模式呢？==

因为这样可以在页面显示切换属性时，快速得到对应skuId的值，比如白色对应的`sku_ids`为30,29，8+128GB对应的`sku_ids`为29,31,27，那么销售属性为`白色、8+128GB`的商品的`skuId`则为二者的交集29

```xml
<resultMap id="SkuItemSaleAttrMap" type="io.niceseason.gulimall.product.vo.SkuItemSaleAttrVo">
        <result property="attrId" column="attr_id"/>
        <result property="attrName" column="attr_name"/>
        <collection property="attrValues" ofType="io.niceseason.gulimall.product.vo.AttrValueWithSkuIdVo">
            <result property="attrValue" column="attr_value"/>
            <result property="skuIds" column="sku_ids"/>
        </collection>
    </resultMap>

    <select id="listSaleAttrs" resultMap="SkuItemSaleAttrMap">
        SELECT attr_id,attr_name,attr_value,GROUP_CONCAT(info.sku_id) sku_ids FROM pms_sku_info info
        LEFT JOIN pms_sku_sale_attr_value ssav ON info.sku_id=ssav.sku_id
        WHERE info.spu_id=#{spuId}
        GROUP BY ssav.attr_id,ssav.attr_name,ssav.attr_value
    </select>
```

#### (3) 获取spu的规格参数信息

由于需要通过`spuId`和`catalogId`查询对应规格参数，所以我们需要通过`pms_attr_group表`获得`catalogId`和`attrGroupName`



<img src="/Snipaste_2020-09-21_00-24-35.png" style="zoom: 50%;" />

然后通过` pms_attr_attrgroup_relation`获取分组对应属性id

<img src="/Snipaste_2020-09-21_00-26-48.png" style="zoom: 50%;" />

再到`   pms_product_attr_value`查询spuId对应的属性

<img src="/Snipaste_2020-09-21_00-27-51.png" style="zoom:50%;" />

最终sql效果,联表含有需要的所有属性

<img src="/Snipaste_2020-09-21_00-29-01.png" style="zoom:50%;" />

```java
@Mapper
public interface ProductAttrValueDao extends BaseMapper<ProductAttrValueEntity> {

    List<SpuItemAttrGroupVo> getProductGroupAttrsBySpuId(@Param("spuId") Long spuId, @Param("catalogId") Long catalogId);
}
```

```xml
<resultMap id="ProductGroupAttrsMap" type="io.niceseason.gulimall.product.vo.SpuItemAttrGroupVo">
    <result property="groupName" column="attr_group_name"/>
    <collection property="attrs" ofType="io.niceseason.gulimall.product.vo.Attr">
        <result property="attrId" column="attr_id"/>
        <result property="attrName" column="attr_name"/>
        <result property="attrValue" column="attr_value"/>
    </collection>
</resultMap>

<select id="getProductGroupAttrsBySpuId" resultMap="ProductGroupAttrsMap">
    SELECT ag.attr_group_name,attr.attr_id,attr.attr_name,attr.attr_value
    FROM pms_attr_attrgroup_relation aar 
    LEFT JOIN pms_attr_group ag ON aar.attr_group_id=ag.attr_group_id
    LEFT JOIN pms_product_attr_value attr ON aar.attr_id=attr.attr_id
    WHERE attr.spu_id = #{spuId} AND ag.catelog_id = #{catalogId}
</select>
```

### 3. 使用异步编排

为了使我们的任务进行的更快，我们可以让查询的各个子任务多线程执行，但是由于各个任务之间可能有相互依赖的关系，因此就涉及到了异步编排。

在这次查询中spu的销售属性、介绍、规格参数信息都需要`spuId`,因此依赖sku基本信息的获取,所以我们要让这些任务在1之后运行。因为我们需要1运行的结果，因此调用`thenAcceptAsync()`可以接受上一步的结果且没有返回值。

最后时，我们需要调用`get()`方法使得所有方法都已经执行完成

```java
public SkuItemVo item(Long skuId) {
    SkuItemVo skuItemVo = new SkuItemVo();
    CompletableFuture<SkuInfoEntity> infoFuture = CompletableFuture.supplyAsync(() -> {
        //1、sku基本信息的获取  pms_sku_info
        SkuInfoEntity skuInfoEntity = this.getById(skuId);
        skuItemVo.setInfo(skuInfoEntity);
        return skuInfoEntity;
    }, executor);

    //2、sku的图片信息    pms_sku_images
    CompletableFuture<Void> imageFuture = CompletableFuture.runAsync(() -> {
        List<SkuImagesEntity> skuImagesEntities = skuImagesService.list(new QueryWrapper<SkuImagesEntity>().eq("sku_id", skuId));
        skuItemVo.setImages(skuImagesEntities);
    }, executor);


    //3、获取spu的销售属性组合-> 依赖1 获取spuId
    CompletableFuture<Void> saleFuture = infoFuture.thenAcceptAsync((info) -> {
        List<SkuItemSaleAttrVo> saleAttrVos = skuSaleAttrValueService.listSaleAttrs(info.getSpuId());
        skuItemVo.setSaleAttr(saleAttrVos);
    }, executor);


    //4、获取spu的介绍-> 依赖1 获取spuId
    CompletableFuture<Void> descFuture = infoFuture.thenAcceptAsync((info) -> {
        SpuInfoDescEntity byId = spuInfoDescService.getById(info.getSpuId());
        skuItemVo.setDesc(byId);
    }, executor);


    //5、获取spu的规格参数信息-> 依赖1 获取spuId catalogId
    CompletableFuture<Void> attrFuture = infoFuture.thenAcceptAsync((info) -> {
        List<SpuItemAttrGroupVo> spuItemAttrGroupVos=productAttrValueService.getProductGroupAttrsBySpuId(info.getSpuId(), info.getCatalogId());
        skuItemVo.setGroupAttrs(spuItemAttrGroupVos);
    }, executor);

    //TODO 6、秒杀商品的优惠信息

    //等待所有任务执行完成
    try {
        CompletableFuture.allOf(imageFuture, saleFuture, descFuture, attrFuture).get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }

    return skuItemVo;
}
```



### 9.3 sku 组合切换

```javascript
 // sku 组合切换
    $(".sku_attr_value").click(function () {
        var skus = new Array();
        // 1、点击的元素添加上自定义属性,为了识别我们刚在被点击的
        $(this).addClass("clicked")
        // 1.1、当前被选中的属性的skus信息
        var curr = $(this).attr("skus").split(",");
        // 2、添加到数组中
        skus.push(curr);
        // 3、去掉同一行的所有的 checked  a->dd->dl
        $(this).parent().parent().find(".sku_attr_value").removeClass("checked")

        // 4、其他项也需要添加到数组里面
        $("a[class='sku_attr_value checked']").each(function () {
            skus.push($(this).attr("skus").split(","))
        })
        
        // 5、取出他们的交集 jquery 中使用 filter方法来取出他们的交集
        // 5.1 属性项可能有多个，内存、版本等等，所以需要遍历
        var filterEle = skus[0]
        for (var i = 1; i < skus.length; i++) {
            filterEle = $(filterEle).filter(skus[i])
        }
        // 6、跳转到交集页面
        console.log(filterEle[0])
        location.href = "http://item.gulimall.com/" + filterEle[0] + ".html"
    })
    $(function () {
        // 默认父元素是灰色
        $(".sku_attr_value").parent().css({"border": "solid 1px #CCC"});
        // 拥有checked样式表示选中当前元素
        $("a[class='sku_attr_value checked']").parent().css({"border": "solid 1px red"});
    }) 	
```



# 10、商品业务 & 认证服务

### 10.1 环境搭建

- 为登录和注册创建一个服务，创建`gulimall-auth-server`模块，导依赖
- 引入`login.html`和`reg.html`，将提供的前端放到  `templates` 目录下
- 并把静态资源放到nginx的static目录下

![image-20201110084252039](image-20201110084252039.png)

### 10.2 前端验证码倒计时

#### 

<img src="/Snipaste_2020-09-22_19-13-55.png" style="zoom:38%;" />

定义id 使用 `Jquery` 触发点击事件

![image-20201110084521166](image-20201110084521166.png)

Jquery

```javascript
$(function () {
    /**
     * 验证码发送
     */
    $("#sendCode").click(function () {
        //判断是否有该样式
        if ($(this).hasClass("disabled")) {
            // 正在倒计时
        } else {
            // 发送验证码
            $.get("/sms/sendCode?phone=" + $("#phoneNum").val(), function (data) {
                if (data.code != 0) {
                    alert(data.msg)
                }
            })
            timeoutChangeStyle();
        }
    })
})
// 60秒
var num = 60;
function timeoutChangeStyle() {
    // 先添加样式，防止重复点击
    $("#sendCode").attr("class", "disabled")
    // 到达0秒后 说明倒计时完成，重置时间，去除样式
    if (num == 0) {
        $("#sendCode").text("发送验证码")
        num = 60;
        // 时间到达后清除样式
        $("#sendCode").attr("class", "");
    } else {
        var str = num + "s 后再次发送"
        $("#sendCode").text(str);
        setTimeout("timeoutChangeStyle()", 1000);
    }
    num--;
}
```

对应效果

![image-20201110084733372](image-20201110084733372.png)

### 10.3 整合短信验证码

#### 1、短信验证我们选择的是阿里云的短信服务

![image-20201110084936446](image-20201110084936446.png)

#### 2、选择对应短信服务进行开通

在云市场就能看到购买的服务

![image-20201110085141506](image-20201110085141506.png)

#### 3、验证短信功能是否能发送

在购买短信的页面，能进行调试短信

![image-20201110085315288](image-20201110085315288.png)

输入对应手机号，appCode 具体功能不做演示

![image-20201110085348103](image-20201110085348103.png)

#### 4、使用 Java 测试短信是否能进行发送

往下拉找到对应 Java 代码

**注意：**

​	服务商提供的**接口地址**，**请求参数**都不同，请参考服务商提供的测试代码

```java
@Test
public void contextLoads() {
   String host = "http://dingxin.market.alicloudapi.com";
	    String path = "/dx/sendSms";
	    String method = "POST";
	    String appcode = "你自己的AppCode";
	    Map<String, String> headers = new HashMap<String, String>();
	    //最后在header中的格式(中间是英文空格)为Authorization:APPCODE 83359fd73fe94948385f570e3c139105
	    headers.put("Authorization", "APPCODE " + appcode);
	    Map<String, String> querys = new HashMap<String, String>();
	    querys.put("mobile", "159xxxx9999");
	    querys.put("param", "code:1234");
	    querys.put("tpl_id", "TP1711063");
	    Map<String, String> bodys = new HashMap<String, String>();


	    try {
	    	/**
	    	* 重要提示如下:
	    	* HttpUtils请从
	    	* https://github.com/aliyun/api-gateway-demo-sign-java/blob/master/src/main/java/com/aliyun/api/gateway/demo/util/HttpUtils.java
	    	* 下载
	    	*
	    	* 相应的依赖请参照
	    	* https://github.com/aliyun/api-gateway-demo-sign-java/blob/master/pom.xml
	    	*/
	    	HttpResponse response = HttpUtils.doPost(host, path, method, headers, querys, bodys);
	    	System.out.println(response.toString());
	    	//获取response的body
	    	//System.out.println(EntityUtils.toString(response.getEntity()));
	    } catch (Exception e) {
	    	e.printStackTrace();
	    }
}
```

需要导入对应工具类，参照注释就行

在`gulimall-third-party`中编写发送短信组件,其中`host`、`path`、`appcode`可以在配置文件中使用前缀`spring.cloud.alicloud.sms`进行配置

```java
@Data
@ConfigurationProperties(prefix = "spring.cloud.alicloud.sms")
@Controller
public class SmsComponent {

    private String host;
    private String path;
    private String appcode;

    public void sendCode(String phone,String code) {
//        String host = "http://dingxin.market.alicloudapi.com";
//        String path = "/dx/sendSms";
        String method = "POST";
//        String appcode = "你自己的AppCode";
        Map<String, String> headers = new HashMap<String, String>();
        //最后在header中的格式(中间是英文空格)为Authorization:APPCODE 83359fd73fe94948385f570e3c139105
        headers.put("Authorization", "APPCODE " + appcode);
        Map<String, String> querys = new HashMap<String, String>();
        querys.put("mobile",phone);
        querys.put("param", "code:"+code);
        querys.put("tpl_id", "TP1711063");
        Map<String, String> bodys = new HashMap<String, String>();


        try {
            /**
             * 重要提示如下:
             * HttpUtils请从
             * https://github.com/aliyun/api-gateway-demo-sign-java/blob/master/src/main/java/com/aliyun/api/gateway/demo/util/HttpUtils.java
             * 下载
             *
             * 相应的依赖请参照
             * https://github.com/aliyun/api-gateway-demo-sign-java/blob/master/pom.xml
             */
            HttpResponse response = HttpUtils.doPost(host, path, method, headers, querys, bodys);
            System.out.println(response.toString());
            //获取response的body
            //System.out.println(EntityUtils.toString(response.getEntity()));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

编写controller，给别的服务提供远程调用发送验证码的接口

```java
@Controller
@RequestMapping(value = "/sms")
public class SmsSendController {

    @Resource
    private SmsComponent smsComponent;

    /**
     * 提供给别的服务进行调用
     * @param phone 电话号码
     * @param code 验证码
     * @return
     */
    @ResponseBody
    @GetMapping(value = "/sendCode")
    public R sendCode(@RequestParam("phone") String phone, @RequestParam("code") String code) {

        //发送验证码
        smsComponent.sendCode(phone,code);
        System.out.println(phone+code);
        return R.ok();
    }
}
```



### 10.4 验证码防刷校验

- 由于发送验证码的接口暴露，为了防止恶意攻击，我们不能随意让接口被调用。

  * 在redis中以`phone-code`将电话号码和验证码进行存储并将当前时间与code一起存储
    * 如果调用时以当前`phone`取出的v不为空且当前时间在存储时间的60s以内，说明60s内该号码已经调用过，返回错误信息
    * 60s以后再次调用，需要删除之前存储的`phone-code`
    * code存在一个过期时间，我们设置为10min，10min内验证该验证码有效

```java
/**
 * 发送短信验证码
 * @param phone 手机号
 * @return
 */
@GetMapping("/sms/sendCode")
@ResponseBody
public R sendCode(@RequestParam("phone") String phone) {
    // 接口防刷,在redis中缓存phone-code
    // 先从redis中拿取
    String redisCode = redisTemplate.opsForValue().get(AuthServerConstant.SMS_CODE_CACHE_PREFIX + phone);
    if(!StringUtils.isEmpty(redisCode)) {
        // 拆分
        long l = Long.parseLong(redisCode.split("_")[1]);
        // 当前系统事件减去之前验证码存入的事件 小于60000毫秒=60秒
        if (System.currentTimeMillis() -l < 60000) {
            // 60秒内不能再发
            R.error(BizCodeEnume.SMS_CODE_EXCEPTION.getCode(),BizCodeEnume.SMS_CODE_EXCEPTION.getMsg());
        }
    }
    // 2、验证码的再次效验
    // 数据存入 =》redis key-phone value - code sms:code:131xxxxx - >45678
    String code = UUID.randomUUID().toString().substring(0,5).toUpperCase();
    // 拼接验证码
    String substring = code+"_" + System.currentTimeMillis();
    // redis缓存验证码并设置过期时间 防止同一个phone在60秒内发出多次验证吗
    redisTemplate.opsForValue().set(AuthServerConstant.SMS_CODE_CACHE_PREFIX+phone,substring,10, TimeUnit.MINUTES);

    // 调用第三方服务发送验证码
    thirdPartFeignService.sendCode(phone,code);
    return R.ok();
}
```



### 10.5 一步一坑注册页环境

#### 1、编写 vo 接收页面提交

- 使用到了 JSR303校验

```java
/**
 * 注册数据封装Vo
 * @author gcq
 * @Create 2020-11-09
 */
@Data
public class UserRegistVo {
    @NotEmpty(message = "用户名必须提交")
    @Length(min = 6,max = 18,message = "用户名必须是6-18位字符")
    private String userName;

    @NotEmpty(message = "密码必须填写")
    @Length(min = 6,max = 18,message = "密码必须是6-18位字符")
    private String password;

    @NotEmpty(message = "手机号码必须提交")
    @Pattern(regexp = "^[1]([3-9])[0-9]{9}$",message = "手机格式不正确")
    private String phone;

    @NotEmpty(message = "验证码必须填写")
    private String code;
}
```

#### 2、页面提交数据与Vo一致

设置 `name` 属性与 `Vo` 一致，方便将传递过来的数据转换成 JSON

![image-20201110100732631](image-20201110100732631.png)

#### 3、数据校验

在`gulimall-auth-server`服务中编写注册的主体逻辑

* 若JSR303校验未通过，则通过`BindingResult`封装错误信息，并重定向至注册页面
* 若通过JSR303校验，则需要从`redis`中取值判断验证码是否正确，正确的话通过会员服务注册
* 会员服务调用成功则重定向至登录页，否则封装远程服务返回的错误信息返回至注册页面

注： `RedirectAttributes`可以通过session保存信息并在重定向的时候携带过去

```java
/**
 * //TODO 重定向携带数据，利用session原理，将数据放在session中，
 * 只要跳转到下一个页面取出这个数据，session中的数据就会删掉
 * //TODO分布式下 session 的问题
 * RedirectAttributes redirectAttributes 重定向携带数据
 * redirectAttributes.addFlashAttribute("errors", errors); 只能取一次
 * @param vo 数据传输对象
 * @param result 用于验证参数
 * @param redirectAttributes 数据重定向
 * @return
 */
@PostMapping("/regist")
public String regist(@Valid UserRegistVo vo, BindingResult result,
                     RedirectAttributes redirectAttributes) {
    // 校验是否通过
    if (result.hasErrors()) {
        // 拿到错误信息转换成Map
        Map<String, String> errors = result.getFieldErrors().stream().collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage));
        //用一次的属性
        redirectAttributes.addFlashAttribute("errors",errors);
        // 校验出错，转发到注册页
        return "redirect:http://auth.gulimall.com/reg.html";
    }

    // 将传递过来的验证码 与 存redis中的验证码进行比较
    String code = vo.getCode();
    String s = redisTemplate.opsForValue().get(AuthServerConstant.SMS_CODE_CACHE_PREFIX + vo.getPhone());
    if (!StringUtils.isEmpty(s)) {
        // 验证码和redis中的一致
        if(code.equals(s.split("_")[0])) {
            // 删除验证码：令牌机制
            redisTemplate.delete(AuthServerConstant.SMS_CODE_CACHE_PREFIX + vo.getPhone());
            // 调用远程服务，真正注册
            R r = memberFeignService.regist(vo);
            if (r.getCode() == 0) {
                // 远程调用注册服务成功，重定向登录页
                return "redirect:http://auth.gulimall.com/login.html";
            } else {
                //调用失败，返回注册页并显示错误信息
                Map<String, String> errors = new HashMap<>();
                errors.put("msg",r.getData(new TypeReference<String>(){}));
                redirectAttributes.addFlashAttribute("errors", errors);
                return "redirect:http://auth.gulimall.com/reg.html";
            }
        } else {
            Map<String, String> errors = new HashMap<>();
            errors.put("code", "验证码错误");
            redirectAttributes.addFlashAttribute("code", "验证码错误");
            // 校验出错，转发到注册页
            return "redirect:http://auth.gulimall.com/reg.html";
        }
    } else {
        //2.2 验证码错误
        Map<String, String> errors = new HashMap<>();
        errors.put("code", "验证码错误");
        redirectAttributes.addFlashAttribute("code", "验证码错误");
        // 校验出错，转发到注册页
        return "redirect:http://auth.gulimall.com/reg.html";
    }
}
```

#### 4、前端页面接收错误信息

![image-20201110101306173](image-20201110101306173.png)

#### 5、异常机制 & 用户注册

通过`gulimall-member`会员服务注册逻辑

* 通过异常机制判断当前注册会员名和电话号码是否已经注册，如果已经注册，则抛出对应的自定义异常，并在返回时封装对应的错误信息
* 如果没有注册，则封装传递过来的会员信息，并设置默认的会员等级、创建时间

Controller

```java
/**
 * 注册
 * @param registVo
 * @return
 */
@PostMapping("/regist")
public R regist(@RequestBody MemberRegistVo registVo) {
    try {
        memberService.regist(registVo);
    } catch (PhoneExsitException e) {
        // 返回对应的异常信息
       return R.error(BizCodeEnume.PHONE_EXIST_EXCEPTION.getCode(),BizCodeEnume.PHONE_EXIST_EXCEPTION.getMsg());
    } catch (UserNameExistException e) {
        return R.error(BizCodeEnume.USER_EXIST_EXCEPTION.getCode(),BizCodeEnume.USER_EXIST_EXCEPTION.getMsg());
    }
    return R.ok();
}
```

```java
@Override
public void register(MemberRegisterVo registerVo) {
    //1 检查电话号是否唯一
    checkPhoneUnique(registerVo.getPhone());
    //2 检查用户名是否唯一
    checkUserNameUnique(registerVo.getUserName());
    //3 该用户信息唯一，进行插入
    MemberEntity entity = new MemberEntity();
    //3.1 保存基本信息
    entity.setUsername(registerVo.getUserName());
    entity.setMobile(registerVo.getPhone());
    entity.setCreateTime(new Date());
    //3.2 使用加密保存密码
    BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    String encodePassword = passwordEncoder.encode(registerVo.getPassword());
    entity.setPassword(encodePassword);
    //3.3 设置会员默认等级
    //3.3.1 找到会员默认登记
    MemberLevelEntity defaultLevel = memberLevelService.getOne(new QueryWrapper<MemberLevelEntity>().eq("default_status", 1));
    //3.3.2 设置会员等级为默认
    entity.setLevelId(defaultLevel.getId());

    // 4 保存用户信息
    this.save(entity);
}

private void checkUserNameUnique(String userName) {
    Integer count = baseMapper.selectCount(new QueryWrapper<MemberEntity>().eq("username", userName));
    if (count > 0) {
        throw new UserExistException();
    }
}

private void checkPhoneUnique(String phone) {
    Integer count = baseMapper.selectCount(new QueryWrapper<MemberEntity>().eq("mobile", phone));
    if (count > 0) {
        throw new PhoneNumExistException();
    }
}
```

 此处引入一个问题

- 密码是直接存入数据库吗？ 这样子会导致数据的不安全，
- 引出了使用 MD5进行加密，但是MD5加密后，别人任然可以暴力破解
- 可以使用加盐的方式，将密码加密后，得到一串随机字符，
- 随机字符和密码和进行验证相同结果返回true否则false



至此注册相关结束~



### 10.6 账号密码登录

#### 1、定义 Vo 接收数据提交

```java
/**
 * @author gcq
 * @Create 2020-11-10
 */
@Data
public class UserLoginVo {
    private String loginacct;
    private String password;
}
```

同时需要保证前端页面提交字段与 Vo 类中一致

在`gulimall-auth-server`模块中的主体逻辑

* 通过会员服务远程调用登录接口
  * 如果调用成功，重定向至首页
  * 如果调用失败，则封装错误信息并携带错误信息重定向至登录页

```java
@RequestMapping("/login")
public String login(UserLoginVo vo,RedirectAttributes attributes){
    R r = memberFeignService.login(vo);
    if (r.getCode() == 0) {
        return "redirect:http://gulimall.com/";
    }else {
        String msg = (String) r.get("msg");
        Map<String, String> errors = new HashMap<>();
        errors.put("msg", msg);
        attributes.addFlashAttribute("errors", errors);
        return "redirect:http://auth.gulimall.com/login.html";
    }
}
```

* 

#### 2、在 Member 服务中编写接口

在`gulimall-member`模块中完成登录

* 当数据库中含有以当前登录名为用户名或电话号且密码匹配时，验证通过，返回查询到的实体

* 否则返回null，并在controller返回`用户名或密码错误`

  ```java
  @RequestMapping("/login")
  public R login(@RequestBody MemberLoginVo loginVo) {
      MemberEntity entity=memberService.login(loginVo);
      if (entity!=null){
          return R.ok();
      }else {
          return R.error(BizCodeEnum.LOGINACCT_PASSWORD_EXCEPTION.getCode(), BizCodeEnum.LOGINACCT_PASSWORD_EXCEPTION.getMsg());
      }
  }
  ```

  

```java
@Override
public MemberEntity login(MemberLoginVo vo) {
    String loginacct = vo.getLoginacct();
    String password = vo.getPassword();

    // 1、以用户名或电话号登录的进行查询，去数据库查询 select * from  ums_member where username=? or mobile =?
    MemberDao memberDao = this.baseMapper;
    MemberEntity memberEntity = memberDao.selectOne(new QueryWrapper<MemberEntity>()
            .eq("username", loginacct).or().
                    eq("mobile", loginacct));
    if (memberDao == null) {
        // 登录失败
        return null;
    } else {
        // 获取数据库的密码
        String passwordDB = memberEntity.getPassword();
        BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        // 和用户密码进行校验
        boolean matches = passwordEncoder.matches(password, passwordDB);
        if(matches) {
            // 密码验证成功 返回对象
            return memberEntity;
        } else {
            return null;
        }
    }
}
```





### 11.7 分布式 Session不共享不同步问题

#### 1、Session原理及分布式下的问题

session存储在服务端，jsessionId存在客户端，每次通过`jsessionid`取出保存的数据

问题：但是正常情况下`session`不可跨域，它有自己的作用范围

我们在auth.gulimall.com中保存session，但是网址跳转到 gulimall.com中，取不出auth.gulimall.com中保存的session，这就造成了微服务下的session不同步问题

![image-20201111103637615](image-20201111103637615.png)

#### 1、Session同步解决方案-分布式下session共享问题

**session要能在不同服务和同服务的集群的共享**

同一个服务复制多个，但是session还是只能在一个服务上保存，浏览器也是只能读取到一个服务的session

![image-20201111104758917](image-20201111104758917.png)



#### 2、Session共享问题解决-session复制





![image-20201111104851977](image-20201111104851977.png)

#### 3、Session共享问题解决-客户端存储

![image-20201111104913888](image-20201111104913888.png)

#### 4、Session共享问题解决-hash一致性

![image-20201111105039741](image-20201111105039741.png)

#### 5、Session共享问题解决-统一存储

![image-20201111105135178](image-20201111105135178.png)

### 11.8 SpringSession整合redis

#### 1、官网文档 阅读

- 进入到 Spring Framework

![image-20201111144109273](image-20201111144109273.png)

##### 2、选择Spring Session文档

![image-20201111144350506](image-20201111144350506.png)

![image-20201111144438592](image-20201111144438592.png)

##### 3、开始使用Spring Session

![image-20201111144639786](image-20201111144639786.png)

![image-20201111144718176](image-20201111144718176.png)

#### 2、整合SpringBoot

##### 1、环境搭建

**添加Pom.xml依赖**

![image-20201111144914600](image-20201111144914600.png)

```xml
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
```



**application.yml 配置**

![image-20201111145601673](image-20201111145601673.png)

```yaml
spring:
  redis:
    host: 192.168.56.102
  session:
    store-type: redis
```


##### 3、reids配置

![image-20201111150056671](image-20201111150056671.png)

##### 4、启动类加上 @EnableRedisHttpSession

```java
// 整合spring session
@EnableRedisHttpSession
public class GulimallAuthServerApplication {
```

#### 3、自定义 SpringSession 完成 Session 子域共享

##### `CookieSerializer`

api文档参考：https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/index.html#api-cookieserializer

![image-20210101124234037](image-20210101124234037.png)

##### 指定redis序列化

文档地址：

https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/index.html#api-redisindexedsessionrepository-config

![image-20210101124513827](image-20210101124513827.png)

redis中json序列化

官网文档地址：https://docs.spring.io/spring-session/docs/2.4.1/reference/html5/index.html#samples

![image-20210101125216426](image-20210101125216426.png)

提供的实例：

https://github.com/spring-projects/spring-session/blob/2.4.1/spring-session-samples/spring-session-sample-boot-redis-json/src/main/java/sample/config/SessionConfig.java

![image-20210101125303807](image-20210101125303807.png)

但是现在还有一些问题：

- 序列化的问题
- cookie的domain的问题

**扩大session作用域**

* 由于默认使用jdk进行序列化，通过导入`RedisSerializer`修改为json序列化

* 并且通过修改`CookieSerializer`扩大`session`的作用域至`**.gulimall.com`

```java

/**
 * SpringSession整合子域
 * 以及redis数据存储为json
 * @author gcq
 * @Create 2020-11-11
 */
@Configuration
public class GulimallSessionConfig {

    /**
     * 设置cookie信息
     * @return
     */
    @Bean
    public CookieSerializer CookieSerializer(){
        DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        // 设置一个域名的名字
        cookieSerializer.setDomainName("gulimall.com");
        // cookie的路径
        cookieSerializer.setCookieName("GULIMALLSESSION");
        return cookieSerializer;
    }

    /**
     * 设置json转换
     * @return
     */
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        // 使用jackson提供的转换器
        return new GenericJackson2JsonRedisSerializer();
    }

}
```

![](/Snipaste_2020-09-23_20-45-36.png)

#### 4、SpringSession 原理  - 装饰者模式

```java
/**
 * 核心原理
 * 1、@EnableRedisHttpSession导入RedisHttpSessionConfiguration配置
 *      1、给容器中添加了一个组件
 *          sessionRepository = 》》》【RedisOperationsSessionRepository】 redis 操作 session session的增删改查封装类
 *      2、SessionRepositoryFilter==>:session存储过滤器，每个请求过来必须经过Filter
 *          1、创建的时候，就自动从容器中获取到了SessionRepostiory
 *          2、原始的request,response都被包装了 SessionRepositoryRequestWrapper、SessionRepositoryResponseWrapper
 *          3、以后获取session.request.getSession()
 *              SessionRepositoryResponseWrapper
 *          4、wrappedRequest.getSession() ==>SessionRepository
 *
 *          装饰者模式
 *          spring-redis的相关功能:
 *                 执行session相关操作后，redis里面存储的时间也会刷新
 */
```

核心源码是：

- `SessionRepositoryFilter` 类下面的 `doFilterInternal` 方法
- 及那个 `request`、`response` 包装成 `SessionRepositoryRequestWrapper`

* 原生的获取`session`时是通过`HttpServletRequest`获取的
* 这里对request进行包装，并且重写了包装request的`getSession()`方法

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
      HttpServletResponse response, FilterChain filterChain)
      throws ServletException, IOException {
   request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

    //对原生的request、response进行包装
   SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(
         request, response, this.servletContext);
   SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(
         wrappedRequest, response);

   try {
       // 包装后的对象应用到了我们后面的整个执行链
      filterChain.doFilter(wrappedRequest, wrappedResponse);
   }
   finally {
      wrappedRequest.commitSession();
   }
}
```

#### (5)session的保存

```java
@GetMapping({"/login.html","/","/index","/index.html"}) // auth
public String loginPage(HttpSession session){
    // 从会话从获取loginUser
    Object attribute = session.getAttribute(AuthServerConstant.LOGIN_USER);// "loginUser";
    System.out.println("attribute:"+attribute);
    if(attribute == null){
        return "login";
    }
    System.out.println("已登陆过，重定向到首页");
    return "redirect:http://gulimall.com";
}


@PostMapping("/login") // auth
public String login(UserLoginVo userLoginVo,
                    RedirectAttributes redirectAttributes,
                    HttpSession session){
    // 远程登录
    R r = memberFeignService.login(userLoginVo);
    if(r.getCode() == 0){
        // 登录成功
        MemberRespVo respVo = r.getData("data", new TypeReference<MemberRespVo>() {});
        // 放入session  // key为loginUser
        session.setAttribute(AuthServerConstant.LOGIN_USER, respVo);//loginUser
        log.info("\n欢迎 [" + respVo.getUsername() + "] 登录");
        // 登录成功重定向到首页
        return "redirect:http://gulimall.com";
    }else {
        HashMap<String, String> error = new HashMap<>();
        // 获取错误信息
        error.put("msg", r.getData("msg",new TypeReference<String>(){}));
        redirectAttributes.addFlashAttribute("errors", error);
        return "redirect:http://auth.gulimall.com/login.html";
    }
}
```

#### 分布式登录总结

登录url：http://auth.gulimall.com/login.html  
（注意是url，不是页面。）  
判断`session`中是否有user对象

*   没有user对象，渲染login.html页面  
    用户输入账号密码后发送给 url:auth.gulimall.com/login  
    根据表单传过来的VO对象，远程调用memberFeignService验证密码
    *   如果验证失败，取出远程调用返回的错误信息，放到新的请求域，重定向到登录url
    *   如果验证成功，远程服务就返回了对应的MemberRespVo对象，  
        然后放到分布式redis-session中，key为"loginUser"，重定向到首页gulimall.com，  
        同时也会带着的GULISESSIONID
        *   重定向到非auth项目后，先经过拦截器看session里有没有loginUser对象
        *   有，放到静态threadLocal中，这样就可以操作本地内存，无需远程调用session
        *   没有，重定向到登录页
*   有user对象，代表登录过了，重定向到首页，session数据还依靠sessionID持有着

额外说明：

问题1：我们有sessionId不就可以了吗？为什么还要在session中放到User对象？  
为了其他服务可以根据这个user查数据库，只有session的话不能再次找到登录session的用户

问题2：threadlocal的作用？

他是为了放到当前session的线程里，threadlocal就是这个作用，随着线程创建和消亡。把threadlocal定义为static的，这样当前会话的线程中任何代码地方都可以获取到。如果只是在session中的话，一是每次还得去redis查询，二是去调用service还得传入session参数，多麻烦啊

问题3：cookie怎么回事？不是在config中定义了cookie的key和序列化器？

序列化器没什么好讲的，就是为了易读和来回转换。而cookie的key其实是无所谓的，只要两个项目里的key相同，然后访问同一个域名都带着该cookie即可。



# 11、单点登录和社交登录

### 11.1 社交登录



![image-20201110124933726](image-20201110124933726.png)

QQ、微博，github等网站的用户量非常大，别的网站为了简化网站的登陆和注册逻辑，引入社交登录功能

步骤

1、用户点击 QQ按钮

2、引导跳转进 QQ 授权页

![image-20201110124945111](image-20201110124945111.png)

3、用户主动点击授权，跳回之前网页



#### 11.1.1 OAuth2.0

上面社交登录的流程就是OAuth协议

OAuth（开放授权）是一个开放标准，允许用户授权第三方移动应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方移动应用或分享他们数据的所有内容，OAuth2.0是OAuth协议的延续版本，但不向后兼容OAuth 1.0即完全废止了OAuth1.0。

- **OAuth：**OAuth（开放授权）是一个开放标准，允许用户授权第三方网站访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方网站或分享他们的数据的内容

- **OAuth2.0：**对于用户相关的 OpenAPI（例如获取用户信息，动态同步，照片，日志，分享等），为了保存用户数据的安全和隐私，第三方网站访问用户数据前都需要显示向用户授权

  

![在这里插入图片描述](/1623316999-3b3dffa72b5f4df39278a74b19492e40.png)

> 微信：https://developers.weixin.qq.com/doc/oplatform/Mobile\_App/WeChat\_Login/Development\_Guide.html
>
> 客户端是
>
> 资源拥有者：用户本人
>
> 授权服务器：QQ服务器，微信服务器等。返回访问令牌
>
> 资源服务器：拿着令牌访问资源服务器看令牌合法性

文档地址：

相关流程分析

![](/1623316999-c29b8d6c5153af723b48c15793ff910f.png)

1、使用Code换取AccessToken，Code只能用一次  
2、同一个用户的accessToken一段时间是不会变化的，即使多次获取

#### 11.2 微博登录准备工作

官方版流程

![img](/oAuth2_01.gif)

##### 1、进入微博开放平台



![image-20201110154702360](image-20201110154702360.png)



##### 2、登录微博，进入微连接，选择网站接入

![image-20201110160834589](image-20201110160834589.png)



##### 3、选择立即接入

![image-20201110161001013](image-20201110161001013.png)

##### 4、创建自己的应用

![image-20201110161032203](image-20201110161032203.png)

##### 5、我们可以在开发阶段

![image-20201110161152105](image-20201110161152105.png)

##### 6、进入高级信息

![image-20201110161407018](image-20201110161407018.png)

##### 7、添加测试账号

![image-20201110161451881](image-20201110161451881.png)

##### 8、进入文档

![image-20201110161634486](image-20201110161634486.png)



#### 11.3 微博登录代码实现

##### 登陆注册流程图

看不清，放大一点

##### **微博登录流程**![image-20201231084733753](image-20201231084733753.png)

###### **注册流程**

![image-20201231084909415](image-20201231084909415.png)

###### **账号密码登录流程：**

![image-20201231012134722](image-20201231012134722.png)

###### **手机验证码发送流程：**

![image-20201231012207446](image-20201231012207446.png)

##### 11.3.1 在登陆页引导用户至授权页

##### 查看微博开放平台文档

https://open.weibo.com/wiki/%E6%8E%88%E6%9D%83%E6%9C%BA%E5%88%B6%E8%AF%B4%E6%98%8E

![image-20201111093019560](image-20201111093019560.png)



#### 

```
https://api.weibo.com/oauth2/authorize?client_id=YOUR_CLIENT_ID&response_type=code&redirect_uri=授权后跳转的uri

示例：
https://api.weibo.com/oauth2/authorize?
client_id=刚才申请的APP-KEY &
response_type=code&
redirect_uri=http://hanhunmall.com/success
```

* `client_id`: 创建网站应用时的`app key` 
* `YOUR_REGISTERED_REDIRECT_URI`: 认证完成后的跳转链接(需要和平台高级设置一致)

##### 11.3.2 点击微博登录后，跳转到微博授权页面

![image-20201111093153199](image-20201111093153199.png)

如果用户同意授权(输入账号密码)，带着code，页面跳转至回调接口 YOUR_REGISTERED_REDIRECT_URI/?code=CODE（hanhunmall.com/success/?code=CODE）

code是我们用来换取令牌的参数

用参数code换取AccessToken

#### (4) 换取token

```
https://api.weibo.com/oauth2/access_token?
client_id=YOUR_CLIENT_ID&
client_secret=YOUR_CLIENT_SECRET&
grant_type=authorization_code&
redirect_uri=YOUR_REGISTERED_REDIRECT_URI&
code=CODE
```

* `client_id`: 创建网站应用时的`app key` 
* `client_secret`: 创建网站应用时的`app secret` 
* `YOUR_REGISTERED_REDIRECT_URI`: 认证完成后的跳转链接(需要和平台高级设置一致)
* `code`：换取令牌的认证码

其中client\_id=YOUR\_CLIENT\_ID&client\_secret=YOUR\_CLIENT\_SECRET可以使用basic方式加入header中，返回值

<img src="/Snipaste_2020-09-23_12-27-09.png" style="zoom: 67%;" />

##### 11.3.4 拿到AccessToken 请求对应接口获取用户信息

https://open.weibo.com/wiki/2/users/show

结果返回json

```java
/**
     * 回调接口
     * @param code
     * @return
     * @throws Exception
     */
    @GetMapping("/oauth2.0/weibo/success")
    public String weibo(@RequestParam("code") String code) throws Exception {
        // 1、根据code换取accessToken
        Map<String, String> map = new HashMap<>();
        map.put("client_id", "1133714539");
        map.put("client_secret", "f22eb330342e7f8797a7dbe173bd9424");
        map.put("grant_type", "authorization_code");
        map.put("redirect_uri", "http://auth.gulimall.com/oauth2.0/weibo/success");
        map.put("code", code);


        HttpResponse response = HttpUtils.doPost("https://api.weibo.com",
                "/oauth2/access_token",
                "post",
                new HashMap<>(),
                map,
                new HashMap<>());

        // 状态码为200请求成功
        if (response.getStatusLine().getStatusCode() == 200 ){
            // 获取到了accessToken
            String json = EntityUtils.toString(response.getEntity());
            SocialUser socialUser = JSON.parseObject(json, SocialUser.class);
            R r = memberFeignService.OAuthlogin(socialUser);
            if (r.getCode() == 0) {
                MemberRespVo data = r.getData("data", new TypeReference<MemberRespVo>() {
                });
                log.info("登录成功:用户:{}",data.toString());

                // 2、登录成功跳转到首页
                return "redirect:http://gulimall.com";
            } else {
                // 注册失败
                return "redirect:http://auth.gulimall.com/login.html";
            }
        } else {
            // 请求失败
            // 注册失败
            return "redirect:http://auth.gulimall.com/login.html";
        }

        // 2、登录成功跳转到首页
        return "redirect:http://gulimall.com";
    }
```

##### token保存

*   登录包含两种流程，实际上包括了注册和登录
*   如果之前未使用该社交账号登录，则使用`token`调用开放api获取社交账号相关信息（头像等），注册并将结果返回
*   如果之前已经使用该社交账号登录，则更新`token`并将结果返回

```java
@Override
public MemberEntity login(SocialUser vo) {
    // 登录和注册合并逻辑
    String uid = vo.getUid();
    MemberDao memberDao = this.baseMapper;
    // 根据社交用户的uuid查询
    MemberEntity memberEntity = memberDao.selectOne(new QueryWrapper<MemberEntity>()
            .eq("social_uid", uid));
    // 能查询到该用户
    if (memberEntity != null ){
        // 更新对应值
        MemberEntity update = new MemberEntity();
        update.setId(memberEntity.getId());
        update.setAccessToken(vo.getAccess_token());
        update.setExpiresIn(vo.getExpires_in());

        memberDao.updateById(update);

        memberEntity.setAccessToken(vo.getAccess_token());
        memberEntity.setExpiresIn(vo.getExpires_in());
        return memberEntity;
    } else {
        // 2、没有查询到当前社交用户对应的记录就需要注册一个
        MemberEntity regist = new MemberEntity();
        try {
            Map<String,String> query = new HashMap<>();
            // 设置请求参数
            query.put("access_token",vo.getAccess_token());
            query.put("uid",vo.getUid());
            // 发送get请求获取社交用户信息
            HttpResponse response = HttpUtils.doGet("https://api.weibo.com/",
                    "2/users/show.json",
                    "get",
                    new HashMap<>(),
                    query);
            // 状态码为200 说明请求成功
            if (response.getStatusLine().getStatusCode() == 200){
                // 将返回结果转换成json
                String json = EntityUtils.toString(response.getEntity());
                // 利用fastjson将请求返回的json转换为对象
                JSONObject jsonObject = JSON.parseObject(json);
                // 拿到需要的值，昵称，性别，头像
                String name = jsonObject.getString("name");
                String gender = jsonObject.getString("gender");
                //.. 拿到多个信息
                regist.setNickname(name);
                regist.setGender("m".equals(gender) ? 1 : 0);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        // 设置社交用户相关信息
        regist.setSocialUid(vo.getUid());
        regist.setAccessToken(vo.getAccess_token());
        regist.setExpiresIn(vo.getExpires_in());
        memberDao.insert(regist);
        return regist;
    }
}
```

### 11.2 SSO(单点登录)

#### 1、什么是SSO

单点登录(SingleSignOn，SSO)，就是通过用户的一次性鉴别登录。当用户在身份认证服务器上登录一次以后，即可获得访问单点登录系统中其他关联系统和应用软件的权限，同时这种实现是不需要管理员对用户的登录状态或其他信息进行修改的，这意味着在多个应用系统中，用户只需一次登录就可以访问所有相互信任的应用系统。这种方式减少了由登录产生的时间消耗，辅助了用户管理，是目前比较流行的 [1]

上面解决了同域名的session问题，但如果`taobao.com`和`tianmao.com`这种不同的域名也想共享session呢？

去百度了解下：https://www.jianshu.com/p/75edcc05acfd

最终解决方案：都去中央认证器

> spring session已经解决不了不同域名的问题了。无法扩大域名

#### sso思路

记住一个核心思想：建议一个公共的登陆点server，他登录了代表这个集团的产品就登录过了

> ![img](/1623316999-8279dfca7e1d33a3c4881c69bb1fb3d6.png)
>
> 上图是CAS官网上的标准流程，具体流程如下：有两个子系统`app1`、`app2`
>
> 1.  用户访问`app1`系统，`app1`系统是需要登录的，但用户现在没有登录。
> 2.  跳转到CAS server，即SSO登录系统，**以后图中的CAS Server我们统一叫做SSO系统。** SSO系统也没有登录，弹出用户登录页。
> 3.  用户填写用户名、密码，SSO系统进行认证后，将登录状态写入SSO的session，浏览器（Browser）中写入SSO域下的Cookie。
> 4.  SSO系统登录完成后会生成一个`ST`（`Service Ticket`），然后跳转到`app1`系统，同时将ST作为参数传递给app1系统。
> 5.  `app1`系统拿到ST后，从后台向SSO发送请求，验证ST是否有效。
> 6.  验证通过后，app系统将登录状态写入session并设置app域下的Cookie。
>
> 至此，跨域单点登录就完成了。以后我们再访问app系统时，app就是登录的。接下来，我们再看看访问app2系统时的流程。
>
> 1.  用户访问`app2`系统，app2系统没有登录，跳转到SSO。
> 2.  由于SSO已经登录了，不需要重新登录认证。
> 3.  SSO生成ST，浏览器跳转到`app2`系统，并将ST作为参数传递给`app2`。
> 4.  `app2`拿到ST，后台访问SSO，验证ST是否有效。
> 5.  验证成功后，`app2`将登录状态写入session，并在`app2`域下写入Cookie。
>
> 这样，app2系统不需要走登录流程，就已经是登录了。SSO，app和app2在不同的域，它们之间的session不共享也是没问题的。
>
> **SSO系统登录后，跳回原业务系统时，带了个参数ST，业务系统还要拿ST再次访问SSO进行验证，觉得这个步骤有点多余。如果想SSO登录认证通过后，通过回调地址将用户信息返回给原业务系统，原业务系统直接设置登录状态，这样流程简单，也完成了登录，不是很好吗？**
>
> **其实这样问题时很严重的，如果我在SSO没有登录，而是直接在浏览器中敲入回调的地址，并带上伪造的用户信息，是不是业务系统也认为登录了呢？这是很可怕的。**

SSO－Single Sign On

*   server：登录服务器、8080 、ssoserver.com
*   web-sample1：项目1 、8081 、client1.com
*   web-sample2：项目1 、8082 、client2.com

3个系统即使域名不一样，想办法给三个系统同步同一个用户的票据；

*   中央认证服务器
*   其他系统都去【中央认证服务器】登录，登录成功后跳转回原服务
*   一个系统登录，都登录；一个系统登出，都登出
*   全系统统一一个sso-sessionId

> 1.  访问服务：SSO客户端发送请求访问应用系统提供的服务资源。
>
> 2.  定向认证：SSO客户端会**重定向**用户请求到SSO**服务器**。
>
> 3.  用户认证：用户身份认证。
>
> 4.  发放票据：SSO服务器会产生一个**随机的Service Ticket**。
>
> 5.  验证票据：SSO服务器验证票据Service Ticket的合法性，验证通过后，允许客户端访问服务。
>
> 6.  传输用户信息：SSO服务器验证票据通过后，传输用户认证结果信息给客户端。
>
> 7.  单点退出：用户退出单点登录。

#### 开源项目

https://gitee.com/xuxueli0323/xxl-sso

*   ssoserver.com 登录认证服务
*   client1.com
*   cleitn2.com

修改HOSTS：127.0.0.1 ssoserver.com+client1.com+client2.com

*   server：登录服务器、8080 、ssoserver.com
*   web-sample1：项目1 、8081 、client1.com
*   web-sample2：项目1 、8082 、client2.com

```bash
# 根项目下
mvn clean package -Dmaven.skip.test=true
# 打包生成了server和client包
# 启动server和client
#server8080  cient1:web-sample8081 cient2:web-sample8082
# 让client12登录一次即可
java -jar server.jar # 8080
java -jar client.jar 
# 启动多个web-sample模拟多个微服务
```

把core项目mvc install 。启动server

#### 流程

*   发送`8081/employees`请求，判断没登录就跳转到`server.com:8080/login.html`登录页，并带上现`url`
*   server登录页的时候，有之前带过来的url信息，发送登录请求的时候也把url继续带着
    *   doLogin登录成功后返回一个token（保存到server域名下）然后重定向
*   登录完后重定向到带的url参数的地址。
*   跳转回业务层的时候，业务层要能感知是登录过的，调回去的时候带个uuid，用uuid去redis里（课上说的是去server里再访问一遍，为了安全性？）看user信息，保存到它系统里自己的session
*   以后无论哪个系统访问，如果session里没有指定的内容的话，就去server登录，登录过的话已经有了server的cookie，所以不用再登录了。回来的时候就告诉了子系统应该去redis里怎么查你的用户内容

> 还得得补充一句，老师课上讲得把票据放到controller里太不合适了，你最起码得放到filter或拦截器里

#### sso解决

client1.com 8081 和 client2.com 8082 都跳转到ssoserver 8080

*   给登录服务器留下痕迹
*   登录服务器要将token信息重定向的时候，带到url地址上
*   其他系统要处理url地址上的token，只要有，将token对应的用户保存到自己的session
*   自己系统将用户保存在自己的session中

```html
<body>
    <form action="/employee" method="get">
        <input type="text" name="username" value="test">
        <button type="submit">查询</button>
    </form>
</body>
```

```java
@GetMapping(value = "/employees") // a系统
public String employees(Model model,
                        HttpSession session,
                        @RequestParam(value = "redisKey", required = false) String redisKey) {

    // 有loginToken这个参数，代表去过server端登录过了，server端里在redis里保存了个对象，而key:uuid给你发过来了
    // 有loginToken这个参数的话代表是从登录页跳回来的，而不是系统a直接传过来的
    // 你再拿着uuid再去查一遍user object，返回后设置到当前的系统session里
    // 提个问题：为什么当时不直接返回user对象，而是只返回个uuid？其实也可以，但是参数的地方也得是required = false。可能也有一些安全问题
    if (!StringUtils.isEmpty(redisKey)) { // 这个逻辑应该写到过滤器或拦截器里
        RestTemplate restTemplate=new RestTemplate();
        // 拿着token去服务器，在服务端从redis中查出来他的username
        ResponseEntity<Object> forEntity =
            restTemplate.getForEntity("http://ssoserver.com:8080/userInfo?redisKey="+ redisKey, Object.class);

        Object loginUser = forEntity.getBody();
        // 设置到自己的session中
        session.setAttribute("loginUser", loginUser);
    }
    // session里有就代表登录过 // 获得user
    Object loginUser = session.getAttribute("loginUser");

    if (loginUser == null) { // 又没有loginToken，session里又没有object，去登录页登录
        return "redirect:" + "http://ssoserver.com:8080/login.html"
            + "?url=http://clientA.com/employees";
    } else {// 登录过，执行正常的业务
        List<String> emps = new ArrayList<>();

        emps.add("张三");
        emps.add("李四");
        model.addAttribute("emps", emps);
        return "employees";
    }
}
```

##### server端

*   子系统都先去`login.html`这个请求，
    *   这个请求会告诉登录过的系统的令牌，
    *   如果没登录过就带着url重新去server端，server给一个登录页，如下

```html
<body>
<form action="/doLogin" method="post">
    <!--刚才要请求数据的url，没有也没关系，就不跳转了呗-->
    <input type="hidden" name="url" th:value="${url}">
    <!--带上当前登录的username-->
<!--    <input type="hidden" name="user" th:value="${username}">-->
        用户名：<input name="username" value="test"><br/>
        密码：<input name="password" type="password" value="test">
    <input type="submit" value="登录">
</form>
</body>
```

当点击登录之后，server端返回一个cookie，子系统重新返回去重新请去业务。于是又来server端验证，这回server端有cookie了，该cookie里有用户在redis中的key，重定向时把key带到url后面，子系统就知道怎么找用户信息了

```java
@Controller
public class LoginController {

	@Autowired
	private StringRedisTemplate stringRedisTemplate;

	@ResponseBody
	@GetMapping("/userInfo") // 得到redis中的存储过的user信息，返回给子系统的session中
	public Object userInfo(@RequestParam("redisKey") String redisKey){
		// 拿着其他域名转发过来的token去redis里查
		Object loginUser = stringRedisTemplate.opsForValue().get(redisKey);
		return loginUser;
	}


	@GetMapping("/login.html") // 子系统都来这
	public String loginPage(@RequestParam("url") String url,
							Model model,
							@CookieValue(value = "redisKey", required = false) String redisKey) {
		// 非空代表就登录过了
		if (!StringUtils.isEmpty(redisKey)) {
			// 告诉子系统他的redisKey，拿着该token就可以查redis了
			return "redirect:" + url + "?redisKey=" + redisKey;
		}
		model.addAttribute("url", url);

		// 子系统都没登录过才去登录页
		return "login";
	}

	@PostMapping("/doLogin") // server端统一认证
	public String doLogin(@RequestParam("username") String username,
						  @RequestParam("password") String password,
						  HttpServletResponse response,
						  @RequestParam(value="url",required = false) String url){
		// 确认用户后，生成cookie、redis中存储 // if内代表取查完数据库了
		if(!StringUtils.isEmpty(username) && !StringUtils.isEmpty(password)){//简单认为登录正确
			// 登录成功跳转 跳回之前的页面
			String redisKey = UUID.randomUUID().toString().replace("-", "");
			// 存储cookie， 是在server.com域名下存
			Cookie cookie = new Cookie("redisKey", redisKey);
			response.addCookie(cookie);
			// redis中存储
			stringRedisTemplate.opsForValue().set(redisKey, username+password+"...", 30, TimeUnit.MINUTES);
			// user中存储的url  重定向时候带着token
			return "redirect:" + url + "?redisKey=" + redisKey;
		}
		// 登录失败
		return "login";
	}

}

```

#### 6、使用 jwt





### 11.3 JWT

### 11.4 登录拦截器

#### 通用登录拦截器

因为订单系统必然涉及到用户信息，因此进入订单系统的请求必须是已经登录的，所以我们需要通过拦截器对未登录订单请求进行拦截

*   先注入拦截器HandlerInterceptor组件
*   在config中实现`WebMvcConfigurer接口.addInterceptor()`方法
*   拦截器和认证器的关系我在前面认证模块讲过，可以翻看，这里不赘述了

```java
@Component
public class LoginUserInterceptor implements HandlerInterceptor {

	public static ThreadLocal<MemberRespVo> threadLocal = new ThreadLocal<>();

	@Override
	public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) throws Exception {

		String uri = request.getRequestURI();
		// 这个请求直接放行
		boolean match = new AntPathMatcher().match("/order/order/status/**", uri);
		if(match){
			return true;
		}
		// 获取session
		HttpSession session = request.getSession();
		// 获取登录用户
		MemberRespVo memberRespVo = (MemberRespVo) session.getAttribute(AuthServerConstant.LOGIN_USER);
		if(memberRespVo != null){
			threadLocal.set(memberRespVo);
			return true;
		}else{
			// 没登陆就去登录
			session.setAttribute("msg", AuthServerConstant.NOT_LOGIN);
			response.sendRedirect("http://auth.gulimall.com/login.html");
			return false;
		}
	}
}
```

```JAVA
@Configuration
public class GulimallWebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/**");
    }
}
```

加上ThreadLocal共享数据，是为了登录后把用户放到本地内存，而不是每次都去远程session里查

在auth-server中登录成功后会把会话设置到session中

```java
MemberRespVo data = login.getData("data",new TypeReference<MemberRespVo>);
session.setAttribute(AuthServerConstant.LOGIN_USER,data);
```

#### 购物车的登录拦截器

因为购物车允许临时用户，所以自定义购物车拦截器

而登录操作在其他服务页面里完成即可。也可以重定向解决

具体代码去购物车博文里找



# 12、购物车

### 12.1 购物车需求

### 1. 数据模型分析

#### (1) 数据存储

购物车是一个读多写多的场景，因此放入数据库并不合适，但购物车又是需要持久化，因此这里我们选用redis存储购物车数据。

#### (2) 数据结构

<img src="/Snipaste_2020-10-01_11-08-57.png" style="zoom:38%;" />

一个购物车是由各个购物项组成的，但是我们用`List`进行存储并不合适，因为使用`List`查找某个购物项时需要挨个遍历每个购物项，会造成大量时间损耗，为保证查找速度，我们使用`hash`进行存储

![image-20201113110938713](image-20201113110938713.png)

![image-20201115154652394](image-20201115154652394.png)

每一个购物项信息，都是一个对象，基本字段包括

```java
skuId:123123, // 商品id
check:true, // 是否选中
title:"apple phone", // 商品标题
defaultImage:'', // 商品默认显示的图片
price:4999, // 商品价格
count:1, // 商品数量
totalPrice:4999 // 购物车中选中商品总价格
skuSaleVo:{} // 。。。。
```

另外，购物车中不止一条数据，因此最终会是对象的数组

```
{...},{....},{....}
```

Redis有5种不同数据结构，这里选择哪一种 比较合适呢? Map<String, List<String>> 首先**不同用户应该有独立的购物车**。

因此购物车应该以用户的作为key来存储，**Value是用户的所有购物车信息**。这样看来基本的 k-v 结构就可以了。

但是，我们对购物车中的商品进行增、删、改操作，**基本都需要根据商品id进行判断**，为了方便后期处理，我们的购物车也应该是 k-v 结构，**key 是商品 id, value 才是这个商品的购物车信息。**

综上所述，我们的购物车结构是一一个**双层Map: Map<String,Map<String,String>>**，在redis中类型为 hash 

**外头key就是用户的id,里面的key就是skuid**

redis中：

新增商品：hset gulimall:cart:7  **(key)334488[商品skuid]  (value)商品的信息**

增加商品数量：hincrby shopcar:uuid1024 334477 1

商品总数：hlen shopcar:uuid

全部选择：hgetall shopcar:uuid1024



![image-20201115155054434]( /image-20201115155054434.png)

![image-20210102202926973](image-20210102202926973.png)



### 12.2 Vo编写 & ThreadLocal身份验证

#### (1) 用户身份鉴别方式

参考京东，在点击购物车时，会为临时用户生成一个`name`为`user-key`的`cookie`临时标识，过期时间为一个月，如果手动清除`user-key`，那么临时购物车的购物项也被清除，所以`user-key`是用来标识和存储临时购物车数据的

<img src="/Snipaste_2020-10-02_09-16-11.png" style="zoom:38%;" />

#### 12.2.1 Vo 编写

购物车

```java

/**
 * 整个购物车
 * 需要计算的属性，必须重写他的get方法，保证每次获取属性都会进行计算
 * @author gcq
 * @Create 2020-11-13
 */
public class Cart {
    /**
     * 商品项
     */
    List<CartItem> items;

    /**
     * 商品数量
     */
    private Integer countNum;

    /**
     * 商品类型数量
     */
    private Integer countType;
    /**
     * 商品总价
     */
    private BigDecimal totalAmount;
    /**
     *  减免价格
     */
    private BigDecimal reduce = new BigDecimal("0");;

    public Integer getCountType() {
        int count = 0;
        if (items !=null && items.size() > 0) {
            for (CartItem item : items) {
                count+=1;
            }
        }
        return count;
    }
    public BigDecimal getTotalAmount() {
        BigDecimal amount = new BigDecimal("0");
        // 1、计算购物项总价
        if (items != null && items.size() > 0) {
            for (CartItem item : items) {
                // 拿到购物车中单个商品的总金额
                BigDecimal totalPrice = item.getTotalPrice();
                // 添加到购物总价中
                amount = amount.add(totalPrice);
            }
        }
        // 2、减去优惠总价
        BigDecimal subtract = amount.subtract(getReduce());

        return subtract;
    }

 
}
```

购物车中的实体

```java
/**
 * 购物项
 * @author gcq
 * @Create 2020-11-13
 */
public class CartItem {

    /**
     * 商品id
     */
    private Long skuId;
    /**
     * 购物车中是否选中
     */
    private Boolean check = true;
    /**
     * 商品的标题
     */
    private String title;
    /**
     * 商品的图片
     */
    private String image;
    /**
     * 商品的属性
     */
    private List<String> skuAttr;
    /**
     * 商品的价格
     */
    private BigDecimal price;
    /**
     * 商品的数量
     */
    private Integer count;
    /**
     * 购物车价格 使用自定义get、set
     */
    private BigDecimal totalPrice;
     /**
     *  计算购物项价格
     * @return
     */
    public BigDecimal getTotalPrice() {
        //价格乘以数量
        return this.price.multiply(new BigDecimal("" + this.count));
    }


```

#### (2) 使用ThreadLocal进行用户身份鉴别信息传递

* 在调用购物车的接口前，先通过session信息判断是否登录，并分别进行用户身份信息的封装，并把`user-key`放在cookie中
* 这个功能使用拦截器进行完成

需求分析：

```java
 /**
     * 浏览器有一个cookie：user-key,标识用户身份，一个月后过期
     * 如果第一次使用jd的购物车功能，都会给一个临时的用户身份
     * 浏览器以后保存，每次访问都会带上这个cookie
     *
     * 登录 session 有
     * 没登录，按照cookie中的user-key来做
     * 第一次：如果没有临时用户，帮忙创建一个临时用户 --> 使用拦截器
     * @return 
     */
```

代码

```java
**
 * 在执行目标方法之前,判断用户的登录状态，并封装给Controller目标请求
 * @author gcq
 * @Create 2020-11-13
 */
public class CartInterceptor implements HandlerInterceptor {

    public static ThreadLocal<UserInfo> threadLocal = new ThreadLocal<>();

    /**
     * 目标方法执行之前
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        UserInfo userInfo = new UserInfo();
        // 拿到session
        HttpSession session = request.getSession();
        // 查看session中是否保存了用户的值
        MemberRespVo member = (MemberRespVo) session.getAttribute(AuthServerConstant.LOGIN_USER);
        if (member != null) {
            // 用户登录
            userInfo.setUserId(member.getId());
        }
        Cookie[] cookies = request.getCookies();
        if (cookies != null && cookies.length > 0) {
            for (Cookie cookie : cookies) {
                String name = cookie.getName();
                // 拿到cookie名字进行判断如果包含 user-key 就复制到userInfo中
                if (name.equals(CartConstant.TEMP_USER_COOKIE_NAME)) {
                    userInfo.setUserKey(cookie.getValue());
                    userInfo.setTempUser(true);
                }
            }
        }
        // 如果没有临时用户一定要分配一个临时用户
        if (StringUtils.isEmpty(userInfo.getUserKey())) {
            String uuid = UUID.randomUUID().toString();
            userInfo.setUserKey(uuid);
        }
        // 全部放行
        threadLocal.set(userInfo);
        return true;
    }

    /**
     * 业务执行之后,分配临时用户，让浏览器保存
     * @param request
     * @param response
     * @param handler
     * @param modelAndView
     * @throws Exception
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        // 拿到用户信息
        UserInfo userInfo = threadLocal.get();
        // 如果没有临时用户一定要保存一个临时用户
        if (!userInfo.isTempUser()) {
            // 持续的延长临时用户的过期时间
            Cookie cookie = new Cookie(CartConstant.TEMP_USER_COOKIE_NAME, userInfo.getUserKey());
            cookie.setDomain("gulimall.com");
            cookie.setMaxAge(CartConstant.TEMP_USER_COOKIE_TIMEOUT);
            response.addCookie(cookie);
        }
    }
}
```



### 12.3 添加购物车

在页面点击加入购物车后将商品添加进购物车

![image-20201115155956396](image-20201115155956396.png)

需求分析：

首先需要在页面拿到要提交的参数，skuid，购买数量，并提交后台完成购物车数据添加

后端如何处理这个数据？

1. **通过 `skuid` 远程查询这个商品的信息**
2. **远程查询sku组合的信息**
3. **如果购物车中已经有该数据如何进行提交**？
   1. **如果有该数据，那么就行更改**，根据 skuid 从 reids 中取出数据
   2. 将数据转换成对象然后加上对应的数量，**再次转换为json存入redis**
4. **前端页面频繁添加购物车如何解决？**
   1. 请求发送过来后我们**重定向到其他的页面用来显示数据**，这时候用户刷新的话也是在其他页面进行刷新

```javascript
 /**
     * 加入到购物车
     */
    $("#addToCartA").click(function() {
        // 购买的数量
        var num = $("#numInput").val();
        // skuid
        var skuid = $(this).attr("skuId");
        location.href="http://cart.gulimall.com/addToCart?skuId=" + skuid + "&num=" + num;
        return false
    })
```

请求发送到Controller后

```java
@GetMapping("/addToCart")
public String addToCart(@RequestParam("skuId") Long skuId,
                        @RequestParam("num") Integer num,
                        RedirectAttributes  res) throws ExecutionException, InterruptedException {

    cartService.addToCart(skuId,num);
    //将数据拼接到url后面
    res.addAttribute("skuId",skuId);
    // 重定向到对应的地址
    return "redirect:http://cart.gulimall.com/addToCartSuccess.html";
}
```

Service

```java
    /**
     * 给购物车里面添加商品
     * @param skuId 商品id
     * @param num 数量
     * @throws ExecutionException
     * @throws InterruptedException
     * 如果远程查询比较慢，比如方法当中有好几个远程查询，都要好几秒以上，等整个方法返回就需要很久，这块你是怎么处理的?
     *  1、为了提交远程查询的效率，可以使用线程池的方式，异步进行请求
     *  2、要做的操作就是将所有的线程全部放到自己手写的线程池里面
     *  3、每一个服务都需要配置一个自己的线程池
     *  4、完全使用线程池来控制所有的请求
     */
    @Override
    public void addToCart(Long skuId, Integer num) throws ExecutionException, InterruptedException {
        // 获取到要操作的购物车
        BoundHashOperations<String, Object, Object> cartOps = getCartOps();
        // 1、如果购物车中已有该商品那么需要做的就是修改商品的数量，如果没有该商品需要进行添加该商品到购物车中
        String result = (String) cartOps.get(skuId.toString()); // 商品id作为键，先去Redis里面查看有没有这个相同的商品
        if (StringUtils.isEmpty(result)) { // 如果返回为空，那么说明购物车中没有这个商品，那就要执行添加
            // 2、添加商品到购物车
            // 第一个异步任务,查询当前商品信息
            CartItem cartItem = new CartItem(); // 购物车中每一个都是一个购物项，封装购物车的内容
            CompletableFuture<Void> completableFutureSkuInfo = CompletableFuture.runAsync(() -> {
                // 2.1、远程查询出当前要添加的商品信息
                // 添加哪个商品到购物车，先查询到这个商品的信息
                R r = productFeignService.getSkuInfo(skuId);
                // 获取远程返回的数据，远程数据都封装在skuInfo中
                SkuInfoVo skuInfo = r.getData("skuInfo", new TypeReference<SkuInfoVo>() {
                });
                cartItem.setCheck(true); // 添加商品到购物车，那么这个商品一定是选中的
                cartItem.setSkuId(skuInfo.getSkuId()); // 查询的是哪个商品的id，那么这个商品id就是哪个
                cartItem.setImage(skuInfo.getSkuDefaultImg()); // 当前商品的图片信息
                cartItem.setPrice(skuInfo.getPrice()); // 当前商品的价格
                cartItem.setTitle(skuInfo.getSkuTitle()); // 当前商品的标题
                cartItem.setCount(num); // 当前商品添加的数量
            }, executor);
            // 3、第二个异步任务，远程查询sku组合信息
            CompletableFuture<Void> completableFutureSkuAttr = CompletableFuture.runAsync(() -> {
                List<String> skuSaleAttrValues = productFeignService.getSkuSaleAttrValues(skuId); // 根据skuid来查
                cartItem.setSkuAttr(skuSaleAttrValues); // 远程查询出来到sku组合信息，需要在购物车中进行显示
            }, executor);
            // 4、两个异步任务都完成，才能把数据放到redis中
            CompletableFuture.allOf(completableFutureSkuInfo,completableFutureSkuAttr).get();
            // 把购物项的数据保存redis中
            String cartItemJson = JSON.toJSONString(cartItem);
            cartOps.put(skuId.toString(),cartItemJson); // 添加商品到购物车中
        } else {
            // 如果购物车中有这个商品，那么需要做的就是将商品的数量进行更改,也即是新添加的商品数量加上当前购物车中商品数量
            CartItem cartItem = JSON.parseObject(result, CartItem.class);
            cartItem.setCount(cartItem.getCount() + num);
            String cartItemJson = JSON.toJSONString(cartItem);
            // 更新redis
            cartOps.put(skuId.toString(),cartItemJson);
        }
    }
```

解决刷新问题

```java
/**
 * 跳转到成功页面
 * @param skuId
 * @param model
 * @return
 */
@GetMapping("/addToCartSuccess.html")
public String addToCartSuccessPage(@RequestParam("skuId") Long skuId,Model model) {

    // 重定向到成功页面，再次查询购物车数据
    CartItem item = cartService.getCartItem(skuId);
    model.addAttribute("item",item);
    return "success";
}
```

### 12.4 获取合并购物车

#### 12.4.1 需求分析

1. **如果用户登录，临时购物车的数据如何显示**？
   1. 将临时购物车的数据合并到用户购物车中
2. **如何显示购物车的数据？**
   1. 从 redis 取出数据放到对象中并渲染出来

#### 12.4.2 代码编写

Controller

```java
@GetMapping("/cart.html")
public String cartListPage(Model model) throws ExecutionException, InterruptedException {
    Cart cart = cartService.getCart();
    model.addAttribute("cart",cart);
    return "cartList";
}
```

Service

拼装 `Key` 从 `redis` 中查询数据

临时购物车如果有数据，当前状态是登录，那就将临时购物车的数据合并到当前用户的购物车

```java
  @Override
    public Cart getCart() throws ExecutionException, InterruptedException {
        /**
         * 需求分析1
         *   1、如果用户登录后，那么临时购物车的数据如何处理？
         *      将临时购物车的数据，放到用户购物车中进行合并
         *   2、如何显示购物车中的数据？
         *      从redis中查询到数据放到对象中，返回到页面进行渲染
         */
        Cart cart = new Cart();
        // 1、购物车的获取操作，分为登录后购物车 和 没登录购物车
        UserInfo userInfo = CartIntercept.threadLocal.get(); // 拿到共享数据
        if (userInfo.getUserId() != null) { // 登录后的购物车
            String cartKey = CART_PREFIX + userInfo.getUserKey();
            String tempCartKey = CART_PREFIX + userInfo.getUserKey();
            // 如果临时购物车中还有数据，那就需要将临时购物车合并到已登录购物车里面
            // 判断临时购物车中是否有数据
            List<CartItem> cartItems = getCartItems(tempCartKey);
            if (cartItems != null ) { // 临时购物车中有数据，需要将临时购物车的数据合并到登录后的购物车
                for (CartItem item : cartItems) { // 拿到临时购物车的所有数据，将他添加到已登录购物车里面来
                    // 调用addToCart()这个方法，他会根据登录状态进行添加，当前是登录状态
                    // 所以这个方法会将临时购物车的数据添加到已登录购物车中
                    addToCart(item.getSkuId(),item.getCount()); // 合并临时和登录后的购物车
                }
                // 合并完成后还需要将临时购物车中的数据删除
                clearCart(tempCartKey);
            }
            // 再次获取用户登录后的购物车【包含临时购物车数据和已登录购物车数据】
            List<CartItem> cartItemList = getCartItems(cartKey);
            cart.setItems(cartItemList);// 将多个购物项设置到购物车中
        } else {
            // 没有登录的购物车，拿到没有登录购物车的数据
            String cartKey = CART_PREFIX + userInfo.getUserKey();
            // 获取购物车中的所有购物项
            List<CartItem> cartItems = getCartItems(cartKey);
            cart.setItems(cartItems);
        }
        return cart;
    }
```

getCaerItems

```java
/**
     * 获取指定用户 (登录用户/临时用户) 购物车里面的数据
     * @param cartKey
     * @return
     */
    private List<CartItem> getCartItems(String cartKey) {
        BoundHashOperations<String, Object, Object> operations = redisTemplate.boundHashOps(cartKey);
        List<Object> values = operations.values(); // 拿到多个购物项
        if (values != null) {
            List<CartItem> cartItemList = values.stream().map((obj) -> {
                String s = (String) obj;
                CartItem cartItem = JSON.parseObject(s, CartItem.class);
                return cartItem;
            }).collect(Collectors.toList());
            return cartItemList;
        }
        return null;
    }
```



### 12.5 选中购物项 & 改变购物项的数量 & 删除购物项

#### 12.5.1 需求分析：

##### 1、选中购物项

1. 在页面中选中购物项后，数据应该要和redis中的数据进行同步
2. 页面选中，reids中数据就要更改

##### 2、改变购物项数量

1. 购物车中，增加了商品的数量，对应的价格，总价也需要进行改变

##### 3、删除购物项

1. 在购物项中删除该条数据，从redis中根据skuid删除这条记录

##### 4、将数据修改或删除后，重新跳转到cart.html 重新加载数据

#### 12.5.2 代码实现

前端页面方法

```java
// 选中购物项
    $(".itemCheck").click(function () {
        var skuId = $(this).attr("skuId");
        var check = $(this).prop("checked");
        location.href = "http://cart.gulimall.com/checkItem?skuId=" + skuId + "&check=" + (check?1:0);
    })

    // 改变购物项数量
    $(".countOpsBtn").click(function() {
        var skuId = $(this).parent().attr("skuId");
        var num = $(this).parent().find(".countOpsNum").text();
        location.href = "http://cart.gulimall.com/countItem?skuId=" + skuId + "&num=" + num;
    })
    var deleteId = 0;
    // 删除购物项
    function deleteItem() {
        location.href = "http://cart.gulimall.com/deleteItem?skuId=" + deleteId;
    }
    $(".deleteItemBtn").click(function() {
        deleteId = $(this).attr("skuId");
    })
```

Controller

```java
 /**
    * @需求描述: 系统管理员 购物车组 模块 用户选中购物项更新redis中对象
    * @创建人: 
    * @创建时间: 2021/01/04 15:37
    * @修改需求:
    * @修改人:
    * @修改时间:
    * @需求思路:
    */
    @GetMapping("/checkItem")
    public String checkItem(@RequestParam("skuId") Long skuId,
                            @RequestParam("checked") Integer checked) {
        cartService.checkItem(skuId,checked);
        // 重定向到购物车页面，重新获取购物车数据
        return "redirect:http://cart.gulimall.com/cart.html";
    }

    /**
    * @需求描述: 系统管理员 购物车组 模块 用户更改购物车中购物项的数量
    * @创建人: 
    * @创建时间: 2021/01/04 16:05
    * @修改需求:
    * @修改人:
    * @修改时间:
    * @需求思路:
    */
    @GetMapping("/updateItem")
    public String updateItem(@RequestParam("skuId") Long skuId,
                             @RequestParam("count") Integer count) {
        cartService.updateItem(skuId,count);
        return "redirect:http://cart.gulimall.com/cart.html";
    }

    /**
    * @需求描述: 系统管理员 购物车服务 模块 用户删除购物车中的购物项
    * @创建人: 
    * @创建时间: 2021/01/04 16:40
    * @修改需求:
    * @修改人:
    * @修改时间:
    * @需求思路:
    */
    @GetMapping("/deleteItem")
    public String deleteItem(@RequestParam("skuId") Long skuId) {
        cartService.deleteItem(skuId);
        return "redirect:http://cart.gulimall.com/cart.html";
    }
```

Service

```java
@Override
    public void checkItem(Long skuId, Integer checked) {
        // 1、从redis中查出购物项设置购物项是否选中。
        BoundHashOperations<String, Object, Object> cartOps = getCartOps();
        String cart = (String) cartOps.get(skuId.toString());
        CartItem cartItem = JSON.parseObject(cart, CartItem.class);
        cartItem.setCheck(checked == 1 ? true : false);
        String jsonStirng = JSON.toJSONString(cartItem);
        cartOps.put(skuId.toString(),jsonStirng); // 更新redis
    }

    @Override
    public void updateItem(Long skuId, Integer count) {
        // 1、用户在页面对某个购物项增加或减少购物项的数量，那么redis中应该也要进行更新
        CartItem cartItem = getCartItem(skuId);
        cartItem.setCount(count);
        BoundHashOperations<String, Object, Object> cartOps = getCartOps();
        cartOps.put(skuId.toString(),JSON.toJSONString(cartItem)); // 更新redis中数据
    }

    @Override
    public void deleteItem(Long skuId) {
        // 获取到当前购物车，然后根据id删除购物车里面的购物项
        BoundHashOperations<String, Object, Object> cartOps = getCartOps();
        cartOps.delete(skuId.toString());
    }	
```



# 13、消息队列 - MQ

### 13.1 RabbitMQ

#### 异步处理

消息发送的时间取决于业务执行的最长的时间

![img](/1621739001571-9f36f928-3632-4068-8235-f1768402bf81.png)

![img](/1621739005626-099bd417-bc62-4293-9ebf-13644d05ee83.png)

![img](/1621739008547-a718ca00-e6ff-4a82-b31f-3003cceb7ce0.png)



#### 应用解耦

![img](/1621739027263-dc11f8be-49d1-4c72-8c09-b429f85d3111.png)

原本是需要**订单系统**直接调用**库存系统**

只需要将请求发送给消息队列，其他的就不需要去处理了，节省了处理业务逻辑的时间

#### 流量消峰

某一时刻如果请求特别的大，那就先把它放入消息队列，从而达到流量消峰的作用

流程图地址：https://www.processon.com/view/link/5fbda8c35653bb1d54f7077b

### 13.2 概述

1. 大多应用中，可通过消息服务中间件来提升系统异步通信，扩展解耦能力
2. 消息服务中两个重要概念：
   1. **消息代理（message broker）** 和 **目的地（destination）**
3. 当消息发送者发送消息后，将由消息代理接管，消息代理保证消息传递到指定目的地
4. 消息队列主要有两种形式的目的地
   1. **队列（Queue）**:点对点消息通信（point - to - point）
   2. **主题（topic）**：发布（publish）/订阅（subscribe）消息通信
5. 点对点式：
   1. 消息发送者发送消息，消息代理将其放入一个队列中，消息接收者从队列中获取消息内容，消息读取后被移出队列
   2. 消息只有唯一的发送者和接受者，但并不是说只能有一个接收者
6. 发布订阅式:
   1. 发送者（发布者）发到消息到主题，多个接收者（订阅者）监听（订阅）这个主题，那么就会在消息到达时同时收到消息
7. **JMS（Java Message Service） Java消息服务：**
   1. 基于JVM消息代理的规范，ActiveMQ、HornetMQ是JMS的实现
8. AMQP（Advanced Message Queuing Protocol）
   1. 高级消息队列协议，也是一个消息代理的规范，兼容JMS
   2. RabbitMQ是AMQP的实现
9. Spring 支持
   1. spring - jms提供了对JMS的支持
   2. spring - rabbit提供了对AMQP的支持
   3. 需要ConnectionFactory的实现来连接消息代理
   4. 提供 **JmsTemplate**、**RabbitTemplate** 来发送消息
   5. @JmsListener（JMS）、@RabbitListener（AMQP）注解在方法上监听消息代理发布的消息
   6. @EnableJms、@EnableRabbit开启支持
10. Spring Boot 自动配置
    1. JmsAutoConfiguration
    2. RabbitAutoConfiguration
11. 市面上的MQ产品
    1. ActiveMQ、RabbitMQ、RocketMQ，kafka

![image-20201116091205853](image-20201116091205853.png)



### 13.3 RabbitMQ概念

RabbitMQ简介：

RabbitMQ是一由erlang开发的AMQP（Advanved Message Queue Protocol）的开源实现

核心概念

**Message**

消息，消息是不具名的，它是由消息头和消息体组成，消息体是不透明的，而消息头则由一系列的可选属性组成，这些属性包括 **routing - key （路由键）**，**priority（相对于其他消息的优先权）**，**delivery - mode（指出该消息可能需要持久性存储）**等

**Publisher**

消息的生产者，也是一个像交换器发布消息的客户端应用程序

**Exchange**

交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列

Exchange有4种类型：**direct(默认)**、**fanout**、**topic**，和**heades**，不同类型的 **Exchange** 转发消息的策略有所区别

**Queue**

消息队列，用来保存消息直到发送给**消费者**，**他是消息的容器**，也是消息的重点，一个消息可以投入一个或多个队列，消息一直在队列里面，等待消费者连接到这个队列将其取走

**Binding**

绑定，**用于消息队列和交换器之间的关联**，一个绑定就是基于路由键将交换器和消息队列连接起来的规则，所有可以将交换器理解成一个由绑定构成的路由表

**Connection**

网路连接，比如一个TCP连接

**Channel**

信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的TCP连接内的虚拟连接，AMQP命令都是通过信道 发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁TCP都 是非常昂贵的开销，所以引入了信道的概念，以复用一条TCP连接。

**Consumer**

消息的消费者，表示一个消息队列中取得消息的客户端应用程序

**Virtual Host**

虚拟主机，表示交换器、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个Virtual host本质上就是一个 mini 版的RabbitMQ 服务器,拥有自己的队列、交换器、绑定和权限机制。Virtual host 是 AMQP 概念的基础，必须在连接时指定,RabbitMQ默认的vhost是/。

**Broker**

表示消息队列服务器实体

![image-20201116100431093](image-20201116100431093.png)

### 13.4 Docker 安装RabbitMQ

```shell
docker run -d --name rabbitmq -p 5671:5671 -p 5672:5672 -p 4369:4369 -p 25672:25672 -p 15671:15671 -p 15672:15672 rabbitmq:management

4369, 25672 (Erlang发现&集群端口)
5672, 5671 (AMQP端口)
15672 (web管理后台端口)
61613, 61614 (STOMP协议端口)
1883, 8883 (MQTT协议端口)
 # 自动启动
docker update rabbitmq --restart=always

```

![image-20201116102734767](image-20201116102734767.png)

![image-20201116103446291](image-20201116103446291.png)

### 13.5 RabbitMQ 运行机制

AMQP 中的消息路由

AMQP 中消息的路由过程和 Java 开发者熟悉的 JMS 存在一些差别，AMQP中增加了 **Exchange** 和 **Binding** 的角色 生产者把消息发布到 Exchange 上，消息最终到达队列并被消费者接收，而 Binding 决定交换器的消息应该发送给哪个队列

![image-20201116104235856](image-20201116104235856.png)

**Exchange 类型**

Exchange 分发消息时根据类型的不同分发策略有区别，目前共四种类型：direct、tanout、topic、headers header匹配AMQP消息的 header 而不是路由键，headers 交换器和 direct 交换器完全一致，但性能差能多，目前几乎用不到了，所以直接看另外三种类型

![image-20201116104546717](image-20201116104546717.png)

![image-20201116104918897](image-20201116104918897.png)

> 生产者是指挥部，交换机好比通信兵，队列是等候命令的士兵。binding好比是入伍登记信息，记录好每个士兵的兵种，接受什么样的任务。

### 13.6 RabbitMQ 整合

#### 1、引入 Spring-boot-starter-amqp

```xml
      <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>

```

#### 2、application.yml配置

```yaml
spring:
  rabbitmq:
    host: 192.168.56.10
    port: 5672
    virtual-host: /
```

#### 3、测试RabbitMQ

##### 1、AmqpAdmin:管理组件

```java
 /**
     * 创建Exchange
     * 1、如何利用Exchange,Queue,Binding
     *      1、使用AmqpAdmin进行创建
     * 2、如何收发信息
     */
    @Test
    public void contextLoads() {
        //	public DirectExchange(
        //	String name, 交换机的名字
        //	boolean durable, 是否持久
        //	boolean autoDelete, 是否自动删除
        //	Map<String, Object> arguments)
        //	{
        DirectExchange directExchange = new DirectExchange("hello-java.exchange",true,false);
        amqpAdmin.declareExchange(directExchange);
        log.info("Exchange[{}]创建成功：","hello-java.exchange");
    }

    /**
     * 创建队列
     */
    @Test
    public void createQueue() {
        // public Queue(String name, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments) {
        Queue queue = new Queue("hello-java-queue",true,false,false);
        amqpAdmin.declareQueue(queue);
        log.info("Queue[{}]:","创建成功");
    }


/**
     * 绑定队列
     */
    @Test
    public void createBinding() {
        // public Binding(String destination, 目的地
        // DestinationType destinationType, 目的地类型
        // String exchange,交换机
        // String routingKey,//路由键
        Binding binding = new Binding("hello-java-queue",
                Binding.DestinationType.QUEUE,
                "hello-java.exchange",
                "hello.java",null);
        amqpAdmin.declareBinding(binding);
        log.info("Binding[{}]创建成功","hello-java-binding");
    }
```



##### 2、RabbitTemplate：消息发送处理组件

```java
 @Autowired
    @Test
    public void sendMessageTest() {
        for(int i = 1; i <=5; i++) {
            if(i%2==0) {
                OrderReturnReasonEntity reasonEntity = new OrderReturnReasonEntity();
                reasonEntity.setId(1l);
                reasonEntity.setCreateTime(new java.util.Date());
                reasonEntity.setName("哈哈");
                //
                String msg = "Hello World";
                // 发送的对象类型的消息，可以是一个json
                rabbitTemplate.convertAndSend("hello-java.exchange","hello.java",reasonEntity);
            } else {
                OrderEntity orderEntity = new OrderEntity();
                orderEntity.setOrderSn(UUID.randomUUID().toString());
                rabbitTemplate.convertAndSend("hello-java.exchange","hello.java",orderEntity);
            }
            log.info("消息发送完成{}");
        }

    }
```



### 13.7 RabbitMQ消息确认机制 - 可靠到达

- 保证消息不丢失，可靠抵达，可以使用事务消息，性能下降250倍，为此引入确认机制
- **publisher** confirmCallback 确认模式
- **publisher** returnCallback 未投递到 queue 退回
- **consumer** ack 机制

![image-20201116163631107](image-20201116163631107.png)

#### 可靠抵达 - ConfirmCallback

spring.rabbitmq.publisher-confirms=true

在创建 `connectionFactory` 的时候设置 PublisherConfirms(true) 选项，开启 `confirmcallback`。

`CorrelationData` 用来表示当前消息唯一性

消息只要被 broker 接收到就会执行 confirmCallback,如果 cluster 模式，需要所有 broker 接收到才会调用 confirmCallback

被 broker 接收到只能表示 message 已经到达服务器，并不能保证消息一定会被投递到目标 queue 里，所以需要用到接下来的 returnCallback

#### 可靠抵达 - ReturnCallback

spring.rabbitmq.publisher-retuns=true

spring.rabbitmq.template.mandatory=true

confirm 模式只能保证消息到达 broker，不能保证消息准确投递到目标 queue 里。在有些模式业务场景下，我们需要保证消息一定要投递到目标 queue 里，此时就需要用到 return 退回模式

这样如果未能投递到目标 queue 里将调用 returnCallback，可以记录下详细到投递数据，定期的巡检或者自动纠错都需要这些数据

#### 可靠抵达 - Ack 消息确认机制

- 消费者获取到消息，成功处理，可以回复Ack给Broker
  - basic.ack 用于肯定确认：broker 将移除此消息
  - basic.nack 用于否定确认：可以指定 beoker 是否丢弃此消息，可以批量
  - basic.reject用于否定确认，同上，但不能批量
- 默认，消息被消费者收到，就会从broker的queue中移除
- 消费者收到消息，默认自动ack，但是如果无法确定此消息是否被处理完成，或者成功处理，我们可以开启手动ack模式
  - 消息处理成功，ack()，接受下一条消息，此消息broker就会移除
  - 消息处理失败，nack()/reject() 重新发送给其他人进行处理，或者容错处理后ack
  - 消息一直没有调用ack/nack方法，brocker认为此消息正在被处理，不会投递给别人，此时客户端断开，消息不会被broker移除，会投递给别人

### 13.8 RabbitMQ 延时队列(实现定时任务)

**场景:**

比如未付款的订单，超过一定时间后，系统自动取消订单并释放占有物品

**常用解决方案：**

Spring的schedule 定时任务轮询数据库

**缺点：**

消耗系统内存，增加了数据库的压力，存在较大的时间误差

**解决：**rabbitmq的消息TTL和死信Exchange结合



#### 使用场景

![image-20201120120737525](image-20201120120737525.png)

时效问题

上一轮扫描刚好扫描，而这个时候刚好下了订单，就没有扫描到，下一轮扫描的时候，订单还没有过期，等到订单过期后30分钟才被扫描到

![image-20201120120914320](image-20201120120914320.png)

#### 消息的TTL（Time To Live）

- 消息的TTL 就是**消息的存活时间**，
- RabbitMQ可以对**队列**还有**消息**分别设置TTL
  - 对队列设置就是没有消费者连着的保持时间，**也可以对每一个消息单独的设置，超过了这个时间我们可以认为这个消息他死了，称之为死信**
  - 如果队列设置了，消息也设置了，那么会**取小**，所以一个消息如果被路由到不同的队列中，这个消息死亡时间有可能不一样的（不同队列设置），这里讲的是单个TTL 因为他是实现延时任务的关键，可以通过**设置消息的 expiration 字段或者 x-message-ttl** 来设置时间两者是一样的效果

#### Dead Letter Exchange（DLX）

- 一个消息在满足如下条件下，会进**死信路由**，记住这里是路由不是队列，一个路由可以对应很多队列，（什么是死信）
  - 一个消息被Consumer拒收了，并且reject方法的参数里requeue是false。也就是说不被再次放在队列里，被其他消费者使用。(basic.reject/ basic.nack) 
  - requeue= false上面的消息的TTL到了，消息过期了。
  - 队列的长度限制满了。排在前面的消息会被丢弃或者扔到死信路由上
- Dead Letter Exchange其实就是一种普通的exchange, 和创建其他exchange没有两样。只是在某一个设置 Dead Letter Exchange的队列中有消息过期了自动触发消息的转发，发送到Dead Letter Exchange中去。
- 我们既可以控制消息在一段时间后变成死信， 又可以控制变成死信的消息被路由到某一个指定的交换机， 结合C者，其实就可以实现一个延时队列

> 死信路由，好比任务失败的复盘机制，好比部队里的军法处。

#### 延时队列实现 - 1

![image-20201120132805292](image-20201120132805292.png)

延时队列实现 - 2

![image-20201120132922164](image-20201120132922164.png)

代码实现：

下单场景

![image-20201120133054368](image-20201120133054368.png)

模式升级

![image-20201120133258725](image-20201120133258725.png)

代码实现：

SpringBoot可以使用@Bean 来初始化Queue、exchange、Biding等

```java
/**
 * 监听队列信息
 * @param orderEntity
 */
@RabbitListener(queues = "order.release.order.queue")
public void listener(OrderEntity orderEntity, Channel channel, Message message) throws IOException {
    System.out.println("收到过期的订单信息，准备关闭订单" + orderEntity.getOrderSn());
    // 确认接收到消息，不批量接收
    channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
}

/**
 * 容器中的 Binding、Queue、exchange 都会自动创建，(RabbitMQ没有的情况下)
 * @return
 */
@Bean
public Queue orderDelayQueue(){
    // 特殊参数
    Map<String,Object> map = new HashMap<>();
    // 设置交换器

    map.put("x-dead-letter-exchange", "order-event-exchange");
    // 路由键
    map.put("x-dead-letter-routing-key","order.release.order");
    // 消息过期时间
    map.put("x-message-ttl",60000);
    Queue queue = new Queue("order.delay.queue", true, false, false,map);
    return queue;
}

/**
 * 创建队列
 * @return
 */
@Bean
public Queue orderReleaseOrderQueue() {
    Queue queue = new Queue("order.release.order.queue", true, false, false);
    return queue;
}

/**
 * 创建交换机
 * @return
 */
@Bean
public Exchange orderEventExchange() {
    return new TopicExchange("order-event-exchange",true,false);
}

/**
 * 绑定关系 将delay.queue和event-exchange进行绑定
 * @return
 */
@Bean
public Binding orderCreateOrderBingding(){
        return new Binding("order.delay.queue",
                Binding.DestinationType.QUEUE,
                "order-event-exchange",
                "order.create.order",
                null);
}

/**
 * 将 release.queue 和 event-exchange进行绑定
 * @return
 */
@Bean
public Binding orderReleaseOrderBinding(){
    return new Binding("order.release.order.queue",
            Binding.DestinationType.QUEUE,
            "order-event-exchange",
            "order.release.order",
            null);
}
```



### 13.9 如何保证消息可靠性 - 消息丢失 & 消息重复

#### 1、消息丢失

- **消息发送出去，由于网络问题没有抵达服务器**
  - 做好容错方法(try-Catch) ，发送消息可能会网络失败，失败后要有重试机制，可记录到数据库，采用定期扫描重发的方式
  - 做好日志记录，每个消息状态是否都被服务器收到都应该记录
  - 做好定期重发，如果消息没有发送成功，定期去数据库扫描未成功的消息进行重发
- **消息抵达Broker, Broker要将消息写入磁盘(持久化)才算成功。此时Broker尚未持久化完成，宕机。**
  - publisher也必须加入确认回调机制，确认成功的消息，修改数据库消息状态。
- **自动ACK的状态下。消费者收到消息，但没来得及消息然后宕机**
  - 一定开启手动ACK，消费成功才移除，失败或者没来得及处理就noAck并重新入队

#### 2、消息重复

- **消息消费成功，事务已经提交，ack时，机器宕机。导致没有ack成功，Broker的消息重新由unack变为ready,并发送给其他消费者**
- **消息消费失败，由于重试机制，自动又将消息发送出去**
- **成功消费，ack时宕机，消息由unack变为ready, Broker又重新发送**
  - 消费者的业务消费接口应该设计为**幂等性**的。比如扣库存有工作单的状态标志
  - 使用**防重表**(redis/mysq|) ，发送消息每一 个都有业务的唯一 标识，处理过就不用处理
  - rabbitMQ的每一个消息都有redelivered字段， 可以获取**是否是被重新投递过来的**，而不是第一次投递过来的

#### 3、消息积压

- **消费者积压**
- **消费者消费能力不足积压**
- **发送者发送流量太大**
  - 上线更多消费者，进行正常消费
  - 上线专门的队列消费服务，将消息先批量取出来，记录数据库，离线慢慢处理





# 14、订单

### 14.1 订单中心

1、订单中心

电商系列涉及到 3 流，分别为信息流、资金流、物流，而订单系统作为中枢将三者有机的集合起来

订单模块是电商系统的枢纽，在订单这个模块上获取多个模块的数据和信息，同时对这些信息进行加工处理后流向下个环节，这一系列就构成了订单的信息疏通

#### 1、订单构成

![image-20201117102129127](image-20201117102129127.png)

##### 1、用户信息

用户信息包括是用户账号、用户等级、用户的收货地址、收货人、收货人电话、用户账号需要绑定手机号码，但是用户绑定的手机号码不一定是收货信息上的电话。用户可以添加多个收货信息，用户等级信息可以用来和促销系统进行匹配，获取商品折扣，同时用户等级还可以获取积分的奖励等

##### 2、订单基础信息

订单基础信息是订单流转的核心，其包括订单类型，父/子订单、订单编号、订单状态、订单流转时间等

1. 订单类型包括实体订单和虚拟订单商品等，这个根据商城商品和服务类型进行区分
2. 同时订单都需要做父子订单处理，之前在初创公司一直只有一个订单，没有做父子订单处理后期需
3. 要进行拆单的时候就比较麻烦，尤其是多商户商场，和不同仓库商品的时候，父子订单就是为后期做拆单准备的。
4. 订单编号不多说了，需要强调的一点是父子订单都需要有订单编号，需要完善的时候可以对订单编号的每个字段进行统一定义和诠释。
5. 订单状态记录订单每次流转过程，后面会对订单状态进行单独的说明。
6. 订单流转时间需要记录下单时间，支付时间，发货时间，结束时间/关闭时间等等



##### 3、商品信息

商品信息从商品库中获取商品的SKU信息、图片、名称、属性规格、商品单价、商户信息等，从用户

下单行为记录的用户下单数量，商品合计价格等



##### 4、优惠信息

优惠信息记录用户参与过的优惠活动，包括优惠促销活动，比如满减、满赠、秒杀等，用户使用的优

惠卷信息，优惠卷满足条件的优惠卷需要展示出来，另外虚拟币抵扣信息等进行记录

为什么把优惠信息单独拿出来而不放在支付信息里面呢?

因为优惠信息只是记录用户使用的条目，而支付信息需要加入数据进行计算，所以做为区分。



##### 5、支付信息

支付流水单号，这个流水单号是在唤起网关支付后支付通道返回给电商业务平台的支付流水号，财务

通过订单号和流水单号与支付通道进行对账使用。

支付方式用户使用的支付方式，比如微信支付、支付宝支付、钱包支付、快捷支付等。支付方式有时候可能有两个一-余额支付+第三方支付。

商品总金额，每个商品加总后的金额:运费，物流产生的费用;优惠总金额，包括促销活动的优惠金额，

优惠券优惠金额，虚拟积分或者虛拟币抵扣的金額，会员折扣的金额等之和;实付金额，用户实际需要

付款的金额。

**用户实付金额=商品总金额+运费 - 优惠总金额**



##### 6、物流信息

物流信息包括配送方式，物流公司，物流单号，物流状态，物流状态可以通过第三方接口来
获取和向用户展示物流每个状态节点。

#### 2、订单状态

##### 1、待付款

用户提交订单后，订单进行预下单，目前主流电商网站都会唤起支付，便于用户快速完成支
付，需要注意的是待付款状态下可以对库存进行锁定，锁定库存需要配置支付超时时间，超
时后将自动取消订单，订单变更关闭状态。

##### 2、已付款/代发货

用户完成订单支付，订单系统需要记录支付时间，支付流水单号便于对账，订单下放到WMS系统，仓库进行调动、配货、分拣，出库等操作

##### 3、待收货/已发货

仓库将商品出库后，订单进入物流环节，订单系统需要同步物流信息，便于用户实时熟悉商品的物流状态

##### 4、已完成

用户确认收货后吗，订单交易完成，后续支付则进行计算，如果订单存在问题进入售后状态

##### 5、已取消

付款之前取消订单，包括超时未付款或用户商户取消订单都会产生这种订单状态

##### 6、售后中

用户在付款后申请退款，或商家发货后用户申请退货



售后也同样存在各种状态，当发起售后申请后生成售后订单，售后订单状态为待审核，等待

商家审核，商家审核通过后订单状态变更为待退货，等待用户将商品寄回，商家收货后订单

状态更新为待退款状态，退款到用户原账户后订单状态更新为售后成功。





### 14.2 订单流程

订单流程是指从订单产生到完成整个流转的过程，从而行程了-套标准流程规则。而不同的产品类型或业务类型在系统中的流程会千差万别，比如上面提到的线上实物订单和虚拟订单的流程，线上实物订单与020订单等，所以需要根据不同的类型进行构建订单流程。不管类型如何订单都包括正向流程和逆向流程，对应的场景就是购买商品和退换货流程，正向流程就是一一个正常的网购步骤:订单生成>支付订单->卖家发货一确认收货>交易成功。而每个步骤的背后，订单是如何在多系统之间交互流转的，

可概括如下图

![image-20201117104613032](image-20201117104613032.png)	

1、订单创建与支付

1. 订单创建前需要预览订单，选择收货信息等
2. 订单创建需要锁定库存，库存有才可创建，否则不能创建
3. 订单创建后超时未支付需要解锁库存
4. 支付成功后，需要进行拆单，根据商品打包方式，所在仓库，物流等进行拆单
5. 支付的每笔流水都需要记录，以待查账
6. 订单创建，支付成功等状态都需要给MQ发送消息，方便其他系统感知订阅

2、逆向流程

1. 修改订单，用户没有提交订单，可以对订单一些信息进行修改，比如配送信息，优惠信息，及其他一些订单可修改范围的内容，此时只需对数据进行变更即可。
2. 订单取消**，用户主动取消订单和用户超时未支付**，两种情况下订单都会取消订单，而超时情况是系统自动关闭订单，所以在订单支付的响应机制上面要做支付的

### 

### 2. 订单登录拦截

因为订单系统必然涉及到用户信息，因此进入订单系统的请求必须是已经登录的，所以我们需要通过拦截器对未登录订单请求进行拦截

```java
public class LoginInterceptor implements HandlerInterceptor {
    public static ThreadLocal<MemberResponseVo> loginUser = new ThreadLocal<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HttpSession session = request.getSession();
        MemberResponseVo memberResponseVo = (MemberResponseVo) session.getAttribute(AuthServerConstant.LOGIN_USER);
        if (memberResponseVo != null) {
            loginUser.set(memberResponseVo);
            return true;
        }else {
            session.setAttribute("msg","请先登录");
            response.sendRedirect("http://auth.gulimall.com/login.html");
            return false;
        }
    }
}

@Configuration
public class GulimallWebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/**");
    }
}
```

### 14.4 订单业务

#### 1、搭建环境

在订单服务下准备好页面



##### 1.1、整合SpringSession

**1、引入pom**

```xml
  <!--整合spring session 解决session问题-->
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
```

**2、配置文件添加**

```yaml
Spring:
	session:
    	store-type: redis
```

**3、启动类加注解**

```java
@EnableRedisHttpSession
```

**4、修改页面中登录**

```html
						<li style="border: 0;">
							<a th:if="${session.loginUser != null }" class="aa">[[${session.loginUser.nickname}]]</a>
							<a th:if="${session.loginUser == null }" href="http://auth.gulimall.com/login.html">你好请登录</a>
						</li>
						<li>
							<a th:if="${session.loginUser == null }" style="color: red;" href="http://auth.gulimall.com/reg.html" class="li_2">免费注册</a>
						</li>
```

##### 1.2、订单登录拦截

任何请求都需要先经过拦截器的验证，才能去执行目标方法，这里是用户是否登录，用户登录了则放行，否则跳转到登陆页面

```java
  /**
     * 目标方法执行之前
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestURI = request.getRequestURI();
        // 指定的请求不拦截
        boolean match1 = new AntPathMatcher().match("/order/order/status/**", requestURI);
        boolean match2 = new AntPathMatcher().match("/payed/notify", requestURI);
        if (match1 || match2) {
            return true;
        }
        MemberRespVo memberRespVo= (MemberRespVo) request.getSession().getAttribute(AuthServerConstant.LOGIN_USER);
        if (memberRespVo != null) { // 用户登陆了
            loginUser.set(memberRespVo); // 放到共享数据中
            return true;
        } else { // 用户没登录
            // 给前端显示提示信息
            request.getSession().setAttribute("msg","请先进行登录");
            // 重定向到登录页面
            response.sendRedirect("http://auth.gulimall.com/login.html");
            return false;
        }
    }
```

#### 2、订单确认页

##### 1）模型抽取

<img src="/Snipaste_2020-10-10_18-38-51.png" style="zoom:38%;" />

**根据图片中商品信息抽取成Vo**

##### 2.1、抽取Vo

**订单确认OrderConfirmVo**

```java

/**
 * 订单确认页需要的数据
 * @author gcq
 * @Create 2020-11-17
 */

public class OrderConfirmVo {

    // 收货地址，ums_member_receive_address
    @Getter @Setter
    List<MemberAddressVo> address;

    // 所有选中的购物项
    @Getter @Setter
    List<OrderItemVo> item;

    // 发票记录...

    /**
     * 优惠卷信息
     */
    @Getter @Setter
    Integer integration;

    /**
     * 订单总额
     */
    BigDecimal total;
    public BigDecimal getTotal() {
        BigDecimal sum = new BigDecimal("0");
        if(item != null) {
            for (OrderItemVo orderItemVo : item) {
                BigDecimal multiply = orderItemVo.getPrice().multiply(new BigDecimal(orderItemVo.getCount().toString()));
                sum = sum.add(multiply);
            }
        }
        return sum;
    }
    @Getter @Setter
    Map<Long,Boolean> stocks;
    /**
     * 应付价格
     */
    BigDecimal payPrice;

    public BigDecimal getPayPrice() {
        BigDecimal sum = new BigDecimal("0");
        if(item != null) {
            for (OrderItemVo orderItemVo : item) {
                BigDecimal multiply = orderItemVo.getPrice().multiply(new BigDecimal(orderItemVo.getCount().toString()));
                sum = sum.add(multiply);
            }
        }
        return sum;
    }

    @Setter
    private Integer count;

    /**
     * 遍历item 拿到商品的数量
     * @return
     */
    public Integer getCount() {
        Integer i = 0;
        if (item != null) {
            for (OrderItemVo orderItemVo : item) {
                i+=orderItemVo.getCount();
            }
        }
        return i;
    }

    @Getter @Setter
    private String orderToken;

}
```

**商品项orderItemVo**

```java
/**
 * 商品项
 * @author gcq
 * @Create 2020-11-17
 */
@Data
public class OrderItemVo {
    /**
     * 商品id
     */
    private Long skuId;
    /**
     * 购物车中是否选中
     */
    private Boolean check = true;
    /**
     * 商品的标题
     */
    private String title;
    /**
     * 商品的图片
     */
    private String image;
    /**
     * 商品的属性
     */
    private List<String> skuAttr;
    /**
     * 商品的价格
     */
    private BigDecimal price;
    /**
     * 商品的数量
     */
    private Integer count;
    /**
     * 购物车价格 使用自定义get、set
     */
    private BigDecimal totalPrice;


    private BigDecimal weight;

}
```

**用户地址MemberAddressVo**

```java
/**
 * 用户地址信息
 * @author gcq
 * @Create 2020-11-17
 */
@Data
public class MemberAddressVo {
    private Long id;
    /**
     * member_id
     */
    private Long memberId;
    /**
     * 收货人姓名
     */
    private String name;
    /**
     * 电话
     */
    private String phone;
    /**
     * 邮政编码
     */
    private String postCode;
    /**
     * 省份/直辖市
     */
    private String province;
    /**
     * 城市
     */
    private String city;
    /**
     * 区
     */
    private String region;
    /**
     * 详细地址(街道)
     */
    private String detailAddress;
    /**
     * 省市区代码
     */
    private String areacode;
    /**
     * 是否默认
     */
    private Integer defaultStatus;
}
```

#### （3）Feign远程调用丢失请求头问题

`feign`远程调用的请求头中没有含有`JSESSIONID`的`cookie`，所以也就不能得到服务端的`session`数据，cart认为没登录，获取不了用户信息

![](/Snipaste_2020-10-10_21-36-18.png)

<img src="/Snipaste_2020-10-10_21-47-54.png" style="zoom: 50%;" />

```java
Request targetRequest(RequestTemplate template) {
  for (RequestInterceptor interceptor : requestInterceptors) {
    interceptor.apply(template);
  }
  return target.apply(template);
}
```

但是在`feign`的调用过程中，会使用容器中的`RequestInterceptor`对`RequestTemplate`进行处理，因此我们可以通过向容器中导入定制的`RequestInterceptor`为请求加上`cookie`。

```java
public class GuliFeignConfig {
    @Bean
    public RequestInterceptor requestInterceptor() {
        return new RequestInterceptor() {
            @Override
            public void apply(RequestTemplate template) {
                //1. 使用RequestContextHolder拿到老请求的请求数据
                ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
                if (requestAttributes != null) {
                    HttpServletRequest request = requestAttributes.getRequest();
                    if (request != null) {
                        //2. 将老请求得到cookie信息放到feign请求上
                        String cookie = request.getHeader("Cookie");
                        template.header("Cookie", cookie);
                    }
                }
            }
        };
    }
}
```

* `RequestContextHolder`为SpingMVC中共享`request`数据的上下文，底层由`ThreadLocal`实现

经过`RequestInterceptor`处理后的请求如下，已经加上了请求头的`Cookie`信息

<img src="/Snipaste_2020-10-10_21-55-45.png" style="zoom: 50%;" />

#### （4）Feign异步情况丢失上下文问题

<img src="/Snipaste_2020-10-10_22-08-32.png" style="zoom:38%;" />

* 由于`RequestContextHolder`使用`ThreadLocal`共享数据，所以在开启异步时获取不到老请求的信息，自然也就无法共享`cookie`了

在这种情况下，我们需要在开启异步的时候将老请求的`RequestContextHolder`的数据设置进去

<img src="/Snipaste_2020-10-10_22-13-47.png" style="zoom: 67%;" />

##### 2.2、订单确认页数据查询

* 查询购物项、库存和收货地址都要调用远程服务，串行会浪费大量时间，因此我们使用`CompletableFuture`进行异步编排
* 可能由于延迟，订单提交按钮可能被点击多次，为了防止重复提交的问题，我们在返回订单确认页时，在`redis`中生成一个随机的令牌，过期时间为30min，提交的订单会携带这个令牌，我们将会在订单提交的处理页面核验此令牌

```java
   @Override
    public OrderConfirmVo confirmOrder() throws ExecutionException, InterruptedException {
        OrderConfirmVo confirmVo = new OrderConfirmVo();
        MemberRespVo memberRespVo = OrderInterceptor.loginUser.get();// 获取当前登录后的用户
        // 异步任务执行之前，先共享数据
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        // 1、第一个异步任务 远程查询用户地址信息
        CompletableFuture<Void> memberFuture = CompletableFuture.runAsync(() -> {
            // 在主线程中拿到原来的数据，在父线程里面共享RequestContextHolder
            // 只有共享，拦截其中才会有数据
            RequestContextHolder.setRequestAttributes(requestAttributes);
            // 根据会员id查出之前会员保存过的收货地址信息
            // 远程查询会员服务的收获地址信息
            List<MemberAddressVo> address = memberFeignService.getAddress(memberRespVo.getId());
            log.error("address:{}",address);
            confirmVo.setAddress(address);
        }, executor);

        // 2、第二个异步任务远程查询购物车中选中给的购物项
        CompletableFuture<Void> addressFuture = CompletableFuture.runAsync(() -> {
            // 每一个线程都来共享之前的请求数据
            RequestContextHolder.setRequestAttributes(requestAttributes);
            // 远程查询购物车中的购物项信息，用来结账
            List<OrderItemVo> currentUserCartItem = cartFeignServicea.getCurrentUserCartItem();// 获取当前用户的购物项数据
            log.error("currentUserCartItem:{}",currentUserCartItem);
            confirmVo.setItem(currentUserCartItem);
            // 查询到购物项信息后，再看查询购物的库存信息
        }, executor).thenRunAsync(() -> { // 只要上面任务执行完成，就开始执行thenRunAsync的任务
            // 3、商品是否有库存
            List<OrderItemVo> items = confirmVo.getItem();
            // 批量查询每一个商品的信息
            // 收集好商品id
            List<Long> collect = items.stream().map(item -> item.getSkuId()).collect(Collectors.toList());
            // 远程查询购物项对应的库存信息
            R data = wareFeignService.hasStock(collect);
            // 得到每一个商品的库存状态信息
            List<SkuHasStockVo> hasStockVo = data.getData(new TypeReference<List<SkuHasStockVo>>() {
            });
            if (hasStockVo != null) {
                Map<Long, Boolean> stockMap = hasStockVo.stream().collect(Collectors.toMap(SkuHasStockVo::getSkuId, SkuHasStockVo::getHasStock));
                confirmVo.setStocks(stockMap);
                log.error("stockMap:{}",stockMap);
            }
        },executor);


        // 4、查询积分信息
        Integer integration = confirmVo.getIntegration();
        confirmVo.setIntegration(integration);

        // 等两个异步任务都完成
        CompletableFuture.allOf(memberFuture, addressFuture).get();
       // 4、防重令牌
        /**
         * 接口幂等性就是用户对同一操作发起的一次请求和多次请求结果是一致的
         * 不会因为多次点击而产生了副作用，比如支付场景，用户购买了商品，支付扣款成功，
         * 但是返回结果的时候出现了网络异常，此时钱已经扣了，用户再次点击按钮，
         * 此时就会进行第二次扣款，返回结果成功，用户查询余额发现多扣钱了，
         * 流水记录也变成了两条。。。这就没有保证接口幂等性
         */
        // 先是再页面中生成一个随机码把他叫做token先存到redis中，然后放到对象中在页面进行渲染。
        // 用户提交表单的时候，带着这个token和redis里面去匹配如果一直那么可以执行下面流程。
        // 匹配成功后再redis中删除这个token，下次请求再过来的时候就匹配不上直接返回
        // 生成防重令牌
        String token = UUID.randomUUID().toString().replace("-","");
        // 存到redis中 设置30分钟超时
        redisTemplate.opsForValue().set(OrderConstant.USER_ORDER_TOKEN_PREFIX + memberRespVo.getId(),token,30, TimeUnit.SECONDS);
        // 放到页面进行显示token，然后订单中带着token来请求
        confirmVo.setOrderToken(token);

        return confirmVo;
```

#### 3、创建订单

##### 1、OrderWebController

```java
 /**
    * @需求描述: 系统管理员 订单组 模块 用户下单功能
    * @创建人: 郭承乾
    * @创建时间: 2021/01/06 12:01
    * @修改需求:
    * @修改人:
    * @修改时间:
    * @需求思路:
    */
    @PostMapping("/submitOrder")
    public String submitOrder(OrderSubmitVo vo, Model model, RedirectAttributes redirectAttributes) {
        SubmitOrderResponseVo responseVo = orderService.submitOrder(vo);
        log.error("======================订单创建成功{}:",responseVo);
        // 根据vo中定义的状态码来验证
        if (responseVo.getCode() == 0 ) { // 订单创建成功
            // 下单成功返回到支付页
            model.addAttribute("submitOrderResp",responseVo);
            return "pay";
        } else { // 下单失败
            // 根据状态码验证对应的状态
            String msg = "下单失败";
            switch (responseVo.getCode()) {
                case 1: msg += "订单信息过期，请刷新后再次提交"; break;
                case 2: msg += "订单商品价格发生变化，请确认后再次提交"; break;
                case 3: msg += "库存锁定失败，商品库存不足"; break;
            }
            redirectAttributes.addFlashAttribute("msg",msg);
            // 重新回到订单确认页面
            return "redirect:http://order.gulimall.com/toTrade";
        }
    }
```

##### 2、Service

具体业务

```java
 @Transactional
    @Override
    public SubmitOrderResponseVo submitOrder(OrderSubmitVo vo) {
        // 先将参数放到共享变量中，方便之后方法使用该参数
        confirmVoThreadLocal.set(vo);
        // 接收返回数据
        SubmitOrderResponseVo response = new SubmitOrderResponseVo();
        response.setCode(0);
        // 通过拦截器拿到用户的数据
        MemberRespVo memberRespVo = LoginInterceptor.loginUser.get();
        /**
         * 不使用原子性验证令牌
         *      1、用户带着两个订单，提交速度非常快，两个订单的令牌都是123，去redis里面查查到的也是123。
         *          两个对比都通过，然后来删除令牌，那么就会出现用户重复提交的问题，
         *      2、第一次差的快，第二次查的慢，只要没删就会出现这些问题
         *      3、因此令牌的【验证和删除必须保证原子性】
         *      String orderToken = vo.getOrderToken();
         *      String redisToken = redisTemplate.opsForValue().get(OrderConstant.USER_ORDER_TOKEN_PREFIX + memberRespVo.getId());
         *         if (orderToken != null && orderToken.equals(redisToken)) {
         *             // 令牌验证通过 进行删除
         *             redisTemplate.delete(OrderConstant.USER_ORDER_TOKEN_PREFIX + memberRespVo.getId());
         *         } else {
         *             // 不通过
         *         }
         */
        // 验证令牌【令牌的对比和删除必须保证原子性】
        // 因此使用redis中脚本来进行验证并删除令牌
        // 0【删除失败/验证失败】 1【删除成功】
        String script = "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
        /**
         * redis lur脚本命令解析
         * if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end
         *  1、redis调用get方法来获取一个key的值，如果这个get出来的值等于我们传过来的值
         *  2、然后就执行删除，根据这个key进行删除，删除成功返回1，验证失败返回0
         *  3、删除否则就是0
         *  总结：相同的进行删除，不相同的返回0
         * 脚本大致意思
         */
        // 拿到令牌
        String orderToken = vo.getOrderToken();
        /**
         * 	public <T> T execute(RedisScript<T> script // redis的脚本
         * 	    , List<K> keys // 对应的key 参数中使用了Array.asList 将参数转成list集合
         * 	    , Object... args) { // 要删除的值
         */
        // 原子验证和删除
        Long result = redisTemplate.execute(new DefaultRedisScript<Long>(script, Long.class)
                , Arrays.asList(OrderConstant.USER_ORDER_TOKEN_PREFIX + memberRespVo.getId())
                , orderToken);
        if (result  == 0L) { // 验证令牌验证失败
            // 验证失败直接返回结果
            response.setCode(1);
            return response;
        } else { // 原子验证令牌成功
            // 下单 创建订单、验证令牌、验证价格、验证库存
            // 1、创建订单、订单项信息
            OrderCreateTo order = createOrder();
            // 2、应付总额
            BigDecimal payAmount = order.getOrder().getPayAmount();
            // 应付价格
            BigDecimal payPrice = vo.getPayPrice();
            /**
             * 电商项目对付款的金额精确到小数点后面两位
             * 订单创建好的应付总额 和购物车中计算好的应付价格求出绝对值。
             */
            if(Math.abs(payAmount.subtract(payPrice).doubleValue()) < 0.01) {
                // 金额对比成功 保存订单
                saveOrder(order);
                // 创建锁定库存Vo
                WareSkuLockedVo wareSkuLockedVo = new WareSkuLockedVo();
                // 准备好商品项
                List<OrderItemVo> lock = order.getOrderItem().stream().map(orderItemEntity -> {
                    OrderItemVo orderItemVo = new OrderItemVo();
                    // 商品购买数量
                    orderItemVo.setCount(orderItemEntity.getSkuQuantity());
                    // skuid 用来查询商品信息
                    orderItemVo.setSkuId(orderItemEntity.getSkuId());
                    // 商品标题
                    orderItemVo.setTitle(orderItemEntity.getSkuName());
                    return orderItemVo;
                }).collect(Collectors.toList());
                // 订单号
                wareSkuLockedVo.setOrderSn(order.getOrder().getOrderSn());
                // 商品项
                wareSkuLockedVo.setLocks(lock);
                // 远程调用库存服务锁定库存
                R r = wareFeignService.orderLockStock(wareSkuLockedVo);
                if (r.getCode() == 0) { // 库存锁定成功
                    // 将订单对象放到返回Vo中
                    response.setOrder(order.getOrder());
                    // 设置状态码
                    response.setCode(0);
                    // 订单创建成功发送消息给MQ
                    rabbitTemplate.convertAndSend("order-event-exchange"
                            ,"order.create.order"
                            ,order.getOrder());
                    return response;
                } else {
                    // 远程锁定库存失败
                    response.setCode(3);
                    return response;
                }
            } else {
                // 商品价格比较失败
                response.setCode(2);
                return response;
            }
        }
    }
 /**
     * 创建订单和订单项
     * @return
     */
    private OrderCreateTo createOrder() {
        OrderCreateTo orderCreateTo = new OrderCreateTo();
        // 1、生成订单号
        String orderSn = IdWorker.getTimeId();
        // 2、构建订单
        OrderEntity orderEntity = buildOrder(orderSn);
        // 3、构建订单项
        List<OrderItemEntity> itemEntities = builderOrderItems(orderSn);
        // 4、设置价格、积分相关信息
        computPrice(orderEntity,itemEntities);
        // 5、设置订单项
        orderCreateTo.setOrderItem(itemEntities);
        // 6、设置订单
        orderCreateTo.setOrder(orderEntity);
        return orderCreateTo;
    }
 /**
     * 构建订单
     * @param orderSn
     * @return
     */
    private OrderEntity buildOrder(String orderSn) {
        // 拿到共享数据
        OrderSubmitVo orderSubmitVo = confirmVoThreadLocal.get();
        // 用户登录登录数据
        MemberRespVo memberRespVo = LoginInterceptor.loginUser.get();

        OrderEntity orderEntity = new OrderEntity();
        // 设置订单号
        orderEntity.setOrderSn(orderSn);
        // 用户id
        orderEntity.setMemberId(memberRespVo.getId());
        // 根据用户收货地址id查询出用户的收获地址信息
        R fare = wareFeignService.getFare(orderSubmitVo.getAddrId());
        FareVo data = fare.getData(new TypeReference<FareVo>() {
        });
        //将查询到的会员收货地址信息设置到订单对象中
        // 运费金额
        orderEntity.setFreightAmount(data.getFare());
        // 城市
        orderEntity.setReceiverCity(data.getMemberAddressVo().getCity());
        // 详细地区
        orderEntity.setReceiverDetailAddress(data.getMemberAddressVo().getDetailAddress());
        // 收货人姓名
        orderEntity.setReceiverName(data.getMemberAddressVo().getName());
        // 收货人手机号
        orderEntity.setReceiverPhone(data.getMemberAddressVo().getPhone());
        // 区
        orderEntity.setReceiverRegion(data.getMemberAddressVo().getRegion());
        // 省份直辖市
        orderEntity.setReceiverProvince(data.getMemberAddressVo().getProvince());
        // 订单刚创建状态设置为 待付款，用户支付成功后将该该状态改成已付款
        orderEntity.setStatus(OrderStatusEnum.CREATE_NEW.getCode());
        // 自动确认时间
        orderEntity.setAutoConfirmDay(7);

        return orderEntity;
    }
/**
     * 构建订单项
     * @param orderSn
     * @return
     */
    private List<OrderItemEntity> builderOrderItems(String orderSn) {
        // 获取购物车中选中的商品
        List<OrderItemVo> currentUserCartItem = cartFeignServicea.getCurrentUserCartItem();
        if (currentUserCartItem != null && currentUserCartItem.size() > 0) {
            List<OrderItemEntity> collect = currentUserCartItem.stream().map(orderItemVo -> {
                // 构建订单项
                OrderItemEntity itemEntity = builderOrderItem(orderItemVo);
                itemEntity.setOrderSn(orderSn);
                return itemEntity;
            }).collect(Collectors.toList());
            return collect;
        }
        return null;
    }
  /**
     * 构建订单项信息
     * @param cartItem
     * @return
     */
    private OrderItemEntity builderOrderItem(OrderItemVo cartItem) {
        OrderItemEntity itemEntity = new OrderItemEntity();
        // 1、根据skuid查询关联的spuinfo信息
        Long skuId = cartItem.getSkuId();
        R spuinfo = productFeignService.getSpuInfoBySkuId(skuId);
        SpuInfoVo spuInfoVo = spuinfo.getData(new TypeReference<SpuInfoVo>() {
        });
        // 2、设置商品项spu信息
        // 品牌信息
        itemEntity.setSpuBrand(spuInfoVo.getBrandId().toString());
        // 商品分类信息
        itemEntity.setCategoryId(spuInfoVo.getCatalogId());
        // spuid
        itemEntity.setSpuId(spuInfoVo.getId());
        // spu_name 商品名字
        itemEntity.setSpuName(spuInfoVo.getSpuName());

        // 3、设置商品sku信息
        // skuid
        itemEntity.setSkuId(skuId);
        // 商品标题
        itemEntity.setSkuName(cartItem.getTitle());
        // 商品图片
        itemEntity.setSkuPic(cartItem.getImage());
        // 商品sku价格
        itemEntity.setSkuPrice(cartItem.getPrice());
        // 商品属性以 ; 拆分
        String skuAttr = StringUtils.collectionToDelimitedString(cartItem.getSkuAttr(), ";");
        itemEntity.setSkuAttrsVals(skuAttr);
        // 商品购买数量
        itemEntity.setSkuQuantity(cartItem.getCount());

        // 4、设置商品优惠信息【不做】
        // 5、设置商品积分信息
        // 赠送积分 移弃小数值
        itemEntity.setGiftIntegration(cartItem.getPrice().multiply(new BigDecimal(cartItem.getCount().toString())).intValue());
        // 赠送成长值
        itemEntity.setGiftGrowth(cartItem.getPrice().multiply(new BigDecimal(cartItem.getCount().toString())).intValue());

        // 6、订单项的价格信息
        // 这里需要计算商品的分解信息
        // 商品促销分解金额
        itemEntity.setPromotionAmount(new BigDecimal("0"));
        // 优惠券优惠分解金额
        itemEntity.setCouponAmount(new BigDecimal("0"));
        // 积分优惠分解金额
        itemEntity.setIntegrationAmount(new BigDecimal("0"));
        // 商品价格乘以商品购买数量=总金额(未包含优惠信息)
        BigDecimal origin = itemEntity.getSkuPrice().multiply(new BigDecimal(itemEntity.getSkuQuantity().toString()));
        // 总价格减去优惠卷-积分优惠-商品促销金额 = 总金额
        origin.subtract(itemEntity.getPromotionAmount())
                .subtract(itemEntity.getCouponAmount())
                .subtract(itemEntity.getIntegrationAmount());
        // 该商品经过优惠后的分解金额
        itemEntity.setRealAmount(origin);
        return itemEntity;
    }
 /**
     * 计算订单涉及到的积分、优惠卷抵扣、促销优惠信息等信息
     * @param orderEntity
     * @param itemEntities
     * @return
     */
    private OrderEntity computPrice(OrderEntity orderEntity, List<OrderItemEntity> itemEntities) {
        // 1、定义好相关金额，然后遍历购物项进行计算
        // 总价格
        BigDecimal total = new BigDecimal("0");
        //相关优惠信息
        // 优惠卷抵扣金额
        BigDecimal coupon = new BigDecimal("0");
        // 积分优惠金额
        BigDecimal integration = new BigDecimal("0");
        // 促销优惠金额
        BigDecimal promotion = new BigDecimal("0");
        // 积分
        BigDecimal gift = new BigDecimal("0");
        // 成长值
        BigDecimal growth = new BigDecimal("0");

        // 遍历订单项将所有的优惠信息进行相加
        for (OrderItemEntity itemEntity : itemEntities) {
            coupon = coupon.add(itemEntity.getCouponAmount()); // 优惠卷抵扣
            integration = integration.add(itemEntity.getIntegrationAmount()); // 积分优惠分解金额
            promotion = promotion.add(itemEntity.getPromotionAmount()); // 商品促销分解金额
            gift = gift.add(new BigDecimal(itemEntity.getGiftIntegration().toString())); // 赠送积分
            growth = growth.add(new BigDecimal(itemEntity.getGiftGrowth())); // 赠送成长值
            total = total.add(itemEntity.getRealAmount()); //优惠后的总金额
        }

        // 2、设置订单金额
        // 订单总金额
        orderEntity.setTotalAmount(total);
        // 应付总额 = 订单总额 + 运费信息
        orderEntity.setPayAmount(total.add(orderEntity.getFreightAmount()));
        // 促销优化金额（促销价、满减、阶梯价）
        orderEntity.setPromotionAmount(promotion);
        // 优惠券抵扣金额
        orderEntity.setCouponAmount(coupon);

        // 3、设置积分信息
        // 订单购买后可以获得的成长值
        orderEntity.setGrowth(growth.intValue());
        // 积分抵扣金额
        orderEntity.setIntegrationAmount(integration);
        // 可以获得的积分
        orderEntity.setIntegration(gift.intValue());
        // 删除状态【0->未删除；1->已删除】
        orderEntity.setDeleteStatus(0);
        return orderEntity;
    }
```

##### 5) 锁定库存

```java
 List<OrderItemVo> orderItemVos = order.getOrderItems().stream().map((item) -> {
                    OrderItemVo orderItemVo = new OrderItemVo();
                    orderItemVo.setSkuId(item.getSkuId());
                    orderItemVo.setCount(item.getSkuQuantity());
                    return orderItemVo;
                }).collect(Collectors.toList());
                R r = wareFeignService.orderLockStock(orderItemVos);
                //5.1 锁定库存成功
                if (r.getCode()==0){
                    responseVo.setOrder(order.getOrder());
                    responseVo.setCode(0);
                    return responseVo;
                }else {
                    //5.2 锁定库存失败
                    String msg = (String) r.get("msg");
                    throw new NoStockException(msg);
                }
```

* 找出所有库存大于商品数的仓库
* 遍历所有满足条件的仓库，逐个尝试锁库存，若锁库存成功则退出遍历

```java
@RequestMapping("/lock/order")
public R orderLockStock(@RequestBody List<OrderItemVo> itemVos) {
    try {
        Boolean lock = wareSkuService.orderLockStock(itemVos);
        return R.ok();
    } catch (NoStockException e) {
        return R.error(BizCodeEnum.NO_STOCK_EXCEPTION.getCode(), BizCodeEnum.NO_STOCK_EXCEPTION.getMsg());
    }
}

@Transactional
@Override
public Boolean orderLockStock(List<OrderItemVo> itemVos) {
    List<SkuLockVo> lockVos = itemVos.stream().map((item) -> {
        SkuLockVo skuLockVo = new SkuLockVo();
        skuLockVo.setSkuId(item.getSkuId());
        skuLockVo.setNum(item.getCount());
        //找出所有库存大于商品数的仓库
        List<Long> wareIds = baseMapper.listWareIdsHasStock(item.getSkuId(), item.getCount());
        skuLockVo.setWareIds(wareIds);
        return skuLockVo;
    }).collect(Collectors.toList());

    for (SkuLockVo lockVo : lockVos) {
        boolean lock = true;
        Long skuId = lockVo.getSkuId();
        List<Long> wareIds = lockVo.getWareIds();
        //如果没有满足条件的仓库，抛出异常
        if (wareIds == null || wareIds.size() == 0) {
            throw new NoStockException(skuId);
        }else {
            for (Long wareId : wareIds) {
                Long count=baseMapper.lockWareSku(skuId, lockVo.getNum(), wareId);
                if (count==0){
                    lock=false;
                }else {
                    lock = true;
                    break;
                }
            }
        }
        if (!lock) throw new NoStockException(skuId);
    }
    return true;
}
```

这里通过异常机制控制事务回滚，如果在锁定库存失败则抛出`NoStockException`s,订单服务和库存服务都会回滚。

#### （3） 分布式事务

分布式情况下，可能出现一些服务事务不一致的情况

* 远程服务假失败
* 远程服务执行完成后，下面其他方法出现异常

<img src="/Snipaste_2020-10-11_09-15-30.png" style="zoom:38%;" />

#### （4）使用seata解决分布式事务问题

导入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

环境搭建

下载senta-server-0.7.1并修改`register.conf`,使用nacos作为注册中心

```shell
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    serverAddr = "#:8848"
    namespace = "public"
    cluster = "default"
  }
```

将`register.conf`和`file.conf`复制到需要开启分布式事务的根目录，并修改`file.conf`

 `vgroup_mapping.${application.name}-fescar-service-group = "default"`

```shell
service {
  #vgroup->rgroup
  vgroup_mapping.gulimall-ware-fescar-service-group = "default"
  #only support single node
  default.grouplist = "127.0.0.1:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
```

使用seata包装数据源

```java
@Configuration
public class MySeataConfig {
    @Autowired
    DataSourceProperties dataSourceProperties;

    @Bean
    public DataSource dataSource(DataSourceProperties dataSourceProperties) {

        HikariDataSource dataSource = dataSourceProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
        if (StringUtils.hasText(dataSourceProperties.getName())) {
            dataSource.setPoolName(dataSourceProperties.getName());
        }
        return new DataSourceProxy(dataSource);
    }
}
```

在大事务的入口标记注解`@GlobalTransactional`开启全局事务，并且每个小事务标记注解`@Transactional`

```java
@GlobalTransactional
@Transactional
@Override
public SubmitOrderResponseVo submitOrder(OrderSubmitVo submitVo) {
}
```

### 5. 使用消息队列实现最终一致性

#### (1) 延迟队列的定义与实现

* 定义：

  延迟队列存储的对象肯定是对应的延时消息，所谓"延时消息"是指当消息被发送以后，并不想让消费者立即拿到消息，而是等待指定时间后，消费者才拿到这个消息进行消费。

* 实现：

  rabbitmq可以通过设置队列的`TTL`和死信路由实现延迟队列

  * TTL：

  >RabbitMQ可以针对Queue设置x-expires 或者 针对Message设置 x-message-ttl，来控制消息的生存时间，如果超时(两者同时设置以最先到期的时间为准)，则消息变为dead letter(死信)

  

  * 死信路由DLX

  >RabbitMQ的Queue可以配置x-dead-letter-exchange 和x-dead-letter-routing-key（可选）两个参数，如果队列内出现了dead letter，则按照这两个参数重新路由转发到指定的队列。

  >- x-dead-letter-exchange：出现dead letter之后将dead letter重新发送到指定exchange
  >- x-dead-letter-routing-key：出现dead letter之后将dead letter重新按照指定的routing-key发送

<img src="/Snipaste_2020-10-11_17-00-18.png" style="zoom: 50%;" />

针对订单模块创建以上消息队列，创建订单时消息会被发送至队列`order.delay.queue`，经过`TTL`的时间后消息会变成死信以`order.release.order`的路由键经交换机转发至队列`order.release.order.queue`，再通过监听该队列的消息来实现过期订单的处理

#### (2) 延迟队列使用场景

<img src="/Snipaste_2020-10-14_15-42-12.png" style="zoom: 25%;" />

**为什么不能用定时任务完成？**

如果恰好在一次扫描后完成业务逻辑，那么就会等待两个扫描周期才能扫到过期的订单，不能保证时效性

<img src="/Snipaste_2020-10-14_15-43-37.png" style="zoom: 25%;" />



#### (3) 定时关单与库存解锁主体逻辑

* 订单超时未支付触发订单过期状态修改与库存解锁

> 创建订单时消息会被发送至队列`order.delay.queue`，经过`TTL`的时间后消息会变成死信以`order.release.order`的路由键经交换机转发至队列`order.release.order.queue`，再通过监听该队列的消息来实现过期订单的处理
>
> * 如果该订单已支付，则无需处理
> * 否则说明该订单已过期，修改该订单的状态并通过路由键`order.release.other`发送消息至队列`stock.release.stock.queue`进行库存解锁

* 库存锁定后延迟检查是否需要解锁库存

> 在库存锁定后通过`路由键stock.locked`发送至`延迟队列stock.delay.queue`，延迟时间到，死信通过`路由键stock.release`转发至`stock.release.stock.queue`,通过监听该队列进行判断当前订单状态，来确定库存是否需要解锁

* 由于`关闭订单`和`库存解锁`都有可能被执行多次，因此要保证业务逻辑的幂等性，在执行业务是重新查询当前的状态进行判断
* 订单关闭和库存解锁都会进行库存解锁的操作，来确保业务异常或者订单过期时库存会被可靠解锁

![](/Snipaste_2020-10-11_22-49-23.png)

<img src="/Snipaste_2020-10-11_22-41-45.png" style="zoom:67%;" />

#### (4) 创建业务交换机和队列

* 订单模块

```java
@Configuration
public class MyRabbitmqConfig {
    @Bean
    public Exchange orderEventExchange() {
        /**
         *   String name,
         *   boolean durable,
         *   boolean autoDelete,
         *   Map<String, Object> arguments
         */
        return new TopicExchange("order-event-exchange", true, false);
    }

    /**
     * 延迟队列
     * @return
     */
    @Bean
    public Queue orderDelayQueue() {
       /**
            Queue(String name,  队列名字
            boolean durable,  是否持久化
            boolean exclusive,  是否排他
            boolean autoDelete, 是否自动删除
            Map<String, Object> arguments) 属性
         */
        HashMap<String, Object> arguments = new HashMap<>();
        //死信交换机
        arguments.put("x-dead-letter-exchange", "order-event-exchange");
        //死信路由键
        arguments.put("x-dead-letter-routing-key", "order.release.order");
        arguments.put("x-message-ttl", 60000); // 消息过期时间 1分钟
        return new Queue("order.delay.queue",true,false,false,arguments);
    }

    /**
     * 普通队列
     *
     * @return
     */
    @Bean
    public Queue orderReleaseQueue() {

        Queue queue = new Queue("order.release.order.queue", true, false, false);

        return queue;
    }

    /**
     * 创建订单的binding
     * @return
     */
    @Bean
    public Binding orderCreateBinding() {
        /**
         * String destination, 目的地（队列名或者交换机名字）
         * DestinationType destinationType, 目的地类型（Queue、Exhcange）
         * String exchange,
         * String routingKey,
         * Map<String, Object> arguments
         * */
        return new Binding("order.delay.queue", Binding.DestinationType.QUEUE, "order-event-exchange", "order.create.order", null);
    }

    @Bean
    public Binding orderReleaseBinding() {
        return new Binding("order.release.order.queue",
                Binding.DestinationType.QUEUE,
                "order-event-exchange",
                "order.release.order",
                null);
    }

    @Bean
    public Binding orderReleaseOrderBinding() {
        return new Binding("stock.release.stock.queue",
                Binding.DestinationType.QUEUE,
                "order-event-exchange",
                "order.release.other.#",
                null);
    }
}
```

* 库存模块

```java
@Configuration
public class MyRabbitmqConfig {

    @Bean
    public Exchange stockEventExchange() {
        return new TopicExchange("stock-event-exchange", true, false);
    }

    /**
     * 延迟队列
     * @return
     */
    @Bean
    public Queue stockDelayQueue() {
        HashMap<String, Object> arguments = new HashMap<>();
        arguments.put("x-dead-letter-exchange", "stock-event-exchange");
        arguments.put("x-dead-letter-routing-key", "stock.release");
        // 消息过期时间 2分钟
        arguments.put("x-message-ttl", 120000);
        return new Queue("stock.delay.queue", true, false, false, arguments);
    }

    /**
     * 普通队列，用于解锁库存
     * @return
     */
    @Bean
    public Queue stockReleaseStockQueue() {
        return new Queue("stock.release.stock.queue", true, false, false, null);
    }


    /**
     * 交换机和延迟队列绑定
     * @return
     */
    @Bean
    public Binding stockLockedBinding() {
        return new Binding("stock.delay.queue",
                Binding.DestinationType.QUEUE,
                "stock-event-exchange",
                "stock.locked",
                null);
    }

    /**
     * 交换机和普通队列绑定
     * @return
     */
    @Bean
    public Binding stockReleaseBinding() {
        return new Binding("stock.release.stock.queue",
                Binding.DestinationType.QUEUE,
                "stock-event-exchange",
                "stock.release.#",
                null);
    }
}
```

#### (5) 库存自动解锁

##### 1）库存锁定

在库存锁定是添加以下逻辑

* 由于可能订单回滚的情况，所以为了能够得到库存锁定的信息，在锁定时需要记录库存工作单，其中包括订单信息和锁定库存时的信息(仓库id，商品id，锁了几件...)
* 在锁定成功后，向延迟队列发消息，带上库存锁定的相关信息

```java
@Transactional
@Override
public Boolean orderLockStock(WareSkuLockVo wareSkuLockVo) {
    //因为可能出现订单回滚后，库存锁定不回滚的情况，但订单已经回滚，得不到库存锁定信息，因此要有库存工作单
    WareOrderTaskEntity taskEntity = new WareOrderTaskEntity();
    taskEntity.setOrderSn(wareSkuLockVo.getOrderSn());
    taskEntity.setCreateTime(new Date());
    wareOrderTaskService.save(taskEntity);

    List<OrderItemVo> itemVos = wareSkuLockVo.getLocks();
    List<SkuLockVo> lockVos = itemVos.stream().map((item) -> {
        SkuLockVo skuLockVo = new SkuLockVo();
        skuLockVo.setSkuId(item.getSkuId());
        skuLockVo.setNum(item.getCount());
        List<Long> wareIds = baseMapper.listWareIdsHasStock(item.getSkuId(), item.getCount());
        skuLockVo.setWareIds(wareIds);
        return skuLockVo;
    }).collect(Collectors.toList());

    for (SkuLockVo lockVo : lockVos) {
        boolean lock = true;
        Long skuId = lockVo.getSkuId();
        List<Long> wareIds = lockVo.getWareIds();
        if (wareIds == null || wareIds.size() == 0) {
            throw new NoStockException(skuId);
        }else {
            for (Long wareId : wareIds) {
                Long count=baseMapper.lockWareSku(skuId, lockVo.getNum(), wareId);
                if (count==0){
                    lock=false;
                }else {
                    //锁定成功，保存工作单详情
                    WareOrderTaskDetailEntity detailEntity = WareOrderTaskDetailEntity.builder()
                            .skuId(skuId)
                            .skuName("")
                            .skuNum(lockVo.getNum())
                            .taskId(taskEntity.getId())
                            .wareId(wareId)
                            .lockStatus(1).build();
                    wareOrderTaskDetailService.save(detailEntity);
                    //发送库存锁定消息至延迟队列
                    StockLockedTo lockedTo = new StockLockedTo();
                    lockedTo.setId(taskEntity.getId());
                    StockDetailTo detailTo = new StockDetailTo();
                    BeanUtils.copyProperties(detailEntity,detailTo);
                    lockedTo.setDetailTo(detailTo);
                    rabbitTemplate.convertAndSend("stock-event-exchange","stock.locked",lockedTo);

                    lock = true;
                    break;
                }
            }
        }
        if (!lock) throw new NoStockException(skuId);
    }
    return true;
}
```

##### 2）监听队列

* 延迟队列会将过期的消息路由至`"stock.release.stock.queue"`,通过监听该队列实现库存的解锁
* 为保证消息的可靠到达，我们使用手动确认消息的模式，在解锁成功后确认消息，若出现异常则重新归队

```java
@Component
@RabbitListener(queues = {"stock.release.stock.queue"})
public class StockReleaseListener {

    @Autowired
    private WareSkuService wareSkuService;

    @RabbitHandler
    public void handleStockLockedRelease(StockLockedTo stockLockedTo, Message message, Channel channel) throws IOException {
        log.info("************************收到库存解锁的消息********************************");
        try {
            wareSkuService.unlock(stockLockedTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }
}
```

##### 3）库存解锁

* 如果工作单详情不为空，说明该库存锁定成功
  * 查询最新的订单状态，如果订单不存在，说明订单提交出现异常回滚，或者订单处于已取消的状态，我们都对已锁定的库存进行解锁
* 如果工作单详情为空，说明库存未锁定，自然无需解锁
* 为保证幂等性，我们分别对订单的状态和工作单的状态都进行了判断，只有当订单过期且工作单显示当前库存处于锁定的状态时，才进行库存的解锁

```java
 @Override
    public void unlock(StockLockedTo stockLockedTo) {
        StockDetailTo detailTo = stockLockedTo.getDetailTo();
        WareOrderTaskDetailEntity detailEntity = wareOrderTaskDetailService.getById(detailTo.getId());
        //1.如果工作单详情不为空，说明该库存锁定成功
        if (detailEntity != null) {
            WareOrderTaskEntity taskEntity = wareOrderTaskService.getById(stockLockedTo.getId());
            R r = orderFeignService.infoByOrderSn(taskEntity.getOrderSn());
            if (r.getCode() == 0) {
                OrderTo order = r.getData("order", new TypeReference<OrderTo>() {
                });
                //没有这个订单||订单状态已经取消 解锁库存
                if (order == null||order.getStatus()== OrderStatusEnum.CANCLED.getCode()) {
                    //为保证幂等性，只有当工作单详情处于被锁定的情况下才进行解锁
                    if (detailEntity.getLockStatus()== WareTaskStatusEnum.Locked.getCode()){
                        unlockStock(detailTo.getSkuId(), detailTo.getSkuNum(), detailTo.getWareId(), detailEntity.getId());
                    }
                }
            }else {
                throw new RuntimeException("远程调用订单服务失败");
            }
        }else {
            //无需解锁
        }
    }
```

#### (6) 定时关单

##### 1) 提交订单

```java
@Transactional
@Override
public SubmitOrderResponseVo submitOrder(OrderSubmitVo submitVo) {

    //提交订单的业务处理。。。
    
    //发送消息到订单延迟队列，判断过期订单
    rabbitTemplate.convertAndSend("order-event-exchange","order.create.order",order.getOrder());

               
}
```

##### 2) 监听队列

创建订单的消息会进入延迟队列，最终发送至队列`order.release.order.queue`，因此我们对该队列进行监听，进行订单的关闭

```java
@Component
@RabbitListener(queues = {"order.release.order.queue"})
public class OrderCloseListener {

    @Autowired
    private OrderService orderService;

    @RabbitHandler
    public void listener(OrderEntity orderEntity, Message message, Channel channel) throws IOException {
        System.out.println("收到过期的订单信息，准备关闭订单" + orderEntity.getOrderSn());
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            orderService.closeOrder(orderEntity);
            channel.basicAck(deliveryTag,false);
        } catch (Exception e){
            channel.basicReject(deliveryTag,true);
        }

    }
}
```

##### 3) 关闭订单

* 由于要保证幂等性，因此要查询最新的订单状态判断是否需要关单
* 关闭订单后也需要解锁库存，因此发送消息进行库存、会员服务对应的解锁

```java
@Override
public void closeOrder(OrderEntity orderEntity) {
    //因为消息发送过来的订单已经是很久前的了，中间可能被改动，因此要查询最新的订单
    OrderEntity newOrderEntity = this.getById(orderEntity.getId());
    //如果订单还处于新创建的状态，说明超时未支付，进行关单
    if (newOrderEntity.getStatus() == OrderStatusEnum.CREATE_NEW.getCode()) {
        OrderEntity updateOrder = new OrderEntity();
        updateOrder.setId(newOrderEntity.getId());
        updateOrder.setStatus(OrderStatusEnum.CANCLED.getCode());
        this.updateById(updateOrder);

        //关单后发送消息通知其他服务进行关单相关的操作，如解锁库存
        OrderTo orderTo = new OrderTo();
        BeanUtils.copyProperties(newOrderEntity,orderTo);
        rabbitTemplate.convertAndSend("order-event-exchange", "order.release.other",orderTo);
    }
}
```

##### 4) 解锁库存

```java
@Slf4j
@Component
@RabbitListener(queues = {"stock.release.stock.queue"})
public class StockReleaseListener {

    @Autowired
    private WareSkuService wareSkuService;

    @RabbitHandler
    public void handleStockLockedRelease(StockLockedTo stockLockedTo, Message message, Channel channel) throws IOException {
        log.info("************************收到库存解锁的消息********************************");
        try {
            wareSkuService.unlock(stockLockedTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }

    @RabbitHandler
    public void handleStockLockedRelease(OrderTo orderTo, Message message, Channel channel) throws IOException {
        log.info("************************从订单模块收到库存解锁的消息********************************");
        try {
            wareSkuService.unlock(orderTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }
}
```

```java
@Override
public void unlock(OrderTo orderTo) {
    //为防止重复解锁，需要重新查询工作单
    String orderSn = orderTo.getOrderSn();
    WareOrderTaskEntity taskEntity = wareOrderTaskService.getBaseMapper().selectOne((new QueryWrapper<WareOrderTaskEntity>().eq("order_sn", orderSn)));
    //查询出当前订单相关的且处于锁定状态的工作单详情
    List<WareOrderTaskDetailEntity> lockDetails = wareOrderTaskDetailService.list(new QueryWrapper<WareOrderTaskDetailEntity>().eq("task_id", taskEntity.getId()).eq("lock_status", WareTaskStatusEnum.Locked.getCode()));
    for (WareOrderTaskDetailEntity lockDetail : lockDetails) {
        unlockStock(lockDetail.getSkuId(),lockDetail.getSkuNum(),lockDetail.getWareId(),lockDetail.getId());
    }
}
```



##### 3、库存自动解锁--->MQ

###### 库存解锁、StockReleaseListener

```java
package com.atguigu.gulimall.ware.listener;

import com.atguigu.common.to.mq.OrderTo;
import com.atguigu.common.to.mq.StockLockedTo;
import com.atguigu.gulimall.ware.service.WareSkuService;
import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;

/**
 * 监听库存延时队列
 * @author gcq
 * @Create 2021-01-07
 */
@Service
@RabbitListener(queues = "stock.release.stock.queue")
public class StockReleaseListener {

    @Autowired
    WareSkuService wareSkuService;

    /**
     * 监听库存队列
     * @param lockedTo
     * @param message
     */
    @RabbitHandler
    public void handleStockLockedRelease(StockLockedTo lockedTo, Message message, Channel channel) throws IOException {
        System.out.println("收到解锁库存的信息");
        try {
            wareSkuService.unLockStock(lockedTo);
            //库存解锁成功没有抛出异常，自动ack机制确认
            channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
        } catch (Exception e) {
            e.printStackTrace();
            // 重发
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }

    /**
     * 订单释放
     *  订单30分钟未支付，订单关闭后发送的消息
     */
    @RabbitHandler
    public void handlerOrderCloseRelease(OrderTo orderTo, Message message, Channel channel) throws IOException {
        System.out.println("订单关闭准备解锁库存......");
        try {
            wareSkuService.unLockStock(orderTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
        } catch (Exception e) {
            e.printStackTrace();
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }
}
```

Service

```java
 /**
     * 解锁库存
     * @param lockedTo
     */
    @Override
    public void unLockStock(StockLockedTo lockedTo) {
        // 工作单详情
        StockDetailTo detail = lockedTo.getDetail();
        // 工作单详情id
        Long detailId = detail.getId();
        // 查询到库存工作单详情
        WareOrderTaskDetailEntity taskDetailEntity = wareOrderTaskDetailService.getById(detailId);
        if (taskDetailEntity != null) {
            // 解锁库存
            // 库存工作单id
            Long id = lockedTo.getId();
            // 查询到库存工作单
            WareOrderTaskEntity TaskEntity = wareOrderTaskService.getById(id);
            // 拿到订单号
            String orderSn = TaskEntity.getOrderSn();
            // 根据订单号查询订单的状态
            R orderStatus = orderFeignService.getOrderStatus(orderSn);
            if (orderStatus.getCode() == 0) {
                OrderVo data = orderStatus.getData(new TypeReference<OrderVo>() {
                });
                // 订单状态为已关闭,那么就需要解锁库存
                if (data == null || data.getStatus() == 4) {
                    // 库存工作单锁定状态为锁定才进行解锁
                    if (taskDetailEntity.getLockStatus() == 1) {
                        unLockStock(detail.getSkuId(), detail.getWareId(), detail.getSkuNum(), detailId);
                    }
                }
            } else {
                // 消息被拒绝后重新放到队列里面，让别人继续消费解锁
                throw new RuntimeException("远败");
            }
        }
    }
```



#### 4、支付

选择的是支付宝支付，根据老师所提供的素材 **alipayTemplate、PayVo、PaySyncVo**,引入到项目中进行开发

##### 1、Controller

跳转到支付宝支付页面，**支付完成后跳转到支付成功的回调页面**

```java
/**
     * 1、跳转到支付页面
     * 2、用户支付成功后，我们要跳转到用户的订单列表页
     * produces 明确方法会返回什么类型，这里返回的是html页面
     * @param orderSn
     * @return
     * @throws AlipayApiException
     */
    @ResponseBody
    @GetMapping(value = "/payOrder",produces = "text/html")
    public String payOrder(@RequestParam("orderSn") String orderSn) throws AlipayApiException {
//        PayVo payVo = new PayVo();
//        payVo.setBody(); // 商品描述
//        payVo.setSubject(); //订单名称
//        payVo.setOut_trade_no(); // 订单号
//        payVo.setTotal_amount(); //总金额
        PayVo payvo = orderService.payOrder(orderSn);
        // 将返回支付宝的支付页面，需要将这个页面进行显示
        String pay = alipayTemplate.pay(payvo);
        System.out.println(pay);
        return pay;
    }
```

##### 2、Service

```java
 /**
     * 计算商品支付需要的信息
     * @param orderSn
     * @return
     */
    @Override
    public PayVo payOrder(String orderSn) {
        PayVo payVo = new PayVo();
        OrderEntity orderEntity = this.getOrderByOrderSn(orderSn);// 根据订单号查询到商品
        // 数据库中付款金额小数有4位，但是支付宝只接受2位，所以向上取整两位数
        BigDecimal decimal = orderEntity.getPayAmount().setScale(2, BigDecimal.ROUND_UP);
        payVo.setTotal_amount(decimal.toString());
        // 商户订单号
        payVo.setOut_trade_no(orderSn);
        // 查询出订单项，用来设置商品的描述和商品名称
        List<OrderItemEntity> itemEntities = orderItemService.list(new QueryWrapper<OrderItemEntity>()
                .eq("order_sn", orderSn));
        OrderItemEntity itemEntity = itemEntities.get(0);
        // 订单名称使用商品项的名字
        payVo.setSubject(itemEntity.getSkuName());
        // 商品的描述使用商品项的属性
        payVo.setBody(itemEntity.getSkuAttrsVals());
        return payVo;
    }
```

##### 3、支付成功后跳转页面

支付成功后跳转到**订单页面**

```java
 /**
    * @需求描述: 系统管理员 会员服务组 模块 用户支付成功后跳转到该页面
    * @创建人: 
    * @创建时间: 2021/01/08 11:13
    * @修改需求:
    * @修改人:
    * @修改时间:
    * @需求思路:
    */
    @GetMapping("/memberOrder.html")
    public String memberList(@RequestParam(value = "pageNum",defaultValue = "1") Integer pageNum, Model model) {
        // 准备分页参数
        Map<String,Object> params = new HashMap<>();
        params.put("page",pageNum);
        // 远程查询当前用户的所有订单
        R r = orderFeignService.listwithItem(params);
        System.out.println(JSON.toJSONString(r));
        if (r.getCode() == 0) {
            model.addAttribute("orders",r);
        }
        return "list";
    }
```

Service

```java
 /**
     * 查询当前用户所有订单
     * @param params
     * @return
     */
    @Override
    public PageUtils queryPageWithItem(Map<String, Object> params) {
        // 当前用户登录数据
        MemberRespVo memberRespVo = LoginInterceptor.loginUser.get();
        // 查询当前用户所有的订单记录
        IPage<OrderEntity> page = this.page(
                new Query<OrderEntity>().getPage(params),
                new QueryWrapper<OrderEntity>()
                        .eq("member_id",memberRespVo.getId())
                        .orderByDesc("id")
        );
        List<OrderEntity> records = page.getRecords(); // 拿到分页查询结果
        List<OrderEntity> orderEntityList = records.stream().map(item -> {
            // 根据订单号查询当订单号对应的订单项
            List<OrderItemEntity> itemEntities = orderItemService.list(new QueryWrapper<OrderItemEntity>()
                    .eq("order_sn", item.getOrderSn()));
            item.setOrderEntityItem(itemEntities);
            return item;
        }).collect(Collectors.toList());
        // 重新设置分页数据
        page.setRecords(orderEntityList);

        return new PageUtils(page);
    }
```

然后页面渲染数据

支付宝文档地址:

https://opendocs.alipay.com/open/270/105900

![image-20210108143131033](image-20210108143131033.png)

![image-20210105093259313](image-20210105093259313.png)

#### 5、收单

##### 订单：OrderListener

```java
@RabbitListener(queues = "order.release.order.queue") // 监听订单的释放队列，能到这个里面的消息都是30分钟后过来的
@Service
public class OrderListener {

    @Autowired
    OrderService orderService;

    /**
     * 订单定时关单
     *      商品下单后，会向MQ中发送一条消息告诉MQ订单创建成功。
     *      那么订单创建30分钟后用户还没有下单，MQ就会关闭该订单
     * @param orderEntity 订单对象
     * @param channel 信道
     * @param message 交换机
     * @throws IOException
     */
    @RabbitHandler
    public void listener(OrderEntity orderEntity, Channel channel, Message message) throws IOException {
        System.out.println("收到过期的订单信息，准备关闭订单：" + orderEntity.getOrderSn());
        try {
            orderService.closeOrder(orderEntity);
            // 关闭订单成功后，ack信息确认
            channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
        } catch (Exception e) {
            e.printStackTrace();
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }

}
```

Service

```java
   @Override
    public void closeOrder(OrderEntity orderEntity) {
        // 订单30分钟的时间可能有属性变动，所以需要根据属性再次查询一次
        OrderEntity entity = this.getById(orderEntity.getId());
        // 当前状态为待付款，说明用户30分钟内还没有付款
        if(entity.getStatus() == OrderStatusEnum.CREATE_NEW.getCode()) {
            OrderEntity updateOrder = new OrderEntity();
            // 根据订单id更新
            updateOrder.setId(entity.getId());
            // 订单状态改成已取消
            updateOrder.setStatus(OrderStatusEnum.CANCLED.getCode());
            // 根据订单对象更新
            this.updateById(updateOrder);
            // 准备共享对象用于发送到MQ中
            OrderTo orderTo = new OrderTo();
            // 拷贝属性
            BeanUtils.copyProperties(entity,orderTo);
            try {
                rabbitTemplate.convertAndSend("order-event-exchange","order.release.other",orderTo);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```



![image-20210105093437013](image-20210105093437013.png)

- 订单在支付页，不支付，一直刷新，订单过期了才支付，订单状态改为已支付了，但是库存解锁了。
  - 使用支付宝自动收单功能解决。只要一段时间不支付，就不能支付了。
- 由于时延等问题。订单解锁完成，正在解锁库存的时候，异步通知才到
  - 订单解锁，手动调用收单
- 网络阻塞问题，订单支付成功的异步通知一直不到达
  - 查询订单列表时，ajax获取当前未支付的订单状态，查询订单状态时，再获取一下支付宝此订单的状态
- 其他各种问题
  - 每天晚上闲时下载支付宝对账单，一 一 进行对账

# 15、幂等性

### 15.1 什么是幂等性

**接口幂等性就是用户对同一操作发起的一次请求和多次请求结果是一致的**，不会因为多次点击而产生了副作用，比如支付场景，用户购买了商品，支付扣款成功，但是返回结果的时候出现了网络异常，此时钱已经扣了，用户再次点击按钮，此时就会进行第二次扣款，返回结果成功，用户查询余额发现多扣钱了，流水记录也变成了两条。。。这就没有保证接口幂等性

### 15.2 那些情况需要防止

用户多次点击按钮

用户页面回退再次提交

微服务互相调用，由于网络问题，导致请求失败，feign触发重试机制

其他业务情况

### 15.3 什么情况下需要幂等

以 SQL 为例，有些操作时天然**幂等**的

SELECT * FROM table WHERE id =? 无论执行多少次都不会改变状态是天然的**幂等**

UPDATE tab1 SET col1=1 WHERE col2=2 无论执行成功多少状态都是一致的，也是**幂等**操作

delete from user where userid=1 多次操作，结果一样，具备**幂等**性

insert into user(userid,name) values(1,' a' ) 如userid为唯一主键，即重复上面的业务，只会插入一条用户记录，具备**幂等**性

------

UPDATE tab1 SET col1=col1+1 WHERE col2=2,每次执行的结果都会发生变化，不是**幂等**的。insert into user(userid,name) values(,a")如userid不是主键，可以重复，那上面业务多次操作，数据都会新增多条，不具备**幂等**性。

### 15.4 幂等解决方案

#### 1、“一次性纸杯方案”--token 机制-针对用户快速点击页面提交按钮等操作

1、服务端提供了发送 `token` 的接口，我们在分析业务的时候，哪些业务是存在幂等性问题的，就必须在执行业务前，先获取 `token`，服务器会把 `token` 保存到 redis 中

2、然后调用业务接口请求时， 把 `token` 携带过去，一般放在请求头部

3、服务器判断 `token` 是否存在 `redis`，存在表示第一次请求，然后删除 `token`，继续执行业务

4、如果判断 `token` 不存在 `redis` 中，就表示重复操作，直接返回重复标记给 `client`，这样就保证了业务代码，不被重复执行

**校验触发过程**

服务器返回的是携带token的页面->用户在token页面上连续点击，发送多个携带token的请求->通过校验token，拦截重复请求。

危险性：

**1、执行业务代码时，先删除 token 还是后删除 token：**

   1.后删除问题比较严重，

- 无法拦截两个同时过来的请求。两个相同请求同时过来。一个正在校验token，然后执行业务代码。此时还没有删除token，可能另一个请求也过来了，也校验了token，执行业务代码。这样token'没有起到拦截重复请求的作用，token机制失效了。 

- 服务闪断，导致token删除失败。可能导致，业务处理成功，但是服务闪断，出现超时，没有删除掉token，别人继续重试，导致业务被执行两次 

2. 先删除可能导致，业务确实没有执行，重试还得带上之前的 token, 由于防重设计导致，请求还是不能执行.

所以，我们最后设计为先删除 token，如果业务调用失败，就重新获取 token 再次请求

**2、Token 获取，比较 和删除 必须是原子性**

1. redis.get（token），token.equals、redis.del（token）,如果说这两个操作都不是原子，可能导致，在高并发下，都 get 同样的数据，判断都成功，继续业务并发执行
2. 可以在 redis 使用 lua 脚本完成这个操作

```java
"if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end"
```

> 通过引入令牌机制，我们将一个存在非幂等操作的页面，变成了“一次性纸杯”，token页面只能发起一次有效操作。

#### 2、各种锁机制

##### 1、数据库悲观锁

select * from xxx where id = 1 for update;

for update 查询的时候锁定这条记录 别人需要等待

悲观锁使用的时候一般伴随事务一起使用，数据锁定时间可能会很长，需要根据实际情况选用，另外需要注意的是，id字段一定是主键或唯一索引，不然可能造成锁表的结果，处理起来会非常麻烦

##### 2、数据库的乐观锁

这种方法适合在更新的场景中

update t_goods set count = count - 1,version = version + 1 where good_id = 2 and version = 1

根据 version 版本，也就是在操作数据库存前先获取当前商品的 version 版本号，然后操作的时候带上 version 版本号，我们梳理下，我们第一次操作库存时，得

到 version 为 1，调用库存服务 version = 2，但返回给订单服务出现了问题，订单服务又一次调用了库存服务，当订单服务传的 version 还是 1，再执行上面的

 sql 语句 就不会执行，因为 version 已经变成 2 了，where 条件不成立，这样就保证了不管调用几次，只会真正处理一次，乐观锁主要使用于处理读多写少的问题

##### 3、业务层分布锁

如果多个机器可能在同一时间处理相同的数据，比如多台机器定时任务拿到了相同的数据，我们就可以加分布式锁，锁定此数据，处理完成后后释放锁，获取锁必须先判断这个数据是否被处理过

#### 3、各种唯一约束

##### 1、数据库唯一约束

插入数据，应该按照唯一索引进行插入，比如订单号，相同订单就不可能有两条订单插入，我们在数据库层面防止重复

这个机制利用了数据库的主键唯一约束的特性，解决了 insert场 景时幂等问题，但主键的要求不是自增的主键，这样就需要业务生成全局唯一的主键

如果是分库分表场景下，路由规则要保证相同请求下，落地在同一个数据库和同一表中，要不然数据库主键约束就不起效果了，因为是不同的数据库和表主键不相关

##### 2、redis set 防重

很多数据需要处理，只能被处理一次，比如我们可以计算数据的 MD5 将其放入 redis 的

set,每次处理数据，先看这个 MD5 是否已经存在，存在就不处理。比如百度网盘发现某个上传文件的md5已经存在，就不做上传。



#### 4、防重表

使用订单表 `orderNo` 做为去重表的唯一索引，把唯一索引插入去重表，再进行业务操作，且他们在同一个事务中，这样就保证了重复请求时，因为去重表有唯一

约束，导致请求失败，避免了幂等性等问题，去重表和业务表应该在同一个库中，这样就保证了在同一个事务，即使业务操作失败，也会把去重表的数据回滚，这

个很好的保证了数据的一致性，

redis防重也算

#### 5、全局请求唯一id

调用接口时，生成一个唯一的id，redis 将数据保存到集合中（去重），存在即处理过，可以使用 nginx 设置每一个请求一个唯一id

proxy_set_header X-Request-Id $Request_id



# 16、本地事务

### 16.1 事务的基本性质

数据库事物的几个特性：原子性（Atomiccity）、一致性（Consistetcy）、隔离性或独立性（Isolation）和持久性（Durabilily），简称ACID

- **原子性：**一系列操作整体不可拆分，要么同时成功要么同时失败
- **一致性**：数据在业务的前后，业务整体一致
  - 转账 A:1000 B:1000  转200  事务成功 A：800 B：1200
- **隔离性：**事物之间需要相互隔离
- **持久性：**一旦事务成功，数据一定会落盘在数据库



> 事务就像一个严肃的班主任，对内要求学生团结，对外划清界限

在以往的单体应用中，我们多个业务操作使用同一条连接操作不同的表，一旦有异常我们很容易整体回滚

![image-20201119115615054](image-20201119115615054.png)

**Business**：我们具体的业务代码
**Storage**：库存业务代码;扣库存
**Order**：订单业务代码;保存订单
**Account**：账号业务代码:减账户余额
比如买东西业务，扣库存，下订单，账户扣款，是一个整体；必须同时成功或者失败



一个事务开始，代表以下的所有操作都在同一个连接里面



### 16.2 事物的隔离级别

- **READ UNCOMMITEED(读未提交)**
  - 该隔离级别的事务会读到其他未提交事务的数据，此现象也称为脏读
- **READ COMMITTED（读提交）**
  - 一个事物可以读取另一个已提交的事务，多次读取会造成不一样的结果，此现象称为不可重复读，复读问题，Oracle 和SQL Server 的默认隔离级别
- **REPEATABLE READ（可重复读）**
  - 该隔离级别是 MySQL 默认的隔离级别，在同一个事物中，SELECT 的结果是事务开始时时间点的状态，因此，同样的 SELECT 操作读到的结果会是一致
  - 但是会有幻读一致,MySQL的 InnoDB 引擎可以通过 next-key locks 机制 来避免幻读
- **SERIALIZABLE（序列化）**
  - 在该隔离级别下事务都是串行顺序执行的，MySQL 数据库的 InnoDB 引擎会给读操作隐士加一把读共享锁，从而避免了脏读、不可重复读、幻读等问题

越大的隔离级别，并发能力越低



### 16.3 事务传播行为

**PROPAGATION_REQUIRED：**如果当前没事事务，就创建一个新事务，如果当前存在事务，就加入该事务，该设置时最常使用的配置

**PROPAGATION_SUPPORTS：**支持当前事务。如果当前存在事务，就加入该事务，如果当前不存在事务，就以非事务执行

**PROPAGATION_MANDATORY：**支持当前事务，如果当前存在事务，就加入该事务，如果当前不存在该事务，就抛出异常

**PROPAGATION_REQUIRES_NEW：**创建新事务，无论当前存不存在事务，都创建新事务

**PROPAGATION_NOT_SUPPORTED：**以非事务的方式执行操作，如果当前存在事务，就把当前事务挂起

**PROPAGATION_NEVER：**以非事务方式运行，如果当前存在事务，则抛出异常 	

**PROPAGATION_NESTED：**如果当前存在事务，就嵌套在该事务内执行，如果当前没有事务，则执行PROPAGATION_REQUIRED相关操作

### 16.4 SpringBoot 事务关键点

在同一个事务内编写两个方法，内部调用的时候，会导致事务失效，原因是没有用到代理对象的缘故

解决

0)、导入spring-boot-starter-aop
1)、@EnableTransactionManagement(proxyTargetClass= true)
2)、@EnableAspectJAutoProxy(lexposeProxy-true)
3)、AopContext.currentProxy() 调用方法

#### 1、事物的自动配置

#### 2、事物的坑



#### （3） 本地事务及其问题

分布式情况下，可能出现一些服务事务不一致的情况

* 远程服务假失败

* 远程服务执行完成后，下面其他方法出现异常。

  这里我们通过10/0，模拟远程扣减积分出问题，发现订单回滚，扣库存没有回滚。

<img src="/Snipaste_2020-10-11_09-15-30-1624450747857.png" style="zoom:38%;" />

本地事务，在分布式系统，只能控制住自己的回滚，控制不了其他服务的回滚

分布式事务问题出现的最大原因，就是网络问题+分布式机器。

#### （4）使用seata解决分布式事务问题及其缺点

导入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

环境搭建

下载senta-server-0.7.1并修改`register.conf`,使用nacos作为注册中心

```shell
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    serverAddr = "#:8848"
    namespace = "public"
    cluster = "default"
  }
```

将`register.conf`和`file.conf`复制到需要开启分布式事务的根目录，并修改`file.conf`

 `vgroup_mapping.${application.name}-fescar-service-group = "default"`

```shell
service {
  #vgroup->rgroup
  vgroup_mapping.gulimall-ware-fescar-service-group = "default"
  #only support single node
  default.grouplist = "127.0.0.1:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
```

使用seata包装数据源

```java
@Configuration
public class MySeataConfig {
    @Autowired
    DataSourceProperties dataSourceProperties;

    @Bean
    public DataSource dataSource(DataSourceProperties dataSourceProperties) {

        HikariDataSource dataSource = dataSourceProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
        if (StringUtils.hasText(dataSourceProperties.getName())) {
            dataSource.setPoolName(dataSourceProperties.getName());
        }
        return new DataSourceProxy(dataSource);
    }
}
```

在大事务的入口标记注解`@GlobalTransactional`开启全局事务，并且每个小事务标记注解`@Transactional`

```java
@GlobalTransactional
@Transactional
@Override
public SubmitOrderResponseVo submitOrder(OrderSubmitVo submitVo) {
}
```


seata的问题：为了保证高并发，不推荐使用seata，因为是加锁，并行化，提升不了效率


### 5. 使用消息队列实现最终一致性

#### (1) 延迟队列的定义与实现

* 定义：

  延迟队列存储的对象肯定是对应的延时消息，所谓"延时消息"是指当消息被发送以后，并不想让消费者立即拿到消息，而是等待指定时间后，消费者才拿到这个消息进行消费。

* 实现：

  rabbitmq可以通过设置队列的`TTL`和死信路由实现延迟队列

  * TTL：

  >RabbitMQ可以针对Queue设置x-expires 或者 针对Message设置 x-message-ttl，来控制消息的生存时间，如果超时(两者同时设置以最先到期的时间为准)，则消息变为dead letter(死信)

  

  * 死信路由DLX

  >RabbitMQ的Queue可以配置x-dead-letter-exchange 和x-dead-letter-routing-key（可选）两个参数，如果队列内出现了dead letter，则按照这两个参数重新路由转发到指定的队列。

  >- x-dead-letter-exchange：出现dead letter之后将dead letter重新发送到指定exchange
  >- x-dead-letter-routing-key：出现dead letter之后将dead letter重新按照指定的routing-key发送

<img src="/Snipaste_2020-10-11_17-00-18-1624450747858.png" style="zoom: 50%;" />

针对订单模块创建以上消息队列，创建订单时消息会被发送至队列`order.delay.queue`，经过`TTL`的时间后消息会变成死信以`order.release.order`的路由键经交换机转发至队列`order.release.order.queue`，再通过监听该队列的消息来实现过期订单的处理

#### (2) 延迟队列使用场景

<img src="/Snipaste_2020-10-14_15-42-12-1624450747858.png" style="zoom: 25%;" />

**为什么不能用定时任务完成？**

如果恰好在一次扫描后完成业务逻辑，那么就会等待两个扫描周期才能扫到过期的订单，不能保证时效性

<img src="/Snipaste_2020-10-14_15-43-37-1624450747859.png" style="zoom: 25%;" />



#### (3) 定时关单与库存解锁主体逻辑

* 订单超时未支付触发订单过期状态修改与库存解锁

> 创建订单时消息会被发送至队列`order.delay.queue`，经过`TTL`的时间后消息会变成死信以`order.release.order`的路由键经交换机转发至队列`order.release.order.queue`，再通过监听该队列的消息来实现过期订单的处理
>
> * 如果该订单已支付，则无需处理
> * 否则说明该订单已过期，修改该订单的状态并通过路由键`order.release.other`发送消息至队列`stock.release.stock.queue`进行库存解锁

* 库存锁定后延迟检查是否需要解锁库存

> 在库存锁定后通过`路由键stock.locked`发送至`延迟队列stock.delay.queue`，延迟时间到，死信通过`路由键stock.release`转发至`stock.release.stock.queue`,通过监听该队列进行判断当前订单状态，来确定库存是否需要解锁

* 由于`关闭订单`和`库存解锁`都有可能被执行多次，因此要保证业务逻辑的幂等性，在执行业务是重新查询当前的状态进行判断
* 订单关闭和库存解锁都会进行库存解锁的操作，来确保业务异常或者订单过期时库存会被可靠解锁

![](/Snipaste_2020-10-11_22-49-23-1624450747859.png)

<img src="/Snipaste_2020-10-11_22-41-45-1624450747859.png" style="zoom:67%;" />

#### (4) 创建业务交换机和队列

* 订单模块

```java
@Configuration
public class MyRabbitmqConfig {
    @Bean
    public Exchange orderEventExchange() {
        /**
         *   String name,
         *   boolean durable,
         *   boolean autoDelete,
         *   Map<String, Object> arguments
         */
        return new TopicExchange("order-event-exchange", true, false);
    }

    /**
     * 延迟队列
     * @return
     */
    @Bean
    public Queue orderDelayQueue() {
       /**
            Queue(String name,  队列名字
            boolean durable,  是否持久化
            boolean exclusive,  是否排他
            boolean autoDelete, 是否自动删除
            Map<String, Object> arguments) 属性
         */
        HashMap<String, Object> arguments = new HashMap<>();
        //死信交换机
        arguments.put("x-dead-letter-exchange", "order-event-exchange");
        //死信路由键
        arguments.put("x-dead-letter-routing-key", "order.release.order");
        arguments.put("x-message-ttl", 60000); // 消息过期时间 1分钟
        return new Queue("order.delay.queue",true,false,false,arguments);
    }

    /**
     * 普通队列
     *
     * @return
     */
    @Bean
    public Queue orderReleaseQueue() {

        Queue queue = new Queue("order.release.order.queue", true, false, false);

        return queue;
    }

    /**
     * 创建订单的binding
     * @return
     */
    @Bean
    public Binding orderCreateBinding() {
        /**
         * String destination, 目的地（队列名或者交换机名字）
         * DestinationType destinationType, 目的地类型（Queue、Exhcange）
         * String exchange,
         * String routingKey,
         * Map<String, Object> arguments
         * */
        return new Binding("order.delay.queue", Binding.DestinationType.QUEUE, "order-event-exchange", "order.create.order", null);
    }

    @Bean
    public Binding orderReleaseBinding() {
        return new Binding("order.release.order.queue",
                Binding.DestinationType.QUEUE,
                "order-event-exchange",
                "order.release.order",
                null);
    }

    @Bean
    public Binding orderReleaseOrderBinding() {
        return new Binding("stock.release.stock.queue",
                Binding.DestinationType.QUEUE,
                "order-event-exchange",
                "order.release.other.#",
                null);
    }
}
```

* 库存模块

```java
@Configuration
public class MyRabbitmqConfig {

    @Bean
    public Exchange stockEventExchange() {
        return new TopicExchange("stock-event-exchange", true, false);
    }

    /**
     * 延迟队列
     * @return
     */
    @Bean
    public Queue stockDelayQueue() {
        HashMap<String, Object> arguments = new HashMap<>();
        arguments.put("x-dead-letter-exchange", "stock-event-exchange");
        arguments.put("x-dead-letter-routing-key", "stock.release");
        // 消息过期时间 2分钟
        arguments.put("x-message-ttl", 120000);
        return new Queue("stock.delay.queue", true, false, false, arguments);
    }

    /**
     * 普通队列，用于解锁库存
     * @return
     */
    @Bean
    public Queue stockReleaseStockQueue() {
        return new Queue("stock.release.stock.queue", true, false, false, null);
    }


    /**
     * 交换机和延迟队列绑定
     * @return
     */
    @Bean
    public Binding stockLockedBinding() {
        return new Binding("stock.delay.queue",
                Binding.DestinationType.QUEUE,
                "stock-event-exchange",
                "stock.locked",
                null);
    }

    /**
     * 交换机和普通队列绑定
     * @return
     */
    @Bean
    public Binding stockReleaseBinding() {
        return new Binding("stock.release.stock.queue",
                Binding.DestinationType.QUEUE,
                "stock-event-exchange",
                "stock.release.#",
                null);
    }
}
```

#### (5) 库存自动解锁

##### 1）库存锁定

在库存锁定是添加以下逻辑

* 由于可能订单回滚的情况，所以为了能够得到库存锁定的信息，在锁定时需要记录库存工作单，其中包括订单信息和锁定库存时的信息(仓库id，商品id，锁了几件...)
* 在锁定成功后，向延迟队列发消息，带上库存锁定的相关信息

```java
@Transactional
@Override
public Boolean orderLockStock(WareSkuLockVo wareSkuLockVo) {
    //因为可能出现订单回滚后，库存锁定不回滚的情况，但订单已经回滚，得不到库存锁定信息，因此要有库存工作单
    WareOrderTaskEntity taskEntity = new WareOrderTaskEntity();
    taskEntity.setOrderSn(wareSkuLockVo.getOrderSn());
    taskEntity.setCreateTime(new Date());
    wareOrderTaskService.save(taskEntity);

    List<OrderItemVo> itemVos = wareSkuLockVo.getLocks();
    List<SkuLockVo> lockVos = itemVos.stream().map((item) -> {
        SkuLockVo skuLockVo = new SkuLockVo();
        skuLockVo.setSkuId(item.getSkuId());
        skuLockVo.setNum(item.getCount());
        List<Long> wareIds = baseMapper.listWareIdsHasStock(item.getSkuId(), item.getCount());
        skuLockVo.setWareIds(wareIds);
        return skuLockVo;
    }).collect(Collectors.toList());

    for (SkuLockVo lockVo : lockVos) {
        boolean lock = true;
        Long skuId = lockVo.getSkuId();
        List<Long> wareIds = lockVo.getWareIds();
        if (wareIds == null || wareIds.size() == 0) {
            throw new NoStockException(skuId);
        }else {
            for (Long wareId : wareIds) {
                Long count=baseMapper.lockWareSku(skuId, lockVo.getNum(), wareId);
                if (count==0){
                    lock=false;
                }else {
                    //锁定成功，保存工作单详情
                    WareOrderTaskDetailEntity detailEntity = WareOrderTaskDetailEntity.builder()
                            .skuId(skuId)
                            .skuName("")
                            .skuNum(lockVo.getNum())
                            .taskId(taskEntity.getId())
                            .wareId(wareId)
                            .lockStatus(1).build();
                    wareOrderTaskDetailService.save(detailEntity);
                    //发送库存锁定消息至延迟队列
                    StockLockedTo lockedTo = new StockLockedTo();
                    lockedTo.setId(taskEntity.getId());
                    StockDetailTo detailTo = new StockDetailTo();
                    BeanUtils.copyProperties(detailEntity,detailTo);
                    lockedTo.setDetailTo(detailTo);
                    rabbitTemplate.convertAndSend("stock-event-exchange","stock.locked",lockedTo);

                    lock = true;
                    break;
                }
            }
        }
        if (!lock) throw new NoStockException(skuId);
    }
    return true;
}
```

##### 2）监听队列

* 延迟队列会将过期的消息路由至`"stock.release.stock.queue"`,通过监听该队列实现库存的解锁
* 为保证消息的可靠到达，我们使用手动确认消息的模式，在解锁成功后确认消息，若出现异常则重新归队

```java
@Component
@RabbitListener(queues = {"stock.release.stock.queue"})
public class StockReleaseListener {

    @Autowired
    private WareSkuService wareSkuService;

    @RabbitHandler
    public void handleStockLockedRelease(StockLockedTo stockLockedTo, Message message, Channel channel) throws IOException {
        log.info("************************收到库存解锁的消息********************************");
        try {
            wareSkuService.unlock(stockLockedTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }
}
```

##### 3）库存解锁

* 如果工作单详情不为空，说明该库存锁定成功
  * 查询最新的订单状态，如果订单不存在，说明订单提交出现异常回滚，或者订单处于已取消的状态，我们都对已锁定的库存进行解锁
* 如果工作单详情为空，说明库存未锁定，自然无需解锁
* 为保证幂等性，我们分别对订单的状态和工作单的状态都进行了判断，只有当订单过期且工作单显示当前库存处于锁定的状态时，才进行库存的解锁

```java
 @Override
    public void unlock(StockLockedTo stockLockedTo) {
        StockDetailTo detailTo = stockLockedTo.getDetailTo();
        WareOrderTaskDetailEntity detailEntity = wareOrderTaskDetailService.getById(detailTo.getId());
        //1.如果工作单详情不为空，说明该库存锁定成功
        if (detailEntity != null) {
            WareOrderTaskEntity taskEntity = wareOrderTaskService.getById(stockLockedTo.getId());
            R r = orderFeignService.infoByOrderSn(taskEntity.getOrderSn());
            if (r.getCode() == 0) {
                OrderTo order = r.getData("order", new TypeReference<OrderTo>() {
                });
                //没有这个订单||订单状态已经取消 解锁库存
                if (order == null||order.getStatus()== OrderStatusEnum.CANCLED.getCode()) {
                    //为保证幂等性，只有当工作单详情处于被锁定的情况下才进行解锁
                    if (detailEntity.getLockStatus()== WareTaskStatusEnum.Locked.getCode()){
                        unlockStock(detailTo.getSkuId(), detailTo.getSkuNum(), detailTo.getWareId(), detailEntity.getId());
                    }
                }
            }else {
                throw new RuntimeException("远程调用订单服务失败");
            }
        }else {
            //无需解锁
        }
    }
```

#### (6) 定时关单

##### 1) 提交订单

```java
@Transactional
@Override
public SubmitOrderResponseVo submitOrder(OrderSubmitVo submitVo) {

    //提交订单的业务处理。。。
    
    //发送消息到订单延迟队列，判断过期订单
    rabbitTemplate.convertAndSend("order-event-exchange","order.create.order",order.getOrder());

               
}
```

##### 2) 监听队列

创建订单的消息会进入延迟队列，最终发送至队列`order.release.order.queue`，因此我们对该队列进行监听，进行订单的关闭

```java
@Component
@RabbitListener(queues = {"order.release.order.queue"})
public class OrderCloseListener {

    @Autowired
    private OrderService orderService;

    @RabbitHandler
    public void listener(OrderEntity orderEntity, Message message, Channel channel) throws IOException {
        System.out.println("收到过期的订单信息，准备关闭订单" + orderEntity.getOrderSn());
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            orderService.closeOrder(orderEntity);
            channel.basicAck(deliveryTag,false);
        } catch (Exception e){
            channel.basicReject(deliveryTag,true);
        }

    }
}
```

##### 3) 关闭订单

* 由于要保证幂等性，因此要查询最新的订单状态判断是否需要关单
* 关闭订单后也需要解锁库存，因此发送消息进行库存、会员服务对应的解锁

```java
@Override
public void closeOrder(OrderEntity orderEntity) {
    //因为消息发送过来的订单已经是很久前的了，中间可能被改动，因此要查询最新的订单
    OrderEntity newOrderEntity = this.getById(orderEntity.getId());
    //如果订单还处于新创建的状态，说明超时未支付，进行关单
    if (newOrderEntity.getStatus() == OrderStatusEnum.CREATE_NEW.getCode()) {
        OrderEntity updateOrder = new OrderEntity();
        updateOrder.setId(newOrderEntity.getId());
        updateOrder.setStatus(OrderStatusEnum.CANCLED.getCode());
        this.updateById(updateOrder);

        //关单后发送消息通知其他服务进行关单相关的操作，如解锁库存
        OrderTo orderTo = new OrderTo();
        BeanUtils.copyProperties(newOrderEntity,orderTo);
        rabbitTemplate.convertAndSend("order-event-exchange", "order.release.other",orderTo);
    }
}
```

##### 4) 解锁库存

```java
@Slf4j
@Component
@RabbitListener(queues = {"stock.release.stock.queue"})
public class StockReleaseListener {

    @Autowired
    private WareSkuService wareSkuService;

    @RabbitHandler
    public void handleStockLockedRelease(StockLockedTo stockLockedTo, Message message, Channel channel) throws IOException {
        log.info("************************收到库存解锁的消息********************************");
        try {
            wareSkuService.unlock(stockLockedTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }

    @RabbitHandler
    public void handleStockLockedRelease(OrderTo orderTo, Message message, Channel channel) throws IOException {
        log.info("************************从订单模块收到库存解锁的消息********************************");
        try {
            wareSkuService.unlock(orderTo);
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (Exception e) {
            channel.basicReject(message.getMessageProperties().getDeliveryTag(),true);
        }
    }
}
```

```java
@Override
public void unlock(OrderTo orderTo) {
    //为防止重复解锁，需要重新查询工作单
    String orderSn = orderTo.getOrderSn();
    WareOrderTaskEntity taskEntity = wareOrderTaskService.getBaseMapper().selectOne((new QueryWrapper<WareOrderTaskEntity>().eq("order_sn", orderSn)));
    //查询出当前订单相关的且处于锁定状态的工作单详情
    List<WareOrderTaskDetailEntity> lockDetails = wareOrderTaskDetailService.list(new QueryWrapper<WareOrderTaskDetailEntity>().eq("task_id", taskEntity.getId()).eq("lock_status", WareTaskStatusEnum.Locked.getCode()));
    for (WareOrderTaskDetailEntity lockDetail : lockDetails) {
        unlockStock(lockDetail.getSkuId(),lockDetail.getSkuNum(),lockDetail.getWareId(),lockDetail.getId());
    }
}
```



# 17、分布式事务

### 17.1 为什么要有分布式事务

分布式系统经常出现的异常

机器宕机、网络异常、消息丢失、消息乱序、不可靠的TCP、存储数据丢失....

![image-20201119170602679](image-20201119170602679.png)



分布式事务是企业中集成的一个难点，也是每一个分布式系统架构中都会涉及到的一个东西，特别是在微服务架构中，几乎是无法避免的

### 17.2 CAP 定理与 BASE 理论

#### 1、CAP 定理

CAP 原则又称 CAP 定理指的是在一个分布式系统中

- **一致性（Consistency）**
  - 在分布式系统中所有数据备份，在同一时刻是否是同样的值，（等同于所有节点访问同一份最新数据的副本）
- **可用性（Avaliability）**
  - 在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求，（对数据更新具备高可用性）
- **分区容错性（Partition tolerance）**
  - 大多数分布式系统都分布在多个子网络，每个自网络叫做一个区（partition）。分区容错性的意思是，区间通信可能失败，比如，一台服务器放在中国另一台服务器放在美国，这就是两个区，它们之间可能无法通信

CAP 的原则是，这三个要素最多只能满足两个点，**不可能三者兼顾**

![image-20201119194354467](image-20201119194354467.png)



一般来说，分区容错无法避免，因此我们可以认为 CAP 的 P 总是成立，CAP 定理告诉我们 剩下的 C 和 A 无法同时做到

分布式实现一致性的 raft 算法

http://thesecretlivesofdata.com/raft/



#### 2、面临的问

对于大多数互联网应用的场景、主机众多、部署分散，而且集群规模越来越大，所以节点故障，网络故障是常态，而且要保证服务可用性达到99.999%，即保证P 和 A,舍弃C

#### 3、BASE 理论

是对CAP的延申，思想即是无法做到强一致性（CAP的一致性就是强一致性），但可以采用适当的弱一致性，即**最终一致性**

BASE 是指

- 基本可用（Basically Avaliable）
  - 基本可用是指分布式系统中在出现故障的时候，允许损失部分可用性（列入响应时间，功能上的可用性）允许损失部分可用性。需要注意的是基本可用不等价于系统不可用
  - 响应时间上的损失，正常情况下搜索引擎需要在0.5秒之内返回给用户相应的查询结果，但由于出现故障（比如系统部分机房断电断网故障），查询的结果响应时间增加到了1~2秒
  - 功能上的损失，购物网站双十一购物高峰，为了保证系统的稳定性，部分消费者会被引入到一个降级页面
- **软状态（Soft State）**
  - 软状态是指允许 系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据会有 多个副本，允许不同副本同步的延时就是软状态的体现。mysglreplication的异步复制也是一种体现。
- **最终一致性( Eventual Consistency)**
  - 最终致性是指系统中的所有数据副本经过一定时间后，最终能够达到一 致的状态。弱一致性和强一致性相反，最终-致性是弱一致性的一种特殊情况。



#### 4、强一致性、弱一致性、最终一致性

从客户端角度，多进程并发访问时，更新过的数据在不同进程如何获取的不同策略，决定了不同的一致性。对于关系型数据库，要求更新过的数据能被后续的访问都能看到，这是强一致性。如果能容忍后续的部分或者全部访问不到，则是弱一致性。如果经过一段时间后要求能访问到更新后的数据，则是最终一致性

### 17.3 分布式事务的几种方案

#### 1、2PC 模式

数据库支持的 2PC[2 phase commit 二阶提交]，又叫 XA Transactions

MySQL 从5.5版本开始支持，Sql Server 2005 开始支持，oracle7 开始支持

其中，XA 是一个两阶段提交协议，该协议分为两个阶段：

第一阶段：事务协调器要求每个涉及到事务的数据库预提交（precommit）此操作，并反应是否可以提交

第二阶段：事务协调器要求每个数据库提交数据

其中，如果有任何一个数据库否认这次提交，那么所有数据库都会要求回滚他们在此事务中的那部分信息

![image-20201120090516086](image-20201120090516086.png)

- XA协议比较简单，而且一旦商业数据库实现了XA协议，使用分布式事务的成本也比较低。
- XA性能不理想，特别是在交易下单链路，往往并发量很高，XA无法满足高并发场景
- XA目前在商业数据库支持的比较理想，在mysq!数据库中支持的不太理想，mysgl的
- XA实现，没有记录prepare阶段日志，主备切换回导致主库与备库数据不一致。
- 许多nosgI也没有支持XA，这让XA的应用场景变得非常狭隘。
- 也有3PC,引入了超时机制(无论协调者还是参与者，在向对方发送请求后，若长时间未收到回应则做出相应处理)

#### 2、柔性事务 - TCC 事务

刚性事务:遵循ACID原则，强一致性。
柔性事务:遵循BASE理论，最终一致性;
与刚性事务不同，柔性事务允许一定时间内，不同节点的数据不一致，但要求最终一致。

![image-20201120090923734](image-20201120090923734.png)

一阶段 prepare 行为:调用自定义的 prepare 逻辑。
二阶段 commit 行为:调用自定义的 commit 逻辑。
二阶段 rollback行为:调用自定义的 rollback 逻辑。
所谓TCC模式，是指支持把自定义的分支事务纳入到全局事务的管理中。

#### 3、柔性事务 - 最大努力通知型方案

按规律进行通知，**不保证数据定能通知成功， 但会提供可查询操作接口进行核对**。这种方案主要用在与第三方系统通讯时，比如:调用微信或支付宝支付后的支付结果通知。这种方案也是结合MQ进行实现，例如:通过MQ发送http请求，设置最大通知次数。达到通知次数后即不再通知。

案例:银行通知、商户通知等(各大交易业务平台间的商户通知:多次通知、查询校对、对账文件)，支付宝的支付成功异步回调

#### 4、柔性事务 - 可靠信息 + 最终一致性方案（异步通知型）

实现:业务处理服务在业务事务提交之前，向实时消息服务请求发送消息，实时消息服务只记录消息数据，而不是真正的发送。业务处理服务在业务事务提交之后，向实时消息服务确认发送。只有在得到确认发送指令后，实时消息服务才会真正发送。





# 18、支付宝支付

### 1、进入蚂蚁金服开放平台

https://open.alipay.com/platform/home.htm

2、下载支付宝官方Demo，进行配置和测试

文档地址：

https://openhome.alipay.com/docCenter/docCenter.htm 创建应用对应文档

https://opendocs.alipay.com/open/200/105304 网页移动应用文档

https://opendocs.alipay.com/open/54/cyz7do 相关Demo

![image-20201122123349754](image-20201122123349754.png)

![image-20201122114005616](image-20201122114005616.png)密码

创建应用

![image-20201122113922427](image-20201122113922427.png)

### 3、使用沙箱进行测试

https://openhome.alipay.com/platform/appDaily.htm?tab=info

![image-20201122125716179](image-20201122125716179.png)

### 4、什么是公钥、私钥、加密、签名和验签?

1、公钥私钥
公钥和私钥是一一个相对概念

它们的公私性是相对于生成者来说的。

一对密钥生成后，保存在生成者手里的就是私钥，生成者发布出去大家用的就是公钥

![image-20201122174141878](image-20201122174141878.png)

### 5、支付宝支付流程

https://opendocs.alipay.com/open/270/105898

#### 1、引导用户进入到支付宝页面

##### 1、pom.xml

```xml
<!--支付宝模块-->
 <dependency>
            <groupId>com.alipay.sdk</groupId>
            <artifactId>alipay-sdk-java</artifactId>
            <version>4.9.28.ALL</version>
        </dependency>
```

##### 2、将素材提供的文件引入到项目中

**AlipayTemplate、PayVo、PaySyncVo**

###### AlipayTemplate

```java
package com.atguigu.gulimall.order.config;

import com.alipay.api.AlipayApiException;
import com.alipay.api.AlipayClient;
import com.alipay.api.DefaultAlipayClient;
import com.alipay.api.request.AlipayTradePagePayRequest;
import com.atguigu.gulimall.order.vo.PayVo;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@ConfigurationProperties(prefix = "alipay")
@Component
@Data
public class AlipayTemplate {

    //在支付宝创建的应用的id
    private   String app_id = "2016092200568607";

    // 商户私钥，您的PKCS8格式RSA2私钥
    private  String merchant_private_key = "XXX";
    // 支付宝公钥,查看地址：https://openhome.alipay.com/platform/keyManage.htm 对应APPID下的支付宝公钥。
    private  String alipay_public_key = "XXX";
    // 服务器[异步通知]页面路径  需http://格式的完整路径，不能加?id=123这类自定义参数，必须外网可以正常访问
    // 支付宝会悄悄的给我们发送一个请求，告诉我们支付成功的信息
    private  String notify_url = "http://member.gulimall.com/memberOrder.html";

    // 页面跳转同步通知页面路径 需http://格式的完整路径，不能加?id=123这类自定义参数，必须外网可以正常访问
    //同步通知，支付成功，一般跳转到成功页
    private  String return_url = "http://member.gulimall.com/memberOrder.html";

    // 签名方式
    private  String sign_type = "RSA2";

    // 字符编码格式
    private  String charset = "utf-8";
    // 订单超时时间，到达超时时间后自动关闭订单不能再继续支付
    private String timeout = "30m";

    // 支付宝网关； https://openapi.alipaydev.com/gateway.do
    private  String gatewayUrl = "https://openapi.alipaydev.com/gateway.do";

    public  String pay(PayVo vo) throws AlipayApiException {

        //AlipayClient alipayClient = new DefaultAlipayClient(AlipayTemplate.gatewayUrl, AlipayTemplate.app_id, AlipayTemplate.merchant_private_key, "json", AlipayTemplate.charset, AlipayTemplate.alipay_public_key, AlipayTemplate.sign_type);
        //1、根据支付宝的配置生成一个支付客户端
        AlipayClient alipayClient = new DefaultAlipayClient(gatewayUrl,
                app_id, merchant_private_key, "json",
                charset, alipay_public_key, sign_type);

        //2、创建一个支付请求 //设置请求参数
        AlipayTradePagePayRequest alipayRequest = new AlipayTradePagePayRequest();
        alipayRequest.setReturnUrl(return_url);
        alipayRequest.setNotifyUrl(notify_url);

        //商户订单号，商户网站订单系统中唯一订单号，必填
        String out_trade_no = vo.getOut_trade_no();
        //付款金额，必填
        String total_amount = vo.getTotal_amount();
        //订单名称，必填
        String subject = vo.getSubject();
        //商品描述，可空
        String body = vo.getBody();

        // timeout_express 订单支付超时时间
        alipayRequest.setBizContent("{\"out_trade_no\":\""+ out_trade_no +"\","
                + "\"total_amount\":\""+ total_amount +"\","
                + "\"subject\":\""+ subject +"\","
                + "\"body\":\""+ body +"\","
                + "\"timeout_express\":\"" + timeout + "\","
                + "\"product_code\":\"FAST_INSTANT_TRADE_PAY\"}");

        String result = alipayClient.pageExecute(alipayRequest).getBody();

        //会收到支付宝的响应，响应的是一个页面，只要浏览器显示这个页面，就会自动来到支付宝的收银台页面
        System.out.println("支付宝的响应："+result);

        return result;
    }
}
```

###### PayVo

```java
package com.atguigu.gulimall.order.vo;

import lombok.Data;

/**
 * 支付使用Vo
 */
@Data
public class PayVo {
    private String out_trade_no; // 商户订单号 必填
    private String subject; // 订单名称 必填
    private String total_amount;  // 付款金额 必填
    private String body; // 商品描述 可空
}

```

###### payAsyncVo

```java
package com.atguigu.gulimall.order.vo;

import lombok.Data;
import lombok.ToString;

import java.util.Date;

/**
 * 支付宝回调参数Vo
 */
@ToString
@Data
public class PayAsyncVo {
    private String gmt_create;
    private String charset;
    private String gmt_payment;
    private Date notify_time;
    private String subject;
    private String sign;
    private String buyer_id;//支付者的id
    private String body;//订单的信息
    private String invoice_amount;//支付金额
    private String version;
    private String notify_id;//通知id
    private String fund_bill_list;
    private String notify_type;//通知类型； trade_status_sync
    private String out_trade_no;//订单号
    private String total_amount;//支付的总额
    private String trade_status;//交易状态  TRADE_SUCCESS
    private String trade_no;//流水号
    private String auth_app_id;//
    private String receipt_amount;//商家收到的款
    private String point_amount;//
    private String app_id;//应用id
    private String buyer_pay_amount;//最终支付的金额
    private String sign_type;//签名类型
    private String seller_id;//商家的id
}
```

##### 3、编写业务跳转到支付页面

```java
/**
 * @author gcq
 * @Create 2021-01-08
 */
@Controller
public class PayWebController {

    @Autowired
    AlipayTemplate alipayTemplate;

    @Autowired
    OrderService orderService;

    /**
     * 1、跳转到支付页面
     * 2、用户支付成功后，我们要跳转到用户的订单列表页
     * produces 明确方法会返回什么类型，这里返回的是html页面
     * @param orderSn
     * @return
     * @throws AlipayApiException
     */
    @ResponseBody
    @GetMapping(value = "/payOrder",produces = "text/html")
    public String payOrder(@RequestParam("orderSn") String orderSn) throws AlipayApiException {
//        PayVo payVo = new PayVo();
//        payVo.setBody(); // 商品描述
//        payVo.setSubject(); //订单名称
//        payVo.setOut_trade_no(); // 订单号
//        payVo.setTotal_amount(); //总金额
        PayVo payvo = orderService.payOrder(orderSn);
        // 将返回支付宝的支付页面，需要将这个页面进行显示
        String pay = alipayTemplate.pay(payvo);
        System.out.println(pay);
        return pay;
    }
}
```

###### Service

```java
 /**
     * 计算商品支付需要的信息
     * @param orderSn
     * @return
     */
    @Override
    public PayVo payOrder(String orderSn) {
        PayVo payVo = new PayVo();
        OrderEntity orderEntity = this.getOrderByOrderSn(orderSn); // 根据订单号查询到商品
        // 数据库中付款金额小数有4位，但是支付宝只接受2位，所以向上取整两位数
        BigDecimal decimal = orderEntity.getPayAmount().setScale(2, BigDecimal.ROUND_UP);
        payVo.setTotal_amount(decimal.toString());
        // 商户订单号
        payVo.setOut_trade_no(orderSn);
        // 查询出订单项，用来设置商品的描述和商品名称
        List<OrderItemEntity> itemEntities = orderItemService.list(new QueryWrapper<OrderItemEntity>()
                .eq("order_sn", orderSn));
        OrderItemEntity itemEntity = itemEntities.get(0);
        // 订单名称使用商品项的名字
        payVo.setSubject(itemEntity.getSkuName());
        // 商品的描述使用商品项的属性
        payVo.setBody(itemEntity.getSkuAttrsVals());
        return payVo;
    }
```

最后生成页面

![image-20210108162512457](image-20210108162512457.png)

#### 2、用户点击付款

![image-20210108162541917](image-20210108162541917.png)

#### 3、付款成功后跳转到成功页面

<span style="color:red">跳转的页面是根据AlipayTemplate定义的回调地址来进行跳转</span>

- notify_url：**支付成功异步回调，返回支付成功相关的信息**
- return_url：**同步通知，支付成功后页面跳转到那里**

```java
   // 服务器[异步通知]页面路径  需http://格式的完整路径，不能加?id=123这类自定义参数，必须外网可以正常访问
    // 支付宝会悄悄的给我们发送一个请求，告诉我们支付成功的信息
    private  String notify_url = "http://member.gulimall.com/memberOrder.html";

// 页面跳转同步通知页面路径 需http://格式的完整路径，不能加?id=123这类自定义参数，必须外网可以正常访问
    //同步通知，支付成功，一般跳转到成功页
    private  String return_url = "http://member.gulimall.com/memberOrder.html";
```

> 这里跳转到的**会员服务的订单页面**，需要自己处理请求,**已在14.4.4.3节完成该功能**

![image-20210108162610017](image-20210108162610017.png)

##### 1、支付成功后异步回调接口处理

<span style="color:blue;font-size:18px">需要有服务器或配置了内网穿透才能接收到该方法</span>

```java
/**
 * 支付宝成功异步回调
 * @author gcq
 * @Create 2021-01-08
 */
@RestController
public class OrderPayedListener {

    @Autowired
    AlipayTemplate alipayTemplate;

    @Autowired
    OrderService orderService;

    /**
     * 支付宝异步通知回调接口,需要拥有内网穿透或服务器
     * @param request
     * @return
     */
    @PostMapping("/payed/notify")
    public String handleAlipayed(PayAsyncVo vo, HttpServletRequest request) throws UnsupportedEncodingException, AlipayApiException {
        /**
         * 重要一步验签名
         *  防止别人通过postman给我们发送一个请求，告诉我们请求成功，为了防止这种效果通过验签
         */
        Map<String,String> params = new HashMap<String,String>();
        Map<String,String[]> requestParams = request.getParameterMap();
        for (Iterator<String> iter = requestParams.keySet().iterator(); iter.hasNext();) {
            String name = (String) iter.next();
            String[] values = (String[]) requestParams.get(name);
            String valueStr = "";
            for (int i = 0; i < values.length; i++) {
                valueStr = (i == values.length - 1) ? valueStr + values[i]
                        : valueStr + values[i] + ",";
            }
            //乱码解决，这段代码在出现乱码时使用
            valueStr = new String(valueStr.getBytes("ISO-8859-1"), "utf-8");
            params.put(name, valueStr);
        }
        // 支付宝验签 防止恶意提交
        boolean signVerified = AlipaySignature.rsaCheckV1(params
                , alipayTemplate.getAlipay_public_key()
                , alipayTemplate.getCharset()
                , alipayTemplate.getSign_type());
        if (signVerified) {
            String result = orderService.handleAlipayed(vo);
            return result;
        } else {
            return "error";
        }
    }
}
```

Service

```java
 @Override
    public String handleAlipayed(PayAsyncVo vo) {
        // 保存交易流水信息,每个月和支付宝进行对账
        PaymentInfoEntity infoEntity = new PaymentInfoEntity();
        // 设置核心字段
        infoEntity.setOrderSn(vo.getOut_trade_no());
        infoEntity.setAlipayTradeNo(vo.getTrade_no());
        infoEntity.setPaymentStatus(vo.getTrade_status());
        infoEntity.setCallbackTime(vo.getNotify_time());
        // 保存订单流水
        paymentInfoService.save(infoEntity);
        /**
         * 支付宝交易状态说明
         *      https://opendocs.alipay.com/open/270/105902
         */
        // TRADE_FINISHED 交易结束、不可退款
        // TRADE_SUCCESS 交易支付成功
        if (vo.getTrade_status().equals("TRADE_SUCCESS") || vo.getTrade_status().equals("TRADE_FINISHED")) {
            String outTradeNo = vo.getOut_trade_no();
            // 支付宝回调成功后，更改订单的支付状态位已支付
            this.baseMapper.updateOrderStatus(outTradeNo,OrderStatusEnum.PAYED.getCode());
        }
        return "success";
    }

```



### 6、内网穿透

#### 1、简介

内网穿透功能可以允许我们使用外网的网址来访问主机
正常的外网需要访问我们项目的流程是:

1、买服务器并且有公网固定IP

2、买域名映射到服务器的IP

3、域名需要进行备案和审核

#### 2、使用场景

1、开发测试(微信、支付宝)

2、智慧互联

3、远程控制

4、私有云

#### 3、内网穿透常用的软件

1、natapp:https://natapp.cn/
2、续断: www.zhexi.tech
3、花生壳: https://www.oray.com/



# 19、秒杀

### 19.1 秒杀业务

秒杀具有瞬间高并发的特点，针对这一特点，必须要做到限流 + 异步 + 缓存（页面静态化） + **独立部署**

限流方式：

1、前端限流，一些高并发的网站直接在前端页面开始限流，列如：小米的验证码

2、nginx限流，直接负载部分请求到错误的静态页面，令牌算法、漏斗算法

3、网关限流、限流的过滤器

4、代码中使用分布式信号量

5、rabbitmq限流（能者多劳 channel.basicQos(1)）保证发挥服务器的所用性能

### 19.2 秒杀流程

参考京东秒杀流程

见秒杀流程图

完整流程图地址：https://www.processon.com/view/link/5fbda8c35653bb1d54f7077b

#### 秒杀方式一

![image-20210108164514905](image-20210108164514905.png)

#### 秒杀方式2

![image-20210108164555957](image-20210108164555957.png)

### 19.3 限流



### 19.4 秒杀核心业务




# 20、定时任务

### 20.1 cron表达式

语法：秒 分 时 日 月 周 年（Spring不支持）

http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/crontrigger.html

![image-20201123095003586](image-20201123095003586.png)

特殊字符：

- `*`（ *“所有值”*）-用于选择字段中的所有值。例如，分钟字段中的“ ***** ”表示*“每分钟”*。
- `？`（ *“无特定值”*）-当您需要在允许使用字符的两个字段之一中指定某个内容而在另一个不允许的字段中指定某些内容时很有用。例如，如果我希望在某个月的某个特定日期（例如，第10天）触发触发器，但不在乎一周中的哪一天发生，则将“ 10”设置为月字段，以及“？” 在“星期几”字段中。请参阅下面的示例以进行澄清。
- `--`用于指定范围。例如，小时字段中的“ 10-12”表示*“小时10、11和12”*。
- `，` -用于指定其他值。例如，“星期几”字段中的“ MON，WED，FRI”表示*“星期一，星期三和星期五的日子”*。
- `/` -用于指定增量。例如，秒字段中的“ 0/15”表示*“秒*0、15、30*和45”*。秒字段中的“ 5/15”表示*“秒*5、20、35*和50”*。您还可以在“ **”字符**后指定“ /” **-在这种情况下，“** ”等效于在“ /”之前使用“ 0”。每月日期字段中的“ 1/3”表示*“从该月的第一天开始每三天触发一次”*。
- `L`（ *“最后一个”*）-在允许使用的两个字段中都有不同的含义。例如，“月”字段中的值“ L”表示*“*月*的最后一天”，即*非January年的1月31日，2月28日。如果单独用于星期几字段，则仅表示“ 7”或“ SAT”。但是，如果在星期几字段中使用另一个值，则表示*“该月的最后xxx天”*，例如“ 6L”表示*“该月的最后一个星期五”*。您还可以指定与该月最后一天的偏移量，例如“ L-3”，这表示日历月的倒数第三天。 *使用“ L”选项时，不要指定列表或值的范围很重要，因为这样会导致混淆/意外结果。*
- `W`（ *“工作日”*）-用于指定最接近给定日期的工作日（星期一至星期五）。例如，如果您要指定“ 15W”作为“月日”字段的值，则含义是： *“离月15日最近的工作日”*。因此，如果15号是星期六，则触发器将在14号星期五触发。如果15日是星期日，则触发器将在16日星期一触发。如果15号是星期二，那么它将在15号星期二触发。但是，如果您将“ 1W”指定为月份的值，并且第一个是星期六，则触发器将在第3个星期一触发，因为它不会“跳过”一个月日的边界。仅当月份中的某天是一天，而不是范围或天数列表时，才可以指定“ W”字符。

> “ L”和“ W”字符也可以在“月日”字段中组合以产生“ LW”，这表示*“每月的最后一个工作日” *。

- `＃` -用于指定每月的第“ XXX”天。例如，“星期几”字段中的值“ 6＃3”表示*“该月的第三个星期五”*（第6天=星期五，“＃3” =该*月的第三个星期五*）。其他示例：“ 2＃1” =该月的第一个星期一，“ 4＃5” =该月的第五个星期三。请注意，如果您指定“＃5”，并且该月的指定星期几中没有5个，则该月将不会触发。

> 法定字符以及月份和星期几的名称不区分大小写。`MON` 与`mon`相同。



### 20.2 cron 示例

阅读技巧：秒 分 时 日 月 周

![image-20201123095544841](image-20201123095544841.png)

使用谷歌翻译后中文意思是这样的

![image-20201123095718609](image-20201123095718609.png)

### 20.3 SpringBoot整合定时任务

定时任务相关注解:

```java
@EnableAsync // 启用Spring异步任务支持
@EnableScheduling // 启用Spring的计划任务执行功能
@Async 异步
@Scheduled(cron = "* * * * * ?")
```

代码：

```java

/**
 * @author gcq
 * @Create 2020-11-23
 *
 * 定时任务
 *      1、@EnableScheduling 开启定时任务
 *      2、Scheduled 开启一个定时任务
 *      3、自动配置类 TaskSchedulingAutoConfiguration 属性绑定在TaskExecutionProperties
 *
 * 异步任务
 *      1、@EnableAsync 开启异步任务功能
 *      2、@Async 给希望异步执行的方法上标注
 *      3、自动配置类 TaskExecutionAutoConfiguration
 *
 */
@Slf4j
@Component
@EnableAsync // 启用Spring异步任务支持
@EnableScheduling // 启用Spring的计划任务执行功能
public class HelloSchedule {

    /**
     * 1、Spring中6位组成，不允许第7位的年
     * 2、在周几的位置，1-7代表周一到周日：MON-SUN
     * 3、定时任务应该阻塞，默认是阻塞的
     *      1、可以让业务以异步的方式运行，自己提交到线程池
     *          CompletableFuture.runAsync(() -> {
     *              xxxService.hello();
     *          })
     *      2、支持定时任务线程池，设置 TaskSchedulingProperties
     *          spring.task.scheduling.pool.size=5
     *      3、让定时任务异步执行
     *          异步执行
     *     解决：使用异步 + 定时任务来完成定时任务不阻塞的功能
     */
//    @Async 异步
//    @Scheduled(cron = "* * * * * ?")
//    public void hello() {
//        log.info("hello...");
//    }
}
```

### 20.4 分布式定时任务



### 20.5定时任务的问题



### 20.6 扩展 - 分布式调整

# 21、秒杀（高并发）系统关注的问题

### 21.1 服务单一职责
秒杀服务即使自己扛不住压力，挂掉，不要影响别人
### 21.2 秒杀链接加密
防止恶意攻击模拟秒杀请求，1000次/s攻击
防止连接暴露，自己工作人员，提前秒杀商品

### 21.3 库存预热 + 快速扣减 

提前把数据加载缓存中，

### 21.4 动静分离

nginx做好动静分离，保证秒杀和商品详情页的动态请求才打到后端的服务集群

使用 CDN 网络，分担服务器压力

### 21.5 恶意请求拦截

识别非法攻击的请求并进行拦截	，网管层

### 21.6 流量错峰

使用各种手段，将流量分担到更大宽度的时间点，比如验证码，加入购物车

### 21.7 限流 & 熔断 & 降级

前端限流 + 后端限流

限制次数，限制总量，快速失败降级运行，熔断隔离防止雪崩

### 21.8 队列消峰

1万个请求，每个1000件被秒杀，双11所有秒杀成功的请求，进入队列，慢慢创建订单，扣减库存即可