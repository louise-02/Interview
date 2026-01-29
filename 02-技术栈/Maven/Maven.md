# 1、常用命令

```
-Dmaven.test.skip=true                              //跳过编译、测试
-Dfile=D:\MvnProject\service-mvn-1.0.0.jar          //jar包文件地址,绝对路径
-DgroupId=pri.roy.mvn.test                          //gruopId--pom坐标，自定义
-DartifactId=mvn-api                                //artifactId--pom坐标，自定义
-Dversion                                           //版本号
-Dpackaging                                         //打包方式
-DrepositoryId                                      //远程库服务器ID
-Durl                                               //远程库服务器地址
-Pxxx 						 //激活 id 为 xxx的profile (如有多个，用逗号隔开);
-s  xxx.settings-nexus.xml			         //配置文件地址
pomFile					         //指定本地pom文件作为组件的pom
```

## 安装 jar 包

```bash
mvn install:install-file  
项目包名 
-DgroupId=com.hxtt.sql.access
工程名 
-DartifactId=accessDriver
版本号
-Dversion=1.0
 包的形式(pom、jar、war)
-Dpackaging=jar 
包所在的路径
-Dfile="D:\download\Access_JDBC30.jar"
```

## 下载 jar

```bash
mvn dependency:get 
-DremoteRepositories=http://47.105.122.146:8082/repository/maven-public  
-DgroupId=com.zkyd 
-DartifactId=zkyd-backend
-Dversion=2.0-RELEASE
```

## 打包跳过测试

```bash
mvn clean package -Dmaven.test.skip=true
```

## 部署文件

```bash
mvn deploy:deploy-file
-DgroupId=com.my 
-DartifactId=test 
-Dversion=1.0-SNAPSHOT 
-Dpackaging=jar 
-Dfile=E:\test.jar  
-Durl=http://localhost:8081/repository/maven-releases
-DrepositoryId=maven-releases
-Pmy-profile
-s D:\develop\maven\apache-maven-3.5.2\conf\settings-nexus.xml
```

# 2、常用依赖

## 启动依赖

```xml
<!-- web -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- 测试模块 -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-test</artifactId>
   <scope>test</scope>
</dependency>

<!-- 切面 -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<!-- 校验 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!--配置文件提示-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>

<!--监控检测-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!--log4j2-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>

<!--quartz定时任务-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>

<!--redis-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## 数据层

```xml
<!--mysql驱动-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>

<!--oracle驱动 oracle.jdbc.OracleDriver -->
<dependency>
    <groupId>com.oracle</groupId>
    <artifactId>ojdbc6</artifactId>
    <version>11.2.0.3</version>
</dependency>

<!--oracle驱动 oracle.jdbc.driver.OracleDriver -->
<dependency>
    <groupId>oracle.jdbc</groupId>
    <artifactId>ojdbc</artifactId>
    <version>8</version>
</dependency>

<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc8</artifactId>
    <version>19.3.0.0</version>
</dependency>


<!--mybatisPlus 起步依赖-->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.0</version>
</dependency>

<!-- druid连接池 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.13</version>
</dependency>

<!-- redis-->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## 监控层

```xml
<!-- 健康监测 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- 健康监测客户端 -->
<dependency>
   <groupId>de.codecentric</groupId>
   <artifactId>spring-boot-admin-starter-client</artifactId>
   <version>2.5.2</version>
</dependency>
```

## 工具类

```xml
<!-- hutool -->
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>4.1.1</version>
    <scope>compile</scope>
</dependency>

<!-- lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>

<!-- easyexcel -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>2.2.6</version>
    <scope>compile</scope>
</dependency>

<!-- fastjson -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.72</version>
</dependency>
```

## cloud

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.4.2</version>
</parent>

<!-- spring cloud 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>2021.0.3</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>

<!-- spring cloud alibaba 依赖 -->
<dependency>
   <groupId>com.alibaba.cloud</groupId>
   <artifactId>spring-cloud-alibaba-dependencies</artifactId>
   <version>2021.1</version>
   <type>pom</type>
   <scope>import</scope>
</dependency>

<!--bootstrap 启动器-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
    <version>3.0.4</version>
</dependency>

<!--注册中心客户端-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.1</version>
</dependency>

<!--配置中心-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2021.1</version>
</dependency>

<!--feign-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- LB 扩展 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

## 其他

```xml
<!--velocity代码生成使用模板 -->
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity</artifactId>
    <version>1.7</version>
</dependency>

<!--itext pdf-->
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itextpdf</artifactId>
    <version>5.5.11</version>
</dependency>
```

# 3、常用插件

```xml
<!-- 跳过单元测试 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <skipTests>true</skipTests>
    </configuration>
</plugin>

<!-- 打包插件 -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <fork>true</fork>
    </configuration>
</plugin>
```

# 4、常用标签

## distributionManagement 用来部署到远程仓库

```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <name>Nexus Release Repository</name>
        <url>http://${nexus.ip}:8082/repository/maven-snapshots/</url>
    </snapshotRepository>
    <repository>
        <id>nexus-releases</id>
        <name>Nexus Release Repository</name>
        <url>http://${nexus.ip}:8082/repository/maven-releases/</url>
    </repository>
</distributionManagement>
```

## pluginRepositories

```xml
<pluginRepositories>
    <pluginRepository>
        <id>nexus</id>
        <name>Team Nexus Repository</name>
        <url>http://${nexus.ip}:8082/repository/maven-public/</url>
    </pluginRepository>
</pluginRepositories>
```

## repository 配置远程仓库

```xml
<repositories>
    <repository>
        <id>nexus</id>
        <name>Team Nexus Repository</name>
        <url>http://${nexus.ip}:8082/repository/maven-public/</url>
    </repository>
    <repository>
        <id>thirdparty</id>
        <url>http://${nexus.ip}:8082/repository/3rd_part/</url>
    </repository>
</repositories>
```

## profiles 用来配置当前环境

```xml
spring:
  profiles:
    active: @spring.active@
    
<profiles>
    <profile>
        <id>local</id>
        //配置环境
        <properties>
            <spring.active>local</spring.active>
        </properties>
        <activation>
  		 //激活状态
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
</profiles>
```

# 5、清理垃圾依赖脚本

新建文本文档.bat

```bat
@echo off
echo @Author Fate
echo @Date Mon Jan 24 2020 10:00:56 GMT+0800
:again
set /p REPOSITORY_PATH=Please enter the local your maven warehouse path:
:: Remove all double quotes
set REPOSITORY_PATH=%REPOSITORY_PATH:"=%
if not exist "%REPOSITORY_PATH%" (
   echo Maven warehouse address not found, please try again.
   goto again
)
for /f "delims=" %%i in ('dir /b /s "%REPOSITORY_PATH%\*lastUpdated*"') do (
   del /s /q "%%i"
)
echo Expired files deleted successfully.
pause;
```

