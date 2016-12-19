---
title: 为filebeat+kafka+logstash添加认证与授权
author: rk700
layout: post
catalog: true
tags:
  - misc
  - how-to
---

我们之前自己开发的几套工具，目前部署在了外网。为了便于监控运营状况，需要对服务日志进行收集展示。

我们采用了目前非常火的ELK，es和kibana之前已经部署好了，所以需要完成的就是日志收集这一环节。具体地，我们采用filebeat+kafka+logstash，即：

- 在需要监控的各个节点上运行filebeat。相比logstash，filebeat更为轻量，也更为专一
- 将filebeat作为kafka的producer，进行缓冲
- 将logstash作为kafka的consumer，对日志进行处理后发送到es

大致类似于下图

![](https://www.elastic.co/assets/blt3922921622a81023/Kafka.png){:.myImage}

出于安全考虑，上述服务的端口绑定的都是内网IP；不仅如此，我们还希望对访问进行控制。

最新的kafka(0.10)支持SASL的GSSAPI和PLAIN这两种认证方式，而filebeat的kafka-output目前只支持SASL/PLAIN，所以我们最后选择了：

- 使用SASL/PLAIN进行身份认证
- 使用kafka自带的Authorizer，配置ACL进行授权管理。

---

## 配置kafka

#### kafka服务端配置SASL

首先，需要修改`config/server.properties`文件，添加以下行:

{% highlight text %}
# 启用的认证模式PLAIN，也可使用GSSAPI
sasl.enabled.mechanisms=PLAIN
# kafka broker之间也需要使用PLAIN方式认证，也可使用GSSAPI
sasl.mechanism.inter.broker.protocol=PLAIN
# 通信为明文。如果需要使用SSL加密通信，则使用SASL_SSL，不过需要配置证书
security.inter.broker.protocol=SASL_PLAINTEXT
{% endhighlight %}

随后，在同一文件中需要设置`listener`，使用明文通信的SASL：(注意区分PLAIN与PLAINTEXT，前者是SASL的一种认证方式，即用户名+密码；后者则指通信过程是明文的，不加密）

{% highlight text %}
listeners=SASL_PLAINTEXT://127.0.0.1:9092
{% endhighlight %}

上述配置告诉了kafka要使用SASL/PLAIN进行身份认证，接下来就需要设置用户名密码。我们创建文件`kafka-server-jaas.conf`如下：

{% highlight text %}
KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="admin_pass"
    user_admin="admin_pass"
    user_producer="producer"
    user_consumer="consumer";
};
{% endhighlight %}

其中设置了3个用户：`admin`，`producer`和`consumer`。`admin`用户是用于kafka broker之间身份认证的，`producer`和`consumer`则分别由filebeat和logstash使用。

实际配置环境时，发现这3行是必须要有的：

{% highlight text %}
username="admin"
password="admin_pass"
user_admin="admin_pass"
{% endhighlight %}

如果删除，kafka就不能正常工作。

最后，就是启动kafka了。文件`kafka-server-jaas.conf`的路径需要设置为`java.security.auth.login.config`的值，我们可以通过环境变量`KAFKA_OPTS`来完成：

{% highlight bash %}
$ KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka_2.11-0.10.1.0/config/kafka-server-jaas.conf" /opt/kafka_2.11-0.10.1.0/bin/kafka-server-start.sh /opt/kafka_2.11-0.10.1.0/config/server.properties
{% endhighlight %}

#### kafka客户端配置SASL

类似于配置服务端，首先我们需要设置相应的properties。例如，对于自带的console-producer，在文件`producer.properties`中添加以下内容：

{% highlight text %}
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
{% endhighlight %}

随后，我们需要将用户名密码信息提供给客户端。新建一个jaas文件，例如`kafka-client-jaas.conf`，其中包含客户端的用户名和密码信息：

{% highlight text %}
KafkaClient {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="producer"
    password="producer";
};
{% endhighlight %}

最后，就可以启动客户端了。以console-producer为例，命令如下：

{% highlight bash %}
$ KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka_2.11-0.10.1.0/config/kafka-client-jaas.conf" ./kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test --producer.config /opt/kafka_2.11-0.10.1.0/config/producer.properties
{% endhighlight %}

就可以向名为test的topic发送信息了。（记得首先创建topic）

而如果我们再次尝试使用console-producer，但不提供用户名密码信息，就会提示无法连接：

{% highlight text %}
$ ./kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test
test123
[2016-12-19 09:45:07,705] WARN Bootstrap broker 127.0.0.1:9092 disconnected (org.apache.kafka.clients.NetworkClient)
^C
{% endhighlight %}

可见此时SASL/PLAIN确实生效了。

以上是为producer配置SASL。对于consumer，在其properties中也设置使用SASL/PLAIN，并将用户名密码通过jaas文件提供给consumer，就可以通过认证接收消息了。具体配置与producer类似，这里就不赘述了。

#### kafka配置ACL

进一步，我们还可以通过ACL对访问权限进行细化。不过在配置前，需要在kafka的server.properties中添加以下内容并重启服务：

{% highlight text %}
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
super.users=User:admin
{% endhighlight %}

`authorizer.class.name`这一项默认是""，我们设置使用kafka自带的`kafka.security.auth.SimpleAclAuthorizer`来启用ACL。而为了正常工作，我们还设置`admin`用户具有超级用户权限，可以访问全部资源。其他用户则默认没有任何访问权限。

kafka提供了脚本`kafka-acls.sh`来进行查看和配置ACL。例如，对于之前的topic test，可以通过以下命令查看其ACL:

{% highlight text %}
$ ./kafka-acls.sh --authorizer-properties zookeeper.connect=127.0.0.1:2181 --list --topic test
Current ACLs for resource `Topic:test`:

{% endhighlight %}

可以看到，默认情况下是没有任何ACL的。我们使用以下下命令添加一条规则，设置来自127.0.0.1的用户producer可以写topic test：

{% highlight text %}
$ ./kafka-acls.sh --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:producer --allow-host 127.0.0.1 --operation Write --topic test
Adding ACLs for resource `Topic:test`:
        User:producer has Allow permission for operations: Write from hosts: 127.0.0.1

Current ACLs for resource `Topic:test`: 
        User:producer has Allow permission for operations: Write from hosts: 127.0.0.1 
{% endhighlight %}

由于kafka的ACL是通过zookeeper储存的，我们可以查看zookeeper中对应内容，确认ACL的配置：

{% highlight text %}
$  ./zookeeper-shell.sh 127.0.0.1:2181
Connecting to 127.0.0.1:2181
Welcome to ZooKeeper!
JLine support is disabled

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
ls /kafka-acl
[Topic]
ls /kafka-acl/Topic
[test]
get /kafka-acl/Topic/test
{"version":1,"acls":[{"principal":"User:producer","permissionType":"Allow","operation":"Write","host":"127.0.0.1"}]}
cZxid = 0x30f
ctime = Mon Dec 19 11:23:08 CST 2016
mZxid = 0x30f
mtime = Mon Dec 19 11:23:08 CST 2016
pZxid = 0x30f
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 116
numChildren = 0
{% endhighlight %}

现在，使用之前的console-producer可以继续向kafka写入内容。但是，如果我们尝试使用producer的用户名密码，去读取topic test的内容，就会提示未授权：

{% highlight text %}
$ KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka_2.11-0.10.1.0/config/kafka-client-jaas.conf" ./kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --from-beginning --consumer.config /opt/kafka_2.11-0.10.1.0/config/producer.properties --topic test
[2016-12-19 14:04:09,911] WARN The configuration 'compression.type' was supplied but isn't a known config. (org.apache.kafka.clients.consumer.ConsumerConfig)
[2016-12-19 14:04:10,163] ERROR Unknown error when running consumer:  (kafka.tools.ConsoleConsumer$)
org.apache.kafka.common.errors.GroupAuthorizationException: Not authorized to access group: console-consumer-75637
{% endhighlight %}

此外，`kafka-acl.sh`还提供了`--producer`和`--consumer`选项，可以方便的直接设置producer和consumer的ACL，具体使用说明可参考帮助文档。值得注意的是，如果使用了`--consumer`选项，则还需要通过`--group`选项指定consumer的group。示例命令如下：

{% highlight text %}
$ ./kafka-acls.sh --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:consumer --topic test --consumer --group test-consumer-group
Adding ACLs for resource `Topic:test`: 
        User:consumer has Allow permission for operations: Describe from hosts: *
        User:consumer has Allow permission for operations: Read from hosts: * 

Adding ACLs for resource `Group:test-consumer-group`: 
        User:consumer has Allow permission for operations: Read from hosts: * 

Current ACLs for resource `Topic:test`: 
        User:consumer has Allow permission for operations: Describe from hosts: *
        User:producer has Allow permission for operations: Write from hosts: *
        User:producer has Allow permission for operations: Describe from hosts: *
        User:consumer has Allow permission for operations: Read from hosts: * 

Current ACLs for resource `Group:test-consumer-group`: 
        User:consumer has Allow permission for operations: Read from hosts: * 
{% endhighlight %}

这里的test-consumer-group，在consumer.properties中设置后，再使用consumer的用户名密码去调用console-consumer，就可以通过认证和授权，读取信息了：

{% highlight bash %}
$ KAFKA_OPTS="-Djava.security.auth.login.config=/opt/kafka_2.11-0.10.1.0/config/kafka-client-consumer-jaas.conf" ./kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --from-beginning --consumer.config /opt/kafka_2.11-0.10.1.0/config/consumer.properties --topic test 
{% endhighlight %}

---

## 配置filebeat

filebeat支持output到kafka，而且可以设置使用SASL/PLAIN进行认证，只需要在文件`filebeat.yml`中填入以下内容即可：

{% highlight yml %}
output.kafka:
  hosts: ["127.0.0.1:9092"]
  topic: '%{[type]}'
  username: "producer"
  password: "producer"
{% endhighlight %}

上述各项的具体含义很明确，如果有疑问可以参考文档。

## 配置logstash

logstash支持从kafka获取input，也支持通过SASL进行认证，这里我们还是统一使用SASL/PLAIN。我们在目录`/etc/logstash/conf.d`下新建文件`test.conf`：

{% highlight ruby %}
input {
    kafka {
        bootstrap_servers => "127.0.0.1:9092"
        security_protocol => "SASL_PLAINTEXT"
        sasl_mechanism => "PLAIN"
        jaas_path => "/etc/logstash/kafka-client-jaas.conf"
        topics => ["test"]
    }   
}

output {
    stdout {
        codec => rubydebug
    }   
}
{% endhighlight %}

该配置文件将保存有consumer用户名密码的文件`/etc/logstash/kafka-client-jaas.conf`设置为`jaas_path`的值。此外，由于logstash默认的consumer group是logstash，我们在配置consumer的ACL时需要注意group需匹配。

不过，在使用logstash与SASL/PLAIN时，总会发生空指针引用导致logstash崩溃的bug：

{% highlight text %}
[2016-12-16T10:58:44,031][ERROR][logstash.inputs.kafka    ] Unable to create Kafka consumer from given configuration {:kafka_error_message=>java.lang.NullPointerException}
[2016-12-16T10:58:44,038][ERROR][logstash.pipeline        ] A plugin had an unrecoverable error. Will restart this plugin.
{% endhighlight %}

分析后发现是logstash的kafka插件的问题：即使使用PLAIN而非GSSAPI进行SASL认证，仍然会访问`sasl_kerberos_service_name`；但是使用了PLAIN，通常就不会去设置`sasl_kerberos_service_name`，而这一项的默认值是`nil`。如果遇到类似的问题，可以参考我提的[PR](https://github.com/logstash-plugins/logstash-input-kafka/pull/158)进行解决。

---

## 总结

在kafka没有提供这些安全机制之前，通过iptables对访问进行控制基本就可以满足要求。但是，通过iptables配置相对不够灵活，而且难以做到细颗粒度(例如在topic层)的权限控制。现在通过SASL以及ACL，就可以对每个topic的读、写分别进行控制，从而应对复杂环境下的要求。

此外，kafka还支持通信SSL加密。如果不使用SSL加密，就存在通信内容被嗅探的风险。不过出于性能考虑，我们权衡后没有使用。如果对安全的要求比较高，可以参考kafka官方文档配置使用SSL加密。

---

## 参考资料

- [https://kafka.apache.org/documentation/#security](https://kafka.apache.org/documentation/#security)
- [https://github.com/Symantec/kafka-security-0.9](https://github.com/Symantec/kafka-security-0.9)
- [https://www.elastic.co/guide/en/beats/filebeat/master/kafka-output.html](https://www.elastic.co/guide/en/beats/filebeat/master/kafka-output.html)
- [https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html)
