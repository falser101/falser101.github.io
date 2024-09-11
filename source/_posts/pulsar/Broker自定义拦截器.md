---
title: Pulsar 自定义拦截器
author: falser101
date: 2024-01-16
headerDepth: 3
category:
  - pulsar
tag:
  - pulsar
---

# 概述
Pulsar暴露了BrokerInterceptor接口，允许用户自定义拦截器，拦截器提供了多个切入点，我们可以在这些切入点上完成自定义的逻辑，切入点如表所示:
| 切入点名 | 描述 |
| --- | --- |
| beforeSendMessage | 当Broker读取完消息，还未发送给Consumer时触发 |
| onPulsarCommand | Pulsar收到任何客户端命令时触发 |
| onConnectionClosed | 当有连接关闭时触发 |
| onWebserviceRequest | 当有管理流相关的HTTP请求时触发 |
| onWebserviceResponse | 当有管理流相关的HTTP响应时触发 |
| onFilter | 管理流HTTP的Filter，当作Serverlet Filter使用即可 |
| messageProduced | 消息发送成功后触发 |

# 自定义拦截器

我们实现messageProduced这个切入点，定义一个AtomicInteger，在消息发送成功后执行incrementAndGet方法，然后打印出Message produced count。

## 实现接口
```java
@Slf4j
public class CounterBrokerInterceptor implements BrokerInterceptor {
    private final AtomicInteger messageCount = new AtomicInteger();

    @Override
    public void messageProduced(ServerCnx cnx, Producer producer, long startTimeNs, long ledgerId,
                                long entryId,
                                Topic.PublishContext publishContext) {
        messageCount.incrementAndGet();
        log.info("Message produced count {}", messageCount.get());
    }
}
```

## 编写项目描述文件
然后还需要在项目的`resources/META-INF/services/broker_interceptor.yml`文件中添加以下内容：
```yaml
name: CounterBrokerInterceptor # 名称
description: CounterBrokerInterceptor
interceptorClass: org.apache.pulsar.interceptor.CounterBrokerInterceptor # 自定义拦截器的全限定名
```

## 添加依赖
由于Pulsar在启动broker时是加载nar包，所以我们需要添加maven依赖，将broker依赖和构建nar包的插件引入
```xml
 <dependencies>
    <dependency>
      <groupId>org.apache.pulsar</groupId>
      <artifactId>pulsar-broker</artifactId>
      <version>3.3.0-SNAPSHOT</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>

  <build>
    <finalName>${project.artifactId}</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.nifi</groupId>
        <artifactId>nifi-nar-maven-plugin</artifactId>
        <version>1.2.0</version>
        <extensions>true</extensions>
        <configuration>
          <finalName>${project.artifactId}-${project.version}</finalName>
        </configuration>
        <executions>
          <execution>
            <id>default-nar</id>
            <phase>package</phase>
            <goals>
              <goal>nar</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

## 修改配置
打包成功后，将target目录下的nar包复制到pulsar的interceptors目录下,并且修改broker.conf的配置以启用拦截器
```properties
# The directory to locate broker interceptors
brokerInterceptorsDirectory=./interceptors

# resources/META-INF/services/broker_interceptor.yml文件中name定义的名称
brokerInterceptors=CounterBrokerInterceptor
```

# 启动并测试
启动broker成功后，我们进行测试，创建生产者并发送消息
```go
package consumer

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/apache/pulsar-client-go/pulsar"
)

func Produce() {
	client, err := pulsar.NewClient(pulsar.ClientOptions{
		URL:               "pulsar://localhost:6650",
		OperationTimeout:  30 * time.Second,
		ConnectionTimeout: 30 * time.Second,
	})
	if err != nil {
		log.Fatalf("Could not instantiate Pulsar client: %v", err)
	}

	defer client.Close()

	producer, err := client.CreateProducer(pulsar.ProducerOptions{
		Topic: "public/default/test",
	})
	var data = make([]byte, 1024)
	for i := 0; i < 100; i++ {
		producer.Send(context.Background(), &pulsar.ProducerMessage{
			Payload: data,
		})
	}

	if err != nil {
		log.Fatal(err)
	}
	defer producer.Close()
	fmt.Println("Published message")
}
```

调用Produce()方法，发送100条消息，查看broker的日志如下图:

![broker日志](/imgs/pulsar/broker-interceptor/发送消息.png)