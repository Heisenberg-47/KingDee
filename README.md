# 苍穹MQ 使用文档

## 引言

在当今的分布式系统中，消息队列（MQ）扮演着至关重要的角色，它不仅提高了系统的解耦性，还增强了系统的可扩展性和容错性。本文档将为您提供如何在苍穹环境中集成和使用消息队列的详细指南，并通过一个具体的例子展示消息的发送与接收过程。

---

## 环境配置

### 启动类配置

在启动类中，您需要设置以下系统属性以配置MQ：

```java
System.setProperty("dubbo.registry.register", "true");
System.setProperty("mq.consumer.register", "true");
System.setProperty("mq.debug.queue.tag", "Heisenberg");
System.setProperty("lightweightdeploy", "false");
System.setProperty("mqConfigFiles.config", "queue/consummqconfig.xml");
```

**属性说明：**

- `dubbo.registry.register`：是否注册到Dubbo注册中心。
- `mq.consumer.register`：配置本节点进行消费者注册，设置为`true`后本节点能接收MQ消息。
- `mq.debug.queue.tag`：用于调试的队列标签，应唯一，确保本机发送的消息被本机上的Java进程消费。
- `lightweightdeploy`：是否为轻量级部署。
- `mqConfigFiles.config`：MQ配置文件的路径。

### MQ配置文件

在`queue/consummqconfig.xml`文件中定义MQ的区域、队列和消费者信息：

```xml
<!-- region为云的标识，queue name可自定义，appid为应用标识，class为消费者的全路径。-->
<root>
    <region name="tpv_base">
        <queue name="demo_queue" appid="tpv_cm">
            <consumer class="tpv.wm.dcs.cm.operate.DemoConsumer"></consumer>
        </queue>
    </region>
</root>
```

这里选择了`tpv_base`的应用，包含一个名为`demo_queue`的队列，关联了一个消费者`DemoConsumer`。

---

## 生产者（消息发送者）

### 代码实现

生产者负责将消息发送到MQ。以下是生产者的代码实现：

```java
package tpv.wm.dcs.cm.operate;

import kd.bos.logging.Log;
import kd.bos.logging.LogFactory;
import kd.bos.entity.plugin.AbstractOperationServicePlugIn;
import kd.bos.entity.plugin.args.EndOperationTransactionArgs;
import kd.bos.mq.MQFactory;
import kd.bos.mq.MessagePublisher;

import java.util.Arrays;

public class AttUploadSubmitOp extends AbstractOperationServicePlugIn {
    Log log = LogFactory.getLog(getClass());

    @Override
    public void endOperationTransaction(EndOperationTransactionArgs e) {
        super.endOperationTransaction(e);
        MessagePublisher mq = MQFactory.get().createSimplePublisher("tpv_base", "demo_queue");
        try {
            log.info("mq发送消息。");
            Arrays.stream(e.getDataEntities())
                    .forEach(single -> mq.publishInDbTranscation(single.getPkValue()+"1"));
        } finally {
            mq.close();
        }
    }
}
```

**代码说明：**

- `MQFactory.get().createSimplePublisher("tpv_base", "demo_queue")`：创建消息发布者，指定区域和队列。
- `mq.publishInDbTranscation()`：在数据库事务中发布消息。

---

## 消费者（消息接收者）

### 代码实现

消费者负责从MQ接收并处理消息。以下是消费者的代码实现：

```java
package tpv.wm.dcs.cm.operate;

import kd.bos.logging.Log;
import kd.bos.logging.LogFactory;
import kd.bos.entity.operate.result.OperateErrorInfo;
import kd.bos.entity.operate.result.OperationResult;
import kd.bos.mq.MessageAcker;
import kd.bos.mq.MessageConsumer;
import kd.bos.orm.query.QFilter;
import kd.bos.servicehelper.QueryServiceHelper;
import kd.bos.servicehelper.operation.OperationServiceHelper;

import java.util.List;

public class DemoConsumer implements MessageConsumer {
    Log log = LogFactory.getLog(getClass());

    @Override
    public void onMessage(Object id, String messageId, boolean resend, MessageAcker acker) {
        log.info("DemoConsumer onMessage demo 开始消费。");
        try {
            OperationResult result = OperationServiceHelper.executeOperate("audit", "tpv_mqtest", new Object[]{id}, OperateOption.create());
            List<OperateErrorInfo> errorInfo = result.getAllErrorInfo();
            if (errorInfo.isEmpty()) {
                acker.ack(messageId);
                return;
            }
            QFilter q1 = new QFilter("id", "=", id);
            QFilter q2 = new QFilter("billstatus", "=", "C");
            boolean discard = QueryServiceHelper.exists("tpv_mqtest", new QFilter[]{q1, q2});
            if (discard) {
                acker.discard(messageId);
                log.info(id + "已经审核，此消息丢弃");
            } else {
                acker.deny(messageId);
                log.error(result.getMessage());
            }
        } catch (Exception e) {
            acker.deny(messageId);
            log.error(e);
        }
    }
}
```

**代码说明：**

- `onMessage(Object id, String messageId, boolean resend, MessageAcker acker)`：处理接收到的消息。
- `acker.ack(messageId)`：确认消息已成功处理。
- `acker.discard(messageId)`：丢弃消息，不再重试。
- `acker.deny(messageId)`：消息处理失败，请求重试。

---

## 效果展示

### MQ配置文件

![MQ配置文件](https://s2.loli.net/2024/11/04/bR4wZ1XvfUBaqlT.png)

### 消息发送成功

![消息发送成功](https://s2.loli.net/2024/11/04/iur3Th9PXobMG6Y.png)

这张截图显示了消息成功发送到MQ的日志输出。`log.info("mq发送消息。")`表明消息已经成功发送，没有出现异常。

### 消息消费成功

![消息消费成功](https://s2.loli.net/2024/11/04/NTdMICx6Z1PWv2f.png)

在这张截图中，您可以看到消费者成功接收并处理了消息。`log.info("DemoConsumer onMessage demo 开始消费。")`对应单据已审核，表明消息已经被正确消费。

### 消息重试机制

![消息重试机制](https://s2.loli.net/2024/11/04/fanijATZxIkuwG6.png)

生产者发送一个异常过去。

![消息重试成功](https://s2.loli.net/2024/11/04/XJHPtVqCmiTzpyG.png)

### 死信队列配置

设置`x-dead-letter-exchange`和`x-message-ttl`参数，以确保消息在一定次数的失败后被发送到死信队列。

---

## FAQ

### 资源文件未正确加载

**Q:** 为什么我的资源文件没有被正确加载？

**A:** 如果您的资源文件（如XML配置文件、图片、数据库脚本等）没有被正确加载，可能是因为项目的`resources`目录没有被设置为资源目录。在大多数IDE（如IntelliJ IDEA或Eclipse）和构建工具（如Maven或Gradle）中，您需要确保`resources`目录被正确配置。

### 设置`resources`目录

**Q:** 如何在IDE中设置`resources`目录？

**A:** 在IDE中设置`resources`目录的步骤通常如下：

- **IntelliJ IDEA:**
  1. 右键点击`resources`目录。
  2. 选择`Mark Directory as` > `Resources Root`。
- **Eclipse:**
  1. 右键点击`resources`目录。
  2. 选择`Build Path` > `Use as Source Folder`。

确保您的资源文件放在这个目录下，IDE将会自动将这些资源文件包含在构建的输出中。

![设置resources目录](https://s2.loli.net/2024/11/04/IN9oFuiqJYOP4wj.png)

### RabbitMQ地址

**Q:** RabbitMQ地址是什么？

**A:** RabbitMQ地址为`ip + :15672`。

![RabbitMQ地址](https://s2.loli.net/2024/11/04/xrAb3SptuCUg4BG.png)

---

以上是对苍穹MQ使用文档的优化版，包括了更清晰的结构、详细的说明和适当的排版，以提高文档的可读性和实用性。希望这能帮助您更好地理解和使用苍穹MQ。
