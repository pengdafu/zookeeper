## 怎么指定 jass 文件

```shell
$ KAFKA_OPTS=-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jass.conf ./bin/kafka-server-start.sh config/server.properties
```

## jass 文件需要注意的地方

最后结尾记得要有 `;` 号。