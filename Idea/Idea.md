# 常用快捷键

| 快捷键             | 说明                          |
| :----------------- | :---------------------------- |
| psvm               | 快速生成 main 方法            |
| ctrl + z           | 撤回到上一步                  |
| ctrl + r           | 替换文本                      |
| ctrl + d           | 复制当前行                    |
| ctrl + x           | 删除当前行                    |
| ctrl + p           | 查看方法参数                  |
| ctrl + f           | 本窗口查找文本                |
| ctrl + h           | 查看子类                      |
| ctrl + o           | 重写父类方法                  |
| ctrl + h           | 查看父子类体系                |
| ctrl + alt + <     | 返回上一步                    |
| ctrl + alt + v     | 提取相同变量                  |
| ctrl + alt + m     | 抽取方法                      |
| ctrl + alt + l     | 格式化代码                    |
| ctrl + alt + t     | 提取代码到 if/try/runnable 等 |
| ctrl + shift + u   | 转换大小写                    |
| shift + shift      | 全局搜索框                    |
| shift + f6         | 选中所有变量并修改            |
| alt + insert       | get/set 方法和构造方法        |
| alt + 鼠标往下多行 | 多行同时修改                  |
| 代码.var           | 补充变量声明                  |

# 文件模板

File - Settings - Editor - File and Code Templates

1、Class 模板

```java
#if (${PACKAGE_NAME} && ${PACKAGE_NAME} != "")package ${PACKAGE_NAME};#end
#parse("File Header.java")
#if($NAME.endsWith("Controller"))
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import javax.annotation.Resource;
#end
#if($NAME.endsWith("Config"))
import org.springframework.context.annotation.Configuration;
#end
#if($NAME.endsWith("Application"))
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
#end
#if($NAME.endsWith("ServiceImpl"))
import org.springframework.stereotype.Service;
#end
/**
 * ${NAME}
 * 
 * @author louise
 * @date ${DATE}
 */
#if($NAME.endsWith("Controller"))
@RestController 
#if($NAME.indexOf("Controller") > 0)
@RequestMapping("$NAME.substring(0,1).toLowerCase()$NAME.substring(1,$NAME.indexOf("Controller"))")
#end
#end
#if($NAME.endsWith("Config"))
@Configuration
#end
#if($NAME.endsWith("Application"))
@SpringBootApplication
#end
#if($NAME.endsWith("ServiceImpl"))
@Service
#end
public class ${NAME} {
    #if($NAME.indexOf("Controller") > 0)
    @Resource
    $NAME.substring(0,$NAME.indexOf("Controller"))Service $NAME.substring(0,1).toLowerCase()$NAME.substring(1,$NAME.indexOf("Controller"))Service;
    #end
    #if($NAME.endsWith("Application"))
    public static void main(String[] args) {
        SpringApplication.run(${NAME}.class,args);
    }
    #end
}
```

2、Interface 模板

```java
#if (${PACKAGE_NAME} && ${PACKAGE_NAME} != "")package ${PACKAGE_NAME};#end
#parse("File Header.java")
/**
 * ${NAME}
 * 
 * @author louise
 * @date ${DATE}
 */
public interface ${NAME} {
}
```

3、Enum 模板

```java
#if (${PACKAGE_NAME} && ${PACKAGE_NAME} != "")package ${PACKAGE_NAME};#end
#parse("File Header.java")
/**
 * ${NAME}
 * 
 * @author louise
 * @date ${DATE}
 */
public enum ${NAME} {
}
```

4、Xml 模板

```
#if($NAME.contains("Mapper"))
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="${NAME}">

</mapper>
#end
```

# 代码模板

File - Settings - Editor - Live Templates

1、创建 Myself 模板组

2、创建代码模板

doc

```java
/**
 *
 */
```

test

```java
@Test
public void test() {

}
```

thread

```java
new Thread(() -> {

}).start();
```

3、左下角选择适用文件类型

# 正则表达式替换

ctrl + r 使用正则表达式替换

**特殊规则**

```
\l 将字符更改为小写，直到字符串中的下一个字符，例如，BAR 变成 bAR
\u 将字符更改为大写，直到字符串中的下一个字符，例如，bar 变成 Bar
\L 将字符更改为小写，直到文字字符串的末尾，例如，BAR 变成 bar
\U 将字符更改为大写，直到文字字符串的末尾，例如，bar 变成 BAR
```

**常用正则**

```
//驼峰 _a 替换 A
[_{1}]([a-z+])
\u$1

//中文
(.*[\u4e00-\u9fa5]+.*)
```

# 激活 Idea

File - Settings - Plugins

1、Manage Plugin Repositories 中配置 https://plugins.zhile.io/ 或 http://plugins.xeblog.cn

2、安装插件 IDE Eval Reset

3、在 Help 菜单中点击 Eval Reset 进行重置

# 注释中引用类方法

注释中标注类或者方法，更容易理解代码

```java
/**
 * {@link com.km.biz.peripheralinterface.TiMaterialBiz#sapWmsPart(List, User)}
 */
```

# 常用插件

.ignore

Easy Javadoc：生成注释

IDE Eval Reset：idea 破解

Maven Helper

MybatisX

Translation

Easy Code：代码生成

RestfulTool：使用 ctrl + alt + / 快速定位接口

Activiti Bpmn visualizer

jbl javatoweb

jclasslib Bytecode Viewer