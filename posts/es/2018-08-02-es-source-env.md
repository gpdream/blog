---
title: "es 6源码debug环境搭建"
date: "2018-08-02T06:33:33.000Z"
tags:
  - "es"
source_url: "https://gpdream.github.io/2018/08/02/es/es-source-env/"
recovery_source: "html"
---
# es 6源码debug环境搭建

es 6.3的源码环境搭建，并使用debug方式运行

### es 源码环境搭建

1. 安装java 10，es 6必需
2. 设置java_home指向java10地址
3. 下载源码`git clone https://github.com/elastic/elasticsearch.git`
4. 进入源码目录，运行`./gradlew idea` 不需要特意安装gradle,gradlew会把这些东西都准备好
5. 使用idea File->New Project From Existing Sources指向源码目录，选择Import project from external model->Gradle，启用 Use auto-import 。 gradle jvm home也要选择java 10的地址。第一次build要较长的时间，等等就好了。

### debug模式启动

1. 在源码根目录运行`./gradlew run --debug-jvm`，运行成功后会提示`[elasticsearch] Listening for transport dt_socket at address: 8000` 这是在等idea来attach 8000端口
2. 使用Idea attach jvm进程,edit configuration -> add remote, 端口指向8000
3. debug启动。可以在org.elasticsearch.bootstrap.Elasticsearch的main方法中打上断点确定是否成功debug运行
4. 访问localhost:9200,看是否成功。成功会返回es集群信息
