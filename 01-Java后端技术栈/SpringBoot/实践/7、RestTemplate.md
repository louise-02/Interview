**RestTemplate** 是 Spring 提供的同步 HTTP 客户端模板。适合在 Spring 项目里调第三方 REST 接口、简单服务间 HTTP 调用。

与 [6、HttpClient](./6、HttpClient.md) 是**两个东西**：RestTemplate 是 Spring 的 API；HttpClient 是 Apache 的底层库。

微服务服务间调用更推荐 OpenFeign，见 [SpringCloud 实践](../../SpringCloud/实践/1、Nacos+Gateway+OpenFeign搭建微服务.md)。

---

# 1、依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

`RestTemplate` 在 `spring-web` 里，随 starter-web 即有。默认底层是 JDK `HttpURLConnection`。

---

# 2、注册 Bean

```java
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

import java.time.Duration;

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
                .setConnectTimeout(Duration.ofSeconds(3))
                .setReadTimeout(Duration.ofSeconds(5))
                .build();
    }
}
```

> 需要 Apache 连接池时，那是 HttpClient 的事，见 [HttpClient](./6、HttpClient.md)；可选通过 `HttpComponentsClientHttpRequestFactory` 挂到 RestTemplate，**非必须**。

---

# 3、GET 请求

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.List;
import java.util.Map;

@Service
public class RemoteUserService {

    @Autowired
    private RestTemplate restTemplate;

    private static final String BASE = "https://jsonplaceholder.typicode.com";

    /** 最简单：直接反序列化 */
    public Map<?, ?> getUser(Long id) {
        String url = BASE + "/users/{id}";
        return restTemplate.getForObject(url, Map.class, id);
    }

    /** 需要状态码、响应头 */
    public ResponseEntity<Map> getUserEntity(Long id) {
        String url = BASE + "/users/{id}";
        return restTemplate.getForEntity(url, Map.class, id);
    }

    /** 泛型集合必须用 exchange + ParameterizedTypeReference */
    public List<Map<String, Object>> listUsers() {
        String url = BASE + "/users";
        ResponseEntity<List<Map<String, Object>>> response = restTemplate.exchange(
                url,
                HttpMethod.GET,
                null,
                new ParameterizedTypeReference<List<Map<String, Object>>>() {}
        );
        return response.getBody();
    }
}
```

**带 Header（Token）的 GET**：

```java
import org.springframework.http.*;

public Map<?, ?> getWithToken(String url, String token) {
    HttpHeaders headers = new HttpHeaders();
    headers.setBearerAuth(token);
    HttpEntity<Void> entity = new HttpEntity<>(headers);

    ResponseEntity<Map> response = restTemplate.exchange(
            url, HttpMethod.GET, entity, Map.class);
    return response.getBody();
}
```

验证：

```bash
# 等价于 getUser(1)
curl https://jsonplaceholder.typicode.com/users/1
```

---

# 4、POST 请求（JSON）

```java
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.HashMap;
import java.util.Map;

@Service
public class RemotePostService {

    @Autowired
    private RestTemplate restTemplate;

    public Map<?, ?> createPost(String title, String body) {
        String url = "https://jsonplaceholder.typicode.com/posts";

        Map<String, Object> requestBody = new HashMap<>();
        requestBody.put("title", title);
        requestBody.put("body", body);
        requestBody.put("userId", 1);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        HttpEntity<Map<String, Object>> entity = new HttpEntity<>(requestBody, headers);

        ResponseEntity<Map> response = restTemplate.postForEntity(url, entity, Map.class);
        return response.getBody();
    }
}
```

简写（只关心 body、不关心 Header 时）：

```java
Map<String, Object> body = Map.of("title", "foo", "body", "bar", "userId", 1);
Map<?, ?> result = restTemplate.postForObject(url, body, Map.class);
```

---

# 5、POST 请求（上传文件 multipart）

RestTemplate 上传文件用 **`MultiValueMap` + `FileSystemResource`**（或 `ByteArrayResource`）。

```java
import org.springframework.core.io.FileSystemResource;
import org.springframework.http.*;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;

import java.io.File;
import java.util.Map;

public class RestTemplateUploadExample {

    private final RestTemplate restTemplate;

    public RestTemplateUploadExample(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    /**
     * @param uploadUrl  接收方地址，如 http://127.0.0.1:8080/api/file/upload
     * @param file       本地文件
     * @param fieldName  与服务端 @RequestParam("file") 一致
     */
    public Map<?, ?> uploadFile(String uploadUrl, File file, String fieldName) {
        MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
        body.add(fieldName, new FileSystemResource(file));
        body.add("description", "RestTemplate 上传");   // 其它文本字段

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.MULTIPART_FORM_DATA);

        HttpEntity<MultiValueMap<String, Object>> entity = new HttpEntity<>(body, headers);

        ResponseEntity<Map> response = restTemplate.postForEntity(uploadUrl, entity, Map.class);
        return response.getBody();
    }
}
```

**从 MultipartFile 转上传**（已在 Controller 拿到文件，再转发到别的服务）：

```java
import org.springframework.core.io.ByteArrayResource;
import org.springframework.web.multipart.MultipartFile;

public Map<?, ?> forwardUpload(String targetUrl, MultipartFile multipartFile) throws IOException {
    ByteArrayResource resource = new ByteArrayResource(multipartFile.getBytes()) {
        @Override
        public String getFilename() {
            return multipartFile.getOriginalFilename();   // 必须重写，否则对方收不到文件名
        }
    };

    MultiValueMap<String, Object> body = new LinkedMultiValueMap<>();
    body.add("file", resource);

    HttpEntity<MultiValueMap<String, Object>> entity =
            new HttpEntity<>(body, new HttpHeaders());   // 不要手动 set Content-Type，让 FormHttpMessageConverter 带 boundary

    return restTemplate.postForObject(targetUrl, entity, Map.class);
}
```

验证（配合 [静态资源映射](./4、静态资源映射与WebMvcConfigurer.md) 的上传接口）：

```java
File file = new File("/path/to/test.jpg");
uploadFile("http://127.0.0.1:8080/api/file/upload", file, "file");
```

---

# 6、统一异常封装

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.ResourceAccessException;
import org.springframework.web.client.RestTemplate;

@Slf4j
@Component
public class RestTemplateHelper {

    private final RestTemplate restTemplate;

    public RestTemplateHelper(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public <T> T get(String url, Class<T> clazz, Object... uriVariables) {
        try {
            return restTemplate.getForObject(url, clazz, uriVariables);
        } catch (HttpClientErrorException | HttpServerErrorException e) {
            log.error("HTTP 错误 status={}, body={}", e.getStatusCode(), e.getResponseBodyAsString());
            throw new BusinessException(502, "远程服务返回错误");
        } catch (ResourceAccessException e) {
            log.error("连接超时或网络不可达", e);
            throw new BusinessException(504, "远程服务不可用");
        }
    }
}
```

`BusinessException` 见 [统一返回值与异常](./1、统一返回值与异常.md)。

---

# 7、Spring Cloud @LoadBalanced（可选）

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder.build();
}

// restTemplate.getForObject("http://user-service/users/1", Map.class);
```

需 `spring-cloud-starter-loadbalancer`，`user-service` 为注册中心里的服务名。

---

# 8、RestTemplate vs HttpClient vs WebClient

| | RestTemplate | HttpClient | WebClient |
| -- | ------------ | ---------- | --------- |
| 归属 | Spring | Apache | Spring |
| 典型用法 | `getForObject` / `HttpEntity` | `HttpGet` / `HttpPost` | 响应式 `Mono` |
| 文件上传 | `MultiValueMap` + `Resource` | `MultipartEntityBuilder` | `MultipartBodyBuilder` |
| 状态 | 维护模式 | 独立库，长期可用 | 新项目推荐 |

---

# 9、常见问题

| 现象 | 处理 |
| ---- | ---- |
| POST JSON 415 | 未设 `Content-Type: application/json` |
| 上传 400 | 字段名与 `@RequestParam` 不一致；`ByteArrayResource` 未重写 `getFilename()` |
| 上传 boundary 错误 | multipart 不要手动写死 `Content-Type`，交给 Spring 生成 |
| 泛型 List 反序列化失败 | 用 `exchange` + `ParameterizedTypeReference`，不要 `List.class` |

---

# 10、注意

- `RestTemplate` 线程安全，全局共用一个 Bean 即可。
- 大文件上传注意内存：`MultipartFile.getBytes()` 会整文件进内存，超大文件用流式或 HttpClient 直传。
- 新项目长期维护可考虑 **WebClient**；纯 Apache 控制看 [HttpClient](./6、HttpClient.md)。
