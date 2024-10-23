# SpringBoot 分层构建 docker image

SpringBoot 从 2.3 版本开始，允许将应用拆分成不同的 layers, 帮助我们更加高效地构建 docker image.

### 使用 SpringBoot 自带的 maven 插件, configuration 节点添加 layers 配置支持分层 ：

```xml

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
        <mainClass>com.example.multilayers.MultiLayersApplication</mainClass>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
    <executions>
        <execution>
            <id>repackage</id>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### maven 打包后的产物提取对应的层, 默认包含四层.

| Syntax                | Description      |
|-----------------------|------------------|
| dependencies          | 正式依赖的 jar 包      |
| spring-boot-loader    | SpringBoot 加载启动类 |
| snapshot-dependencies | 快照依赖             |
| application           | 应用代码             |

```shell
# mvn 打包
mvn -B clean package

# 提取
java -Djarmode=layertools -jar your-app-name.jar extract
```

### docker image 构建,将不经常更改的层级保持在底部,将经常更改的层级应放在顶部.

Dockerfile 如下
```shell
FROM openjdk:17.0.2 AS builder
WORKDIR /application
ARG JAR_FILE=multi-layers-start/target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM openjdk:17.0.2
WORKDIR /application
COPY --from=builder multi-layers-start/target/application/dependencies/ ./
COPY --from=builder multi-layers-start/target/application/spring-boot-loader/ ./
COPY --from=builder multi-layers-start/target/application/snapshot-dependencies/ ./
COPY --from=builder multi-layers-start/target/application/application/ ./
ENTRYPOINT ["java","org.springframework.boot.loader.JarLauncher"]
```

### 构建 docker image
```shell
docker build . --tag multi-layers:latest
```

### 在不修改pom的情况下，打包构建出来的镜像层只动了 application 层,其他层都使用的是缓存
![img.png](/doc/img.png)