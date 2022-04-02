



### 实战

```dockerfile
FROM centos
MAINTAINER taozhiming
RUN groupadd solr
RUN useradd solr -g solr
##安装jdk配置jdk环境
COPY ./Docker/jdk-8u311-linux-x64.tar.gz /usr/local
RUN tar -zxvf /usr/local/jdk-8u311-linux-x64.tar.gz -C /usr/local
RUN rm -rf /usr/local/jdk-8u311-linux-x64.tar.gz
ENV JAVA_HOME=/usr/local/jdk1.8.0_311
ENV PATH=$JAVA_HOME/bin:$PATH
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

## 安装solr软件
COPY ./target/solr4totoro-8.5.1.tar.gz /usr/local
RUN tar -zxvf /usr/local/solr4totoro-8.5.1.tar.gz -C /usr/local
RUN rm -rf /usr/local/solr4totoro-8.5.1.tar.gz
RUN chown -R solr.solr /usr/local/solr4totoro-8.5.1/*
RUN mkdir -p data/solr

##删除脚本目录
RUN rm -rf /usr/local/solr4totoro-8.5.1/bin/solr
COPY ./Docker/solr /usr/local/solr4totoro-8.5.1/bin/
RUN chmod +x /usr/local/solr4totoro-8.5.1/bin/*
## 配置solr软件目录
COPY ./Docker/solr_80 /data/solr/solr_80
RUN chown -R solr.solr /data/solr/*
#VOLUME ["/data/solr"]
ENTRYPOINT ["./data/solr/solr_80/bin/solr.sh", "start"]
CMD ["solr4totoro_8.5.1","zk_host","jvm"]
EXPOSE 80


```

