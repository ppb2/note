SOLR安全认证

### 新增用户

```Shell
curl --user solr:SolrRocks http://localhost:8983/solr/admin/authentication -H 'Content-type:application/json' -d '{"set-user": {"tom":"TomIsCool", "harry":"HarrysSecret"}}'
```

### 删除用户

```Shell
curl --user solr:SolrRocks http://localhost:8983/solr/admin/authentication -H 'Content-type:application/json' -d  '{"delete-user": ["tom", "harry"]}'
```



### solrj使用

```Java
SolrRequest req ;//create a new request object
req.setBasicAuthCredentials(userName, password);
solrClient.request(req);
```

### 手动操作

1. 定义安全配置
2. 第3行指定认证插件BasicAuthPlugin
3. 第5行 指定用户名为solr，密码为xxx
4. 第11行 设置管理员账户，可允许修改安全配置
5. 第13行 为solr用户赋予admin权限



```JSON
{
"authentication":{ 
   "blockUnknown": true, 
   "class":"solr.BasicAuthPlugin",
   "credentials":{"solr":"IV0EHq1OnNrj6gvRCwvFwTrZ1+z1oBbnQdiVC3otuq0= Ndd7LKvVBAaZIF0QAVi1ekCfAJXr1GGfLtRUXhgrF8c="}, 
   "realm":"My Solr users", 
   "forwardCredentials": false 
},
"authorization":{
   "class":"solr.RuleBasedAuthorizationPlugin",
   "permissions":[{"name":"security-edit",
      "role":"admin"}], 
   "user-role":{"solr":"admin"} 
}}
```

1. 修改solr集群下的[security.json](https://solr.apache.org/guide/8_5/authentication-and-authorization-plugins.html#enable-plugins-with-security-json)节点 立即生效

### 平滑方案

1. 为solr所有项目添加solrj鉴权
   1. ins-totoro4index-platform
   2. ins-totoro4search-platform
   3. ins-totoro4kernel-internal
   4. ins-searchmonitor-internal

```Java
SolrRequest req ;//create a new request object
req.setBasicAuthCredentials(userName, password);
solrClient.request(req);
```

1. 为所有集群设置统一的鉴权方式



### 攻击方式

![img](https://raw.githubusercontent.com/ppb2/note/main/imgs/asynccode)

1. 非通过solr admin界面的方式攻击，图上为url构造的攻击方式
2. 屏蔽solr admin不起作用

### 是否必须加鉴权分析

1. 攻击方式的特点https://mp.weixin.qq.com/s/HMtAz6_unM1PrjfAzfwCUQ（考虑是否屏蔽攻击方式解决）
2. 漏洞不是重点，重点是要加鉴权（需求方要求）

### 性能问题

1. zk的鉴权配置是缓存的不存在每次都从zk拉取鉴权信息
2. 请求获取head信息做编码对比认证信息

结论：性能损耗几乎可以忽略

### 密码变更问题

1. solr支持同时设置多个认证。
2. 认证切换过程中可以新老并行。
3. 待新认证方式完成后，下掉老的认证方式