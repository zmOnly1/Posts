### ES安全

#### 要求

- elasticsearch.yml文件中，network.host被错误配置为0.0.0.0
- 身份认证
  - 鉴定用户是否合法
- 用户鉴权
  - 指定那个用户可以访问哪个索引
- 传输加密
- 日志审计

#### 免费方案

- 设置Nginx反向代理
- 安装免费的Security插件
  - Search Guard - https://search-guard.com
  - ReadOnly REST - https://gihub.com/sscarduzio/elasticsearch-readonlyrest-plugin
- X-Pack的Basic版
  - https://www.elastic.co/what-is/elastic-stack-security

#### Authentication

- 认证体系的集中类型
  - 用户名密码
  - 秘钥或Kerberos票据
- Realms： X-Pack中的认证服务
  - 内置Realms（免费）
    - File/Native(用户名密码保存至Elasticsearch)
  - 外部Realms（收费）
    - LDAP/Active Directory / PKI /SAML /Kerberos

- RBAC，索引级，字段级，集群级
  - User: Authenticated User
  - Role: A named set of permissions
  - Permission: A set of one or more privileges against a secured resource
  - Privilege: A named group of 1 or more actions that user may execute againse a secured resource

#### 实战

1. elasticsearch.yml

   ```
   network.host: 192.168.1.100
   ```

2. 打开认证与授权

   ```
   bin/elasticsearch -E node.name=node0 -E cluster.name=my_cluster -E path.data=node0_data -E http.port=9200 -E xpack.security.enabled=true
   ```

   启动后：``[2021-07-07T15:02:28,974][INFO ][o.e.x.s.s.SecurityStatusChangeListener] [node0] Active license is now [BASIC]; Security is enabled``

3. 创建默认的用户和分组

   ```
   bin/elasticsearch-setup-passwords -E http.port=9200 interactive
   ```

   如果ES还没起起来会报错：``Connection failure to: http://127.0.0.1:9200/_security/_authenticate?pretty failed: Connection refused: connect``

4. 尝试访问ES：`http://localhost:9200/_cat/nodes`， 会提示输入账号密码：

   输入账号：`elastic`, 密码：`changeme`

5. 当集群开启身份认证之后，配置Kibana

   - 修改Kibana配置文件`config/kibana.yml`:

     ```
     elasticsearch.username: "kibana"
     elasticsearch.password: "changeme"
     ```

6. 启动Kibana：`http://localhost:5601`

   - 登录， 账号：`elastic`, 密码： `changeme`

   - 进入Dev Tools， 创建测试数据:

     ```
     GET orders/_search
     
     DELETE orders
     POST orders/_bulk
     {"index": {}}
     {"product": "1", "price":18,"payment":"master", "card":"987654321", "name": "jack"}
     {"index": {}}
     {"product": "2", "price":99,"payment":"visa", "card":"123456789", "name":= "bob"}
     ```

   - 创建一个Role，配置为对某个索引只读权限/创建一个用户，把用户加入Role

     1. 进入`management/Security/Roles`

        ```
        Role name: read_orders
        Run As privileges: kibana_user
        Indices: orders
        Privileges: read
        Kibana All Spaces: read
        ```

     2. 进入`management/Security/User`

        ```
        Username: demo
        Password: changeme
        Roles: read_orders
        ```

     3. 退出当前账号，使用刚才创建的用户`demo`登录：进入Dev Tools

        ```
        #有权限读取
        GET orders/_search
        #会报错，没权限删除
        DELETE orders
        ```

#### 集群加密通讯

##### 为节点创建证书

1. TLS，TLS协议要求Trusted Certificate Authority(CA)欠费的X.509的证书

2. 证书认证的同步级别

   - Certificate - 节点加入需要使用相同CA签发的证书
   - Full Verification - 节点加入集群需要相同CA签发的证书，还需要验证Host name或IP地址
   - No Verification - 任何节点都可以加入，开发环境中用于诊断目的

3. 生成节点证书：

   - `bin/elasticsearch-certutil ca`，两次enter，生成默认文件`elastic-stack-ca.p12`，空密码
   - `bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12`,两次enter，生成默认的证书文件`elastic-certificates.p12`
   - 把`elastic-certificates.p12`拷贝到`config/certs/elastic-certificates.p12`
   - 参考：https://www.elastic.co/guide/en/elasticsearch/reference/7.1/configuring-tls.html

4. 配置节点间通信，修改配置文件`elasticsearch.yml`

   ```
   xpack.security.transport.ssl.enabled: true
   xpack.security.transport.ssl.verification_mode: certificate 
   xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12 
   xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12 
   ```

   也可以通过命令行的方式：启动第一个节点

   ```
   bin/elasticsearch -E node.name=node0 -E cluster.name=my_cluster -E path.data=node0_data -E http.port=9200 -E transport.port=9300 -E cluster.initial_master_nodes=node0 -E discovery.seed_hosts=localhost:9300 -E xpack.security.enabled=true -E xpack.security.transport.ssl.enabled=true -E xpack.security.transport.ssl.verification_mode=certificate -E xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12 -E xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12 
   ```

   启动第二个节点：

   ```
   bin/elasticsearch -E node.name=node1 -E cluster.name=my_cluster -E path.data=node1_data -E http.port=9201 -E transport.port=9301 -E cluster.initial_master_nodes=node0 -E discovery.seed_hosts=localhost:9300,localhost:9301 -E xpack.security.enabled=true -E xpack.security.transport.ssl.enabled=true -E xpack.security.transport.ssl.verification_mode=certificate -E xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12 -E xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12
   ```

   尝试启动第三个节点：没有配置证书，加入集群将会失败

   ```
   bin/elasticsearch -E node.name=node2 -E cluster.name=my_cluster -E path.data=node2_data -E http.port=9202 -E transport.port=9302 -E cluster.initial_master_nodes=node0 -E discovery.seed_hosts=localhost:9300,localhost:9301,localhost:9302 -E xpack.security.enabled=true -E xpack.security.transport.ssl.enabled=true -E xpack.security.transport.ssl.verification_mode=certificate 
   ```

#### 配置HTTPS

1. 修改`elasticsearch.yml`

   ```
   xpack.security.http.ssl.enabled: true
   xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12 
   xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12 
   ```

2. 命令行启动：

   ```
   bin/elasticsearch -E node.name=node0 -E cluster.name=my_cluster -E path.data=node0_data -E http.port=9200 -E xpack.security.enabled=true -E xpack.security.transport.ssl.enabled=true -E xpack.security.transport.ssl.verification_mode=certificate -E xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12 -E xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12 -E xpack.security.http.ssl.enabled=true -E xpack.security.http.ssl.keystore.path=certs/elastic-certificates.p12 -E xpack.security.http.ssl.truststore.path=certs/elastic-certificates.p12 
   ```

   - 使用http访问将失败`http://localhost:9200/_cat/nodes`,

   - 需要使用https访问：`https://localhost:9200/_cat/nodes`
   - Kibana也将连接失败

3. 配置Kibana连接ES HTTPS

   ```
   elasticsearch.hosts: ["https://localhost:9200"]
   elasticsearch.ssl.certificateAuthorities: /path/to/your/ca.crt
   ```

4. 将证书导出成pem格式

   ```
   openssl pkcs12 -in elastic-certificates.p12 -cacerts -nokeys -out elastic-ca.pem
   ```

   将文件`elastic-ca.pem`拷贝到`config/certs`目录下

5. 修改kibana的配置文件`config/kibana.yml`

   ```
   elasticsearch.hosts: ["https://localhost:9200"]
   elasticsearch.ssl.certificateAuthorities: [ "D:/ELK/kibana-7.10.2-windows-x86_64/config/certs/elastic-ca.pem" ]
   elasticsearch.ssl.verificationMode: certificate
   ```

6. 重启Kibana：`bin/kibana -c config/kibana.yml -p 5602`

7. 配置HTTPS访问Kibana

   1. 生成ca证书：执行下面命令将会生成一个压缩文件：elastic-stack-ca.zip

   ```
   bin/elasticsearch-certutil ca --pem
   ```

   2. 解压`elastic-stack-ca.zip`，得到两个文件`ca.key`和`ca.crt`，把这两个配置拷贝到`config/certs/`目录下

   3. 修改kibana配置文件`config/kibana.yml`

      ```
      server.ssl.enabled: true
      server.ssl.certificate: config/certs/ca.crt
      server.ssl.key: config/certs/ca.key
      ```

   4. `kibana.yml`

      ```
      elasticsearch.username: "kibana"
      elasticsearch.password: "changeme"
      
      server.ssl.enabled: true
      server.ssl.certificate: config/certs/ca.crt
      server.ssl.key: config/certs/ca.key
      
      elasticsearch.hosts: ["https://localhost:9200"]
      elasticsearch.ssl.certificateAuthorities: [ "D:/ELK/kibana-7.10.2-windows-x86_64/config/certs/elastic-ca.pem" ]
      elasticsearch.ssl.verificationMode: certificate
      ```

      