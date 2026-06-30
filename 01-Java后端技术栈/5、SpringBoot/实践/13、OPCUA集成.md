Spring Boot 集成 **Eclipse Milo** 实现 **OPC UA** 客户端/服务端，用于工业设备数据采集、SCADA、MES 与 PLC 通信等场景。

当前使用版本：**Milo 0.6.11** + **BouncyCastle 1.70**（证书与加密）。

---

# 1、依赖

```xml
<!-- OPC UA Client -->
<dependency>
    <groupId>org.eclipse.milo</groupId>
    <artifactId>sdk-client</artifactId>
    <version>0.6.11</version>
</dependency>

<!-- OPC UA Server（按需，仅采集可不引） -->
<dependency>
    <groupId>org.eclipse.milo</groupId>
    <artifactId>sdk-server</artifactId>
    <version>0.6.11</version>
</dependency>

<!-- Milo 证书、加密依赖 -->
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk15on</artifactId>
    <version>1.70</version>
</dependency>
```

---

# 2、测试类

```java
package other.opcua;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.LoggerContext;
import org.eclipse.milo.opcua.sdk.client.OpcUaClient;
import org.eclipse.milo.opcua.sdk.client.api.identity.UsernameProvider;
import org.eclipse.milo.opcua.stack.core.AttributeId;
import org.eclipse.milo.opcua.stack.core.security.SecurityPolicy;
import org.eclipse.milo.opcua.stack.core.types.builtin.LocalizedText;
import org.eclipse.milo.opcua.stack.core.types.builtin.NodeId;
import org.eclipse.milo.opcua.stack.core.types.builtin.unsigned.UInteger;
import org.eclipse.milo.opcua.stack.core.types.enumerated.MonitoringMode;
import org.eclipse.milo.opcua.stack.core.types.enumerated.TimestampsToReturn;
import org.eclipse.milo.opcua.stack.core.types.structured.MonitoredItemCreateRequest;
import org.eclipse.milo.opcua.stack.core.types.structured.MonitoringParameters;
import org.eclipse.milo.opcua.stack.core.types.structured.ReadValueId;
import org.slf4j.LoggerFactory;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.List;

/**
 * MesTest
 *
 * @author louise
 * @date 2023/9/21
 */
public class MesTest {
    //opc ua服务端地址
    private final static String endPointUrl = "opc.tcp://172.23.0.8:49320";

    static {
        LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
        List<ch.qos.logback.classic.Logger> loggerList = loggerContext.getLoggerList();
        loggerList.forEach(logger -> {
            logger.setLevel(Level.DEBUG);
        });
    }

    public static void main(String[] args) throws Exception {
        //创建OPC UA客户端
        OpcUaClient opcUaClient = createClient();

        //开启连接
        opcUaClient.connect().get();

        // 订阅消息
        subscribe(opcUaClient);

        // 关闭连接
        opcUaClient.disconnect().get();


    }

    private static OpcUaClient createClient() throws Exception {
        Path securityTempDir = Paths.get(System.getProperty("java.io.tmpdir"), "security");
        Files.createDirectories(securityTempDir);
        if (!Files.exists(securityTempDir)) {
            throw new Exception("unable to create security dir: " + securityTempDir);
        }
        return OpcUaClient.create(endPointUrl,
                endpoints ->
                        endpoints.stream()
                                .filter(e -> e.getSecurityPolicyUri().equals(SecurityPolicy.None.getUri()))
                                .findFirst(),
                configBuilder ->
                        configBuilder
                                .setApplicationName(LocalizedText.english("WS-A")) // huazh-01
                                .setApplicationUri("ns=2;s=WS-A.TR.TR21-OUT") // ns=2:s=huazh-01.device1.data-huazh
                                //访问方式 new AnonymousProvider()
                                .setIdentityProvider(new UsernameProvider("mes", "123456"))
                                .setRequestTimeout(UInteger.valueOf(5000))
                                .build()
        );
    }

    private static void subscribe(OpcUaClient client) throws Exception {
        //创建发布间隔1000ms的订阅对象
        client.getSubscriptionManager()
                .createSubscription(1000.0)
                .thenAccept(t -> {
                    //节点ns=2;s=test.device2.test2
                    NodeId nodeId = new NodeId(2, "WS-A.TR.TR21-OUT");
                    ReadValueId readValueId = new ReadValueId(nodeId, AttributeId.Value.uid(), null, null);
                    //创建监控的参数
                    MonitoringParameters parameters = new MonitoringParameters(UInteger.valueOf(1), 1000.0, null, UInteger.valueOf(10), true);
                    //创建监控项请求
                    //该请求最后用于创建订阅。
                    MonitoredItemCreateRequest request = new MonitoredItemCreateRequest(readValueId, MonitoringMode.Reporting, parameters);
                    List<MonitoredItemCreateRequest> requests = new ArrayList<>();
                    requests.add(request);
                    //创建监控项，并且注册变量值改变时候的回调函数。
                    t.createMonitoredItems(
                            TimestampsToReturn.Both,
                            requests,
                            (item, id) -> item.setValueConsumer((it, val) -> {
                                System.out.println("=====订阅nodeid====== :" + it.getReadValueId().getNodeId());
                                System.out.println("=====订阅value===== :" + val.getValue().getValue());
                            })
                    );
                }).get();

        //持续订阅
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

# 3、项目实践

## opcua 启动 OpCuaApplicationListener

```java
package cn.stylefeng.guns.traceManage.opcua;

import cn.hutool.core.thread.ThreadUtil;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.milo.opcua.sdk.client.OpcUaClient;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

import java.util.*;


/**
 * opCua发布
 *
 * @author Louise
 * @date 2023/10/10
 */
@Slf4j
@Component
public class OpCuaApplicationListener implements ApplicationListener<ApplicationReadyEvent> {

    static final String END_POINT_URL = "opc.tcp://172.23.0.8:49320";
    static final String USERNAME = "mes";
    static final String PASSWORD = "123456";

    static final Set<String> KEYS = new LinkedHashSet<>();

    static {
        KEYS.add("WS-A.TR.TR21-OUT");
    }


    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        ThreadUtil.execute(() -> {
            OpcUaClient client = OpcUaUtil.createClient(END_POINT_URL, USERNAME, PASSWORD);
            try {
                OpcUaUtil.subscribeEvent(KEYS, client);
            } catch (Exception e) {
                log.info("OpCuaApplicationListener Error");
            }
            log.info("OpCuaApplicationListener Start Success");
        });
    }
}
```

## 工具类 OpcUaUtil 失败重连

```java
package cn.stylefeng.guns.traceManage.opcua;

import cn.hutool.core.util.StrUtil;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.milo.opcua.sdk.client.OpcUaClient;
import org.eclipse.milo.opcua.sdk.client.api.config.OpcUaClientConfig;
import org.eclipse.milo.opcua.sdk.client.api.identity.AnonymousProvider;
import org.eclipse.milo.opcua.sdk.client.api.identity.IdentityProvider;
import org.eclipse.milo.opcua.sdk.client.api.identity.UsernameProvider;
import org.eclipse.milo.opcua.sdk.client.subscriptions.ManagedDataItem;
import org.eclipse.milo.opcua.sdk.client.subscriptions.ManagedSubscription;
import org.eclipse.milo.opcua.stack.client.DiscoveryClient;
import org.eclipse.milo.opcua.stack.core.security.SecurityPolicy;
import org.eclipse.milo.opcua.stack.core.types.builtin.LocalizedText;
import org.eclipse.milo.opcua.stack.core.types.builtin.NodeId;
import org.eclipse.milo.opcua.stack.core.types.builtin.unsigned.UInteger;
import org.eclipse.milo.opcua.stack.core.types.structured.EndpointDescription;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CountDownLatch;

/**
 * OpcUaUtil
 *
 * @author louise
 * @date 2023/10/9
 */
@Slf4j
public class OpcUaUtil {

    public static OpcUaClient createClient(String endPointUrl, String username, String password) {
        log.info(endPointUrl);
        try {
            //获取安全策略
            List<EndpointDescription> endpointDescription = DiscoveryClient.getEndpoints(endPointUrl).get();
            //过滤出一个自己需要的安全策略
            EndpointDescription endpoint = endpointDescription.stream()
                    .filter(e -> e.getSecurityPolicyUri().equals(SecurityPolicy.None.getUri()))
                    .findFirst().orElse(null);
            IdentityProvider identityProvider = new AnonymousProvider();
            if (StrUtil.isNotEmpty(username) || !StrUtil.isNotEmpty(password)) {
                identityProvider = new UsernameProvider(username, password);
            }
            // 设置配置信息
            OpcUaClientConfig config = OpcUaClientConfig.builder()
                    // opc ua 自定义的名称
                    .setApplicationName(LocalizedText.english("plc"))
                    // 地址
                    .setApplicationUri(endPointUrl)
                    // 安全策略等配置
                    .setEndpoint(endpoint)
                    .setIdentityProvider(identityProvider)
                    //等待时间
                    .setRequestTimeout(UInteger.valueOf(5000))
                    .build();
            // 准备连接
            OpcUaClient opcClient;
            opcClient = OpcUaClient.create(config);
            //开启连接
            opcClient.connect().get();
            log.info("======== opc connect success========");
            return opcClient;
        } catch (Exception e) {
            log.error("======== opc connect fail will reconnect again========");
            try {
                Thread.sleep(5000);
            } catch (InterruptedException ignored) {
            }
            return createClient(endPointUrl, username, password);
        }
    }

    /**
     * 订阅使用
     */
    public static void subscribeEvent(Set<String> keys, OpcUaClient client) throws Exception {
        final CountDownLatch eventLatch = new CountDownLatch(1);
        //处理订阅业务
        handlerNode(keys, client);
        //添加订阅监听器，用于处理断线重连后的订阅问题
        client.getSubscriptionManager().addSubscriptionListener(new SubscriptionListener(keys, client));
        //持续监听
        eventLatch.await();
    }

    public static void handlerNode(Set<String> keys, OpcUaClient client) {
        try {
            //创建订阅
            ManagedSubscription subscription = ManagedSubscription.create(client);
            List<NodeId> nodeIdList = new ArrayList<>();
            for (String key : keys) {
                nodeIdList.add(new NodeId(2, key));
            }
            //监听
            List<ManagedDataItem> dataItemList = subscription.createDataItems(nodeIdList);
            for (ManagedDataItem managedDataItem : dataItemList) {
                managedDataItem.addDataValueListener(new MyDataValueListener());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## SubscriptionListener

```java
package cn.stylefeng.guns.traceManage.opcua;

import lombok.extern.slf4j.Slf4j;
import org.eclipse.milo.opcua.sdk.client.OpcUaClient;
import org.eclipse.milo.opcua.sdk.client.api.subscriptions.UaSubscription;
import org.eclipse.milo.opcua.sdk.client.api.subscriptions.UaSubscriptionManager;
import org.eclipse.milo.opcua.stack.core.UaException;
import org.eclipse.milo.opcua.stack.core.types.builtin.DateTime;
import org.eclipse.milo.opcua.stack.core.types.builtin.StatusCode;

import java.util.Set;


/**
 * @author tarzan
 */
@Slf4j
public class SubscriptionListener implements UaSubscriptionManager.SubscriptionListener {


    private final Set<String> keys;
    private final OpcUaClient client;

    public SubscriptionListener(Set<String> keys, OpcUaClient client) {
        this.keys = keys;
        this.client = client;
    }

    @Override
    public void onKeepAlive(UaSubscription subscription, DateTime publishTime) {
        log.info("onKeepAlive");
    }

    /**
     * 状态更改
     */
    @Override
    public void onStatusChanged(UaSubscription subscription, StatusCode status) {
        log.info("onStatusChanged");
    }

    /**
     * 发布失败
     */
    @Override
    public void onPublishFailure(UaException exception) {
        log.error("onPublishFailure");
    }

    /**
     * 数据丢失
     */
    @Override
    public void onNotificationDataLost(UaSubscription subscription) {
        log.info("onNotificationDataLost");
    }

    /**
     * 重连时 尝试恢复之前的订阅失败时 会调用此方法
     *
     * @param uaSubscription 订阅
     * @param statusCode     状态
     */
    @Override
    public void onSubscriptionTransferFailed(UaSubscription uaSubscription, StatusCode statusCode) {
        log.info("恢复订阅失败 需要重新订阅");
        //在回调方法中重新订阅
        OpcUaUtil.handlerNode(keys, client);
    }
}
```

## 接收数据 逻辑处理 MyDataValueListener

```java
package cn.stylefeng.guns.traceManage.opcua;

import cn.hutool.extra.spring.SpringUtil;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.milo.opcua.sdk.client.subscriptions.ManagedDataItem;
import org.eclipse.milo.opcua.stack.core.types.builtin.DataValue;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * 数据监听
 *
 * @author Louise
 * @date 2023/10/10
 */
@Slf4j
public class MyDataValueListener implements ManagedDataItem.DataValueListener {

    TrStationClient trStationClient = SpringUtil.getBean(TrStationClient.class);

    EnStationClient enStationClient = SpringUtil.getBean(EnStationClient.class);

    @Override
    public void onDataValueReceived(ManagedDataItem managedDataItem, DataValue dataValue) {
        String no = managedDataItem.getNodeId().getIdentifier().toString();

        String trRegex = ".*TR21.*";
        String enRegex = "WS-A\\.EN\\.(EN[\\d]+)";

        if (dataValue.getValue().getValue() instanceof Boolean && dataValue.getValue().getValue() == Boolean.TRUE) {
            log.info(managedDataItem.getNodeId().getIdentifier().toString() + ":" + "next car");
            if (no.matches(trRegex)) {
                trStationClient.next();
            }
            if (no.matches(enRegex)) {
                Pattern r = Pattern.compile(enRegex);
                Matcher m = r.matcher(no);
                if (m.find()) {
                    enStationClient.next("A" + m.group(1));
                }
            }
        }
    }
}
```

