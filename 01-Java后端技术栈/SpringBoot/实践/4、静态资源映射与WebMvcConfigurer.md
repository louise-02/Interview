把**磁盘目录**或 **classpath 目录**下的图片、文件映射成 HTTP 可访问 URL。上传后立刻能通过浏览器打开，不必再走 Controller 读流。

原理：`ResourceHandlerRegistry` 把 URL 模式映射到 `Resource` 位置。见 [4、Web与数据层整合](../4、Web与数据层整合.md#8cors-与-webmvcconfigurer)。

---

# 1、依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

---

# 2、配置项（绝对路径用）

`application.yml`：

```yaml
file:
  upload:
    # 本机目录，末尾不要加 /；Windows 示例：D:/data/upload
    path: /data/upload
    # 对外 URL 前缀，访问：http://host:port/upload/xxx.jpg
    url-prefix: /upload/**
```

---

# 3、WebMvcConfigurer 映射

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Value("${file.upload.path}")
    private String filePath;

    @Value("${file.upload.url-prefix}")
    private String pathPatterns;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 绝对路径：磁盘目录 → URL
        // 注意 file: 前缀；目录末尾加 /
        registry.addResourceHandler(pathPatterns)
                .addResourceLocations("file:" + filePath + "/");

        // 相对路径：classpath 下 files 目录 → /pic/**
        // 文件放在 src/main/resources/files/ 下
        registry.addResourceHandler("/pic/**")
                .addResourceLocations("classpath:/files/");
    }
}
```

**映射关系**

| 请求 URL | 实际文件 |
| -------- | -------- |
| `GET /upload/2024/a.jpg` | `/data/upload/2024/a.jpg` |
| `GET /pic/logo.png` | `classpath:/files/logo.png` |

> Spring Boot 默认还会映射 `classpath:/static/`、`classpath:/public/` 等到 `/**`，自定义 `addResourceHandlers` **不会覆盖**默认规则，是**追加**。

---

# 4、上传接口（写入绝对路径目录）

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

@RestController
@RequestMapping("/api/file")
public class FileUploadController {

    @Value("${file.upload.path}")
    private String filePath;

    @Value("${file.upload.url-prefix}")
    private String urlPrefix;

    @PostMapping("/upload")
    public Map<String, String> upload(@RequestParam("file") MultipartFile file) throws IOException {
        String original = file.getOriginalFilename();
        String ext = original != null && original.contains(".")
                ? original.substring(original.lastIndexOf(".")) : "";
        String fileName = UUID.randomUUID() + ext;

        File dir = new File(filePath);
        if (!dir.exists()) {
            dir.mkdirs();
        }
        file.transferTo(new File(dir, fileName));

        // urlPrefix 配置为 /upload/**，对外路径去掉 /** 只拼文件名
        String accessUrl = urlPrefix.replace("/**", "") + "/" + fileName;

        Map<String, String> result = new HashMap<>();
        result.put("fileName", fileName);
        result.put("url", accessUrl);
        return result;
    }
}
```

验证：

```bash
mkdir -p /data/upload
curl -F "file=@/path/to/test.jpg" http://127.0.0.1:8080/api/file/upload
# 返回 url: /upload/xxx.jpg
curl -I http://127.0.0.1:8080/upload/xxx.jpg
```

---

# 5、classpath 静态文件

在 `src/main/resources/files/demo.png` 放一张图：

```bash
curl -I http://127.0.0.1:8080/pic/demo.png
```

打包进 jar 后仍可用；**不能**通过接口往 classpath 里写（只读）。

---

# 6、注意

| 点 | 说明 |
| -- | ---- |
| `file:` 前缀 | 磁盘路径必须写 `file:/data/upload/`，漏写 `file:` 会找不到 |
| 目录权限 | 进程用户对 `file.upload.path` 要有读写权限 |
| 安全 | `/upload/**` 不要映射整个系统盘；生产应对文件名做白名单、防目录穿越 |
| 拦截器 | 静态资源若被拦截器拦住，在 `excludePathPatterns` 加 `/upload/**`、`/pic/**` |
| 缓存 | 可加 `.setCachePeriod(3600)` 或 `resourceChain(true)` 开资源链 |

```java
registry.addResourceHandler("/upload/**")
        .addResourceLocations("file:" + filePath + "/")
        .setCachePeriod(3600);
```

---

# 7、与 Nginx 分工

- **开发 / 小项目**：Boot 直接 `addResourceHandlers` 够用。
- **生产大流量静态资源**：Nginx `alias` 或 OSS + CDN；应用只返回 URL。
