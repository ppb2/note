Elasticsearch安全认证

### 一、集群身份认证

在elasticsearch.yml配置文件中加入

```YAML
xpack.security.enabled: true
```

### 二、集群内部安全通信

- 设置当前脚本执行环境（可选）

```JavaScript
export ES_PATH_CONF=/data/elasticsearch/elasticsearch_8015/config
```

- 生成证书

```Shell
bin/elasticsearch-certutil ca
```

- 为集群中的每个节点生成证书和私钥

```Shell
bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```

执行命令后输入需要设置的密码

- 将证书拷贝到elasticsearch的每个节点下面config目录下

```Shell
elastic-certificates.p12
```

- 配置elasticsearch.yml文件

```Plain%20Text
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate

xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

- 如果在创建证书的过程中加了密码，需要将你的密码加入到你的Elasticsearch keystore中去

```Shell
bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

- 对于已有的集群需将elastic-certificates.p12  elasticsearch.keystore拷贝到所有节点中覆盖
- 将elastic-certificates.p12  elasticsearch.keystore  拷贝到totoro4kernel中的webapp/elasticsearch_demo/config文件夹中覆盖，使新增集群可以直接使用

### 三、龙猫平台自动化

使用elasticsearch-keystore add "bootstrap.password"  将上述过程添加的密码设置为集群默认密码

使用expect命令完成交互式设置密码部分

```Shell
#!/usr/bin/expect
spawn /usr/local/elasticsearch-7.10.2/bin/elasticsearch-keystore add "bootstrap.password"
expect "*password:"
send "设置的密码\r"
expect eof
```

若需要修改密码，可执行以下命令

```Shell
curl -H 'Content-Type: application/json' --user username:oldpassword  -XPOST "10.110.13.242:8010/_security/user/username/_password" -d '{"password" : "@lpst2022"}'
```

### 四、客户端添加鉴权

- httpclient方式

```Dart
String user = PropUtil.getInstance().get("es.auth.user");
String password = PropUtil.getInstance().get("es.auth.password");
BASE_AUTHORIZATION = new String(Base64.getEncoder().encode((user + ":" + password).getBytes(StandardCharsets.UTF_8)), StandardCharsets.UTF_8);
header.put("Authorization", "Basic " + BASE_AUTHORIZATION);
```

- client直连方式

```Dart
String esUser = PropUtil.getInstance().get("es.auth.user");
String esPwd = PropUtil.getInstance().get("es.auth.password");
final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(esUser, esPwd));

RestClientBuilder builder = RestClient.builder(httpHosts);
builder.setHttpClientConfigCallback(
        httpClientBuilder -> httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider));
highLevelClient = new RestHighLevelClient(builder);
```