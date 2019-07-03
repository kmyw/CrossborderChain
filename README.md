# 跨境链

------



##1. 数据模型
### 1.1 Bizview业务视图启动流程
####1.1.1 Bizview业务视图系统结构
![此处输入图片的描述][1]
其中Bizview Server是整个系统的枢纽，负责解析Client传来的API请求，并与MySQL或OSS或Blcokchain通信，处理请求，然后将请求结果返还给客户端。在这个结构中，需要配置地方有四处，分别是：MySQL、OSS、Blockchain和Bizview。

####1.1.2 MySQL配置注意事项
1.配置远程连接。
修改/etc/mysql/mysql.conf.d/目录下的mysqld.cnf文件，将其bind-address选项由默认的127.0.0.1改成0.0.0.0，或直接注释掉这个选项。
*默认127.0.0.1表示数据库只接受来自本地访问3306端口的请求。改为0.0.0.0后则表示来自任意地址访问3306端口的请求都会被交予MySQL数据库处理。*

2.配置权限。
运行以下两条命令：
```
mysql>GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
mysql>flush privileges;
*其中:*
*"root" "123456" 是数据库用户名和密码*
*'%' 允许访问数据库的IP地址，%意思是任意IP，也可以指定IP*
*flush privileges 刷新权限信息*
```
####1.1.3 OSS配置注意事项
1.在OSS对象平台上新建bucket,需要将其权限设置为公共读写。
2.同时需要在Bizview业务视图的application.properties配置文件中写入相关选项。具体配置详见1.15 Bizview配置注意事项。

####1.1.4 Blockchain配置注意事项
在跨境链平台上找到存证测试链，下载证书与密钥，最后从平台上得到的关于Blockchain端的配置文件应该有：trust.keystore、key_pkcs8.pem、cert.pem。

####1.1.5 Bizview配置注意事项
Bizview运行目录结构
```
|---bizview
|---runApp.sh//启动命令脚本
|---config
|---trust.keystore//账户私钥
|---key_pkcs8.pem//ssl密钥
|---cert.pem//ssl证书
|---sdk.properties//应用与区块链节点交互通证配置
|---application.properties//应用功能配置
|---content.json//数据模型配置
```
其中：
runApp.sh为docker命令脚本，负责登陆平台的仓库，拉取镜像，将config文件目录挂载到docker中并运行镜像。只需运行无需修改。

trust.keystore、key_pkcs8.pem和cert.pem从平台下载后放到相应的目录下即可。

sdk.properties是Bizview存证业务视图应用与区块链节点交互的通证配置文件，在这份文件中我们可以配置与应用通信的区块链节点，通信的ssl密钥与证书，账户私钥以及通信超时时间等。
```
sdk.properties配置

biz.sdk.primary=47.101.176.248 8080
biz.sdk.backups=106.14.207.29 8080;47.100.233.155 8080;47.101.174.35 8080
biz.sdk.ssl_key=/config/key_pkcs8.pem
biz.sdk.ssl_cert=/config/cert.pem
biz.sdk.ssl_key_password=1234!abcd
biz.sdk.trust_store=/config/trust.keystore
biz.sdk.trust_store_password=mychain
biz.sdk.wait_response_timeout=100000
```

application.properties是Bizview存证业务视图应用的功能配置文件，数据业务逻辑层面的配置文件，在这份文件中主要可以配置MySQL的链接、OSS的链接、指明sdk.properties的路径、指定是否开启数据业务模型定制的功能。
```
application.properties部分配置

######## 数据源 ########
mysql.config.flag=true
spring.datasource.hikari.jdbcUrl=jdbc:mysql://172.17.16.4:3306/antfin?useUnicode=yes&characterEncoding=UTF-8&autoReconnect=true
spring.datasource.hikari.username=root
spring.datasource.hikari.password=TJTJTJ

####oss地址，权限控制##########
spring.oss.flag=true//开启oss功能
spring.oss.accesskeyid=LTAI87tzMlbpbn5g//在OSS对象平中查看
spring.oss.accesskeysecret=XhP1InVyzD7gz253bHxLKCuJYiedKu//在OSS对象平台中查看
spring.oss.endpoint=oss-cn-shanghai.aliyuncs.com//在OSS对象平台查看，注意区分内外网。使用阿里的云服务器在内网，非阿里的云服务器均在外网
spring.oss.bucketname=yuhutest//在OSS对象平台中可查看
baas.url=localhost:8443//不用改动

//是否开启jsonSchema模式（业务模型JSON模式），配置false时，选择业务模型jar序列化模式
json.schema.flag=true

//json.schema.flag=true时，必须配置content.json的路径，请将content.json拷贝到config目录
content=/config

//sdk配置文件路径，请将配置中文件路径改成/config/XXXX，并将配置文件拷贝到config目录
sdk.properties=/config/sdk.properties
```

content.json是当在application.properties中指定开启数据业务模型定制时，就在这份文件中进行初始化定制。具体将会在数据模型定制小节中描述。

### 1.2. 数据模型
当开启数据模型定制功能时，便需要在content.json文件中定义特定的数据模型，其定义结构如下：
```
[
{"category":"0x002e0000",
"categoryName":"Goods_CodeRule",
"description":"商品与码规则",
"properties":[{"field":"GoodsName",
"type":"String",
"comment":"商品名称",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"GoodsID",
"type":"String",
"comment":"商品编号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""}]}

,{"category":"0x002e0001",
"categoryName":"confirmedPurchaseOrder",
"description":"确认采购单",
"properties":[{"field":"PurchaseOrderID",
"type":"String",
"comment":"采购单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Buyer",
"type":"String",
"comment":"采购方",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Seller",
"type":"String",
"comment":"供应商",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ContractDate",
"type":"String",
"comment":"合同日期",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Confirmor",
"type":"String",
"comment":"确认人",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ConfirmDate",
"type":"String",
"comment":"确认时间",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"EORI",
"type":"String",
"comment":"商品海关备案号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"GoodsName",
"type":"String",
"comment":"商品名称",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"GoodsSpecification",
"type":"String",
"comment":"商品规格",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Price",
"type":"String",
"comment":"单价",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"CurrencyValue",
"type":"String",
"comment":"币值",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Amount",
"type":"String",
"comment":"数量",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ValidPeriod",
"type":"String",
"comment":"有效期",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeparturePort",
"type":"String",
"comment":"起运港",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Destination",
"type":"String",
"comment":"目的港",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"TransportMethod",
"type":"String",
"comment":"运输方式",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""}]}

,{"category":"0x002e0002",
"categoryName":"InBondedWarehouse",
"description":"入保税仓库",
"properties":[{"field":"PurchaseID",
"type":"String",
"comment":"采购单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"RKDH",
"type":"String",
"comment":"入库单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Warehouse",
"type":"String",
"comment":"仓库",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"InWarehouseDate",
"type":"String",
"comment":"入库日期",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Confirmor",
"type":"String",
"comment":"确认人",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ConfirmDate",
"type":"String",
"comment":"确认时间",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"EORI",
"type":"String",
"comment":"商品海关备案号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"GoodsName",
"type":"String",
"comment":"商品名称",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Amount",
"type":"String",
"comment":"入库数量",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeclarationID",
"type":"String",
"comment":"报关单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeclarationInspectionID",
"type":"String",
"comment":"报检单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""}]}

,{"category":"0x002e0003",
"categoryName":"OutBondedWarehouse",
"description":"出保税仓库",
"properties":[{"field":"OrderID",
"type":"String",
"comment":"订单编号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ListID",
"type":"String",
"comment":"清单编号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"Subscriber",
"type":"String",
"comment":"订购人",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"SubscribePlatform",
"type":"String",
"comment":"订购平台",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"SubscribeDate",
"type":"String",
"comment":"订购时间",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"PaymentID",
"type":"String",
"comment":"支付单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"PaymentCompany",
"type":"String",
"comment":"支付公司",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"PaymentDate",
"type":"String",
"comment":"支付时间",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ExpressCompany",
"type":"String",
"comment":"快递公司",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"ExpressOrderID",
"type":"String",
"comment":"快递编号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"EORI",
"type":"String",
"comment":"商品海关备案号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeclaredPrice",
"type":"String",
"comment":"申报单价",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"PurchaseAmount",
"type":"String",
"comment":"购买数量",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeclarationID",
"type":"String",
"comment":"报关单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeclarationInspectionID",
"type":"String",
"comment":"报检单号",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeliveryWarehouse",
"type":"String",
"comment":"发货仓库",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""},
{"field":"DeliveryDate",
"type":"String",
"comment":"发货时间",
"isIndex":0,
"length":0,
"allow_null":0,
"regex":""}]}]
```
当运行业务视图时，Bizview Server会将config目录下的content.json文件中的业务模型读入MySQL数据库中，一个业务模型对应数据库中的一张表，表名对应业务模型在定义时的"categoryName"字段。
当Bizview在运行时，可以使用业务视图提供的"新增业务视图"接口新增一个业务模型，新增成功后可以在数据库中看到相应新增模型的表。

```
ubuntu@VM-16-4-ubuntu:~$ curl -X POST "http://localhost:8080/api/category/alertCategory" -H "Content-Type: application/json" -d "{\"category\":\"0x00571239\",\"categoryName\":\"Testing233\",\"description\":\"\",\"properties\":[{\"field\":\"contry\",\"type\":\"String\",\"comment\":\"\",\"isIndex\":0,\"length\":0,\"allow_null\":1,\"regex\":\"\"}]}"
{"errMessage":"","success":true,"responseData":null}
```

```
mysql> show tables;
+-----------------------------+
| Tables_in_antfin            |
+-----------------------------+
| agreement_infoc             |
| testing233                  |
+-----------------------------+
```

## 2. MySQL
### 2.1. MySQL与Blockchain的关系
MySQL中会保存该用户创建的所有业务模型，每个业务模型对应一张表，之后Bizview会将Blockchain中的数据同步到MySQL中，也就是说同一个数据存了两份，一份在Blockchain上，一份在MySQL中。

## 3. OSS
### 3.1. OSS如何与Bizview视图交互
首先使用大文件存储接口将文件上传到OSS上，得到返回数据，即上传文件的key与文件hash。
```
ubuntu@VM-16-4-ubuntu:~$ curl -X POST "http://localhost:8080/api/category/uploadFile" -F file=@/home/ubuntu/info/info.txt
{"errMessage":"","success":true,"responseData":"ossUrlOfBizView:1561514684380/info.txt"}
```
接着使用存证数据上链接口将文件的key上传至链上，同时接口会将文件hash与文件key上传至区块链上。
```
curl -X POST "http://localhost:8080/api/category/saveData" -H "Content-Type: application/json" -d "{\"category\": \"files\",  \"filekey\": \"ossUrlOfBizView:1561514684380/info.txt\"}"
{"errMessage":"","success":true,"responseData":"f9977ab052c2244966ae213beb622807c734bf05652202c40633891d4b4df01f"}
```
之后便可用大文件存储查询接口查询所上传的文件。
大文件存储查询接口将业务模型定义字段作为查询条件，当查询的数据中包含 OSS 存储 key 时，Bizview 服务会校验该文件的 hash 与链上文件 hash 是否一致，下载校验通过之后的文件。

## 4. config配置文件
具体配置及影响如上文所述。

## 5. 系统对外API概述

### 注册品牌与码规则

请求方式

RestFul URL：/api/register/business

HTTP 请求方法：POST

内容类型：JSON

```
curl -X POST "http://localhost:8080/api/register/business" -H "Content-Type: application/json" \
-d "{\"category\": \"Goods_CodeRule\",\"GoodsName\":\"testgood1\",\"CoodsID\":\"0x00038324\"}"
```

### 确认采购单

RestFul URL：/api/confirmOrder

HTTP 请求方法：POST

内容类型：JSON
```
curl -X POST "http://localhost:8080/api/confirmOrder" -H "Content-Type: application/json" \
-d "{\"category\": \"confirmedPurchaseOrder\",\"PurchaseOrderID\":\"0x00038320\",\"Buyer\":\"testbuyer1\",\"Seller\":\"testseller1\",\"ContractDate\":\"2019-07-01\",\"Confirmor\":\"testconfirmor1\",\"ConfirmDate\":\"2019-07-02\",\"EORI\":\"123456\",\"GoodsName\":\"testgood1\",\"GoodsSpecification\":\"big\",\"Price\":\"233\",\"CurrencyValue\":\"56\",\"Amount\":\"2333\",\"ValidPeriod\":\"2\",\"DeparturePort\":\"SH\",\"Destination\":\"TW\",\"TransportMethod\":\"ship\"}"
```

### 入保税仓库

RestFul URL：/api/inbound

HTTP 请求方法：POST

内容类型：JSON
```
curl -X POST "http://localhost:8080/api/inbound" -H "Content-Type: application/json" \
-d "{\"category\":\"InBondedWarehouse\",\"PurchaseID\":\"0x00038320\",\"RKDH\":\"0x00012345\",\"Warehouse\":\"shunfeng\",\"InWarehouseDate\":\"2019-07-01\",\"Confirmor\":\"testconfirmor1\",\"ConfirmDate\":\"2019-07-02\",\"EORI\":\"123456\",\"GoodsName\":\"testgood1\",\"Amount\":\"2333\",\"DeclarationID\":\"1234567\",\"DeclarationInspectionID\":\"7654321\"}"
```

### 出保税仓库

RestFul URL：/api/outbound

HTTP 请求方法：POST

内容类型：JSON
```
curl -X POST "http://localhost:8080/api/outbound" -H "Content-Type: application/json" \
-d "{\"category\":\"OutBondedWarehouse\",\"OrderID\":\"0x12345678\",\"ListID\":\"0x87654321\",\"Subscriber\":\"\"}" 
```

### 查询商品信息

RestFul URL：/api/commodity/check

HTTP 请求方法：POST

内容类型：JSON

```
curl -X POST "http://localhost:8080/api/commodity/check" -H "Content-Type: application/json" -d 
```

### 查询区块链交易信息

RestFul URL：/api/tx/check

HTTP 请求方法：POST

内容类型：JSON
```
curl -X POST "http://localhost:8080/api/tx/check" -H "Content-Type: application/json" -d 
```

[1]: http://processon.com/chart_image/5b937abce4b0d4d65bfc4e8b.png
