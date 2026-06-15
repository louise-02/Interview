**Apache HttpClient** 是独立的 HTTP 客户端库，不依赖 Spring。适合在非 Web 项目、定时任务、工具类里直接发 HTTP 请求；需要连接池、精细控制超时和 multipart 上传时用它。

与 [7、RestTemplate](./7、RestTemplate.md) 是**两个东西**：HttpClient 自己建连发请求；RestTemplate 是 Spring 封装的模板，底层可以换实现，但 API 是 Spring 的。

---

# 1、依赖

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>
```

纯 Java 项目写死版本即可，例如 `4.5.14`。

---

# 2、创建客户端（连接池 + 超时）

```java
import org.apache.http.client.config.RequestConfig;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;

import java.util.concurrent.TimeUnit;

public class HttpClientFactory {

    public static CloseableHttpClient create() {
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(200);
        cm.setDefaultMaxPerRoute(50);

        RequestConfig config = RequestConfig.custom()
                .setConnectTimeout(3000)              // TCP 连接超时 ms
                .setSocketTimeout(5000)                 // 读响应超时 ms
                .setConnectionRequestTimeout(2000)      // 从池取连接超时 ms
                .build();

        return HttpClients.custom()
                .setConnectionManager(cm)
                .setDefaultRequestConfig(config)
                .evictIdleConnections(30, TimeUnit.SECONDS)
                .evictExpiredConnections()
                .build();
    }
}
```

Spring 项目可注册 Bean：

```java
@Bean(destroyMethod = "close")
public CloseableHttpClient httpClient() {
    return HttpClientFactory.create();
}
```

用完记得 `close()`，或用 try-with-resources 包住单次 `CloseableHttpResponse`。

---

# 3、GET 请求

```java
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.util.EntityUtils;

import java.nio.charset.StandardCharsets;

public class HttpClientGetExample {

    public static String get(CloseableHttpClient httpClient, String url) throws Exception {
        HttpGet httpGet = new HttpGet(url);
        // 可选 Header
        httpGet.setHeader("Accept", "application/json");
        httpGet.setHeader("Authorization", "Bearer your-token");

        try (CloseableHttpResponse response = httpClient.execute(httpGet)) {
            int status = response.getStatusLine().getStatusCode();
            String body = EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);

            if (status >= 200 && status < 300) {
                return body;
            }
            throw new RuntimeException("GET 失败 status=" + status + ", body=" + body);
        }
    }
}
```

**带查询参数**（不要手拼字符串，用 URIBuilder）：

```java
import org.apache.http.client.utils.URIBuilder;

URI uri = new URIBuilder("https://api.example.com/search")
        .addParameter("keyword", "java")
        .addParameter("page", "1")
        .build();
HttpGet httpGet = new HttpGet(uri);
```

验证（公开测试 API）：

```java
CloseableHttpClient client = HttpClientFactory.create();
String json = HttpClientGetExample.get(client, "https://jsonplaceholder.typicode.com/users/1");
System.out.println(json);
client.close();
```

---

# 4、POST 请求（JSON）

```java
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.ContentType;
import org.apache.http.entity.StringEntity;

public class HttpClientPostExample {

    public static String postJson(CloseableHttpClient httpClient, String url, String jsonBody)
            throws Exception {
        HttpPost httpPost = new HttpPost(url);
        httpPost.setHeader("Content-Type", "application/json;charset=UTF-8");

        StringEntity entity = new StringEntity(jsonBody, ContentType.APPLICATION_JSON);
        httpPost.setEntity(entity);

        try (CloseableHttpResponse response = httpClient.execute(httpPost)) {
            int status = response.getStatusLine().getStatusCode();
            String body = EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);

            if (status >= 200 && status < 300) {
                return body;
            }
            throw new RuntimeException("POST 失败 status=" + status + ", body=" + body);
        }
    }
}
```

调用示例：

```java
String json = "{\"title\":\"foo\",\"body\":\"bar\",\"userId\":1}";
String result = HttpClientPostExample.postJson(
        client,
        "https://jsonplaceholder.typicode.com/posts",
        json);
```

**POST 表单**（`application/x-www-form-urlencoded`）：

```java
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.message.BasicNameValuePair;

List<BasicNameValuePair> params = new ArrayList<>();
params.add(new BasicNameValuePair("username", "admin"));
params.add(new BasicNameValuePair("password", "123456"));
httpPost.setEntity(new UrlEncodedFormEntity(params, StandardCharsets.UTF_8));
```

---

# 5、POST 请求（上传文件 multipart）

```java
import org.apache.http.entity.mime.MultipartEntityBuilder;
import org.apache.http.entity.mime.content.FileBody;
import org.apache.http.entity.mime.content.StringBody;

import java.io.File;

public class HttpClientUploadExample {

    public static String uploadFile(CloseableHttpClient httpClient,
                                    String url,
                                    File file,
                                    String fileFieldName) throws Exception {
        HttpPost httpPost = new HttpPost(url);

        // multipart/form-data
        org.apache.http.HttpEntity multipart = MultipartEntityBuilder.create()
                .addPart(fileFieldName, new FileBody(file))           // 文件字段名与接口一致
                .addPart("description", new StringBody("备注", ContentType.TEXT_PLAIN))
                .build();

        httpPost.setEntity(multipart);
        // Content-Type 含 boundary，由 MultipartEntityBuilder 自动设置，不要手动写死

        try (CloseableHttpResponse response = httpClient.execute(httpPost)) {
            int status = response.getStatusLine().getStatusCode();
            String body = EntityUtils.toString(response.getEntity(), StandardCharsets.UTF_8);
            if (status >= 200 && status < 300) {
                return body;
            }
            throw new RuntimeException("上传失败 status=" + status + ", body=" + body);
        }
    }
}
```

**multipart 依赖**（`FileBody` 在 httpmime 里）：

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpmime</artifactId>
</dependency>
```

调用：

```java
File file = new File("/path/to/photo.jpg");
String resp = HttpClientUploadExample.uploadFile(
        client,
        "http://127.0.0.1:8080/api/file/upload",
        file,
        "file");    // 与服务端 @RequestParam("file") 一致
```

---

# 6、封装成 Service（Spring 项目）

```java
import org.springframework.stereotype.Service;

@Service
public class RemoteApiService {

    private final CloseableHttpClient httpClient;

    public RemoteApiService(CloseableHttpClient httpClient) {
        this.httpClient = httpClient;
    }

    public String getUser(Long id) throws Exception {
        return HttpClientGetExample.get(
                httpClient,
                "https://jsonplaceholder.typicode.com/users/" + id);
    }

    public String createPost(String json) throws Exception {
        return HttpClientPostExample.postJson(
                httpClient,
                "https://jsonplaceholder.typicode.com/posts",
                json);
    }

    public String upload(File file) throws Exception {
        return HttpClientUploadExample.uploadFile(
                httpClient,
                "http://127.0.0.1:8080/api/file/upload",
                file,
                "file");
    }
}
```

---

# 7、常见问题

| 现象 | 处理 |
| ---- | ---- |
| 上传 400 | 表单字段名 `file` 与服务端 `@RequestParam` 不一致 |
| 中文乱码 | `StringEntity` / `EntityUtils` 指定 `StandardCharsets.UTF_8` |
| 连接池耗尽 | 调大 `maxPerRoute`；检查是否未 `close()` response |
| `Connection reset` | 开 `evictIdleConnections`；调 `validateAfterInactivity` |

---

# 8、与 RestTemplate 怎么选

| 场景 | 建议 |
| ---- | ---- |
| 已在 Spring 项目、调 REST API | [RestTemplate](./7、RestTemplate.md) |
| 非 Spring、脚本、对 HttpClient API 要完全控制 | 本篇 HttpClient |
| 微服务服务间调用 | OpenFeign（见 SpringCloud 实践） |

两者不要混在一坨配置里；RestTemplate 若要用连接池，是**可选**把 HttpClient 设成它的底层工厂，那是 RestTemplate 的事，见 RestTemplate 篇末尾一句即可。
