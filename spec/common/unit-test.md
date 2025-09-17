# Unit Test

本專案的資料夾結構使用 Clean Architecture 建構, 不強求 TDD, 但在 Jacoco 內設定 domain service, UseCaseImpl, adapter 和 DB 的方法都需要經過測試:

* domain/service/**
* domain/service/usecase/** 
* infrastructure/adapter/**
* infrastructure/persistence/repo/**

domain service 的方法要囊括 Spec 內的 Scenario
而 controller 在 domain service test 完整的情況下，可以只測試 Happy Path + 1~2 個錯誤情境, 但要確保返回的 ApiResponse 符合預期。

* common package 內的 untils 工具全部都需要測試, 但是不包含在 coverage 內。
* gateway 的 config 不須測試, JwtFilter 則需要測試。

在完成分支推送後, 請確保程式無 Major Issue, Code Coverage 於 Sonar 保持 80% 以上

## 範例

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.casha</groupId>
        <artifactId>casha-backend</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>auth-service</artifactId>
    <packaging>jar</packaging>

    <name>auth-service</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <sonar.coverage.exclusions>
            **/com/casha/auth/application/**,
            **/com/casha/common/**,
            **/AuthApplication.java
        </sonar.coverage.exclusions>
    </properties>

    <dependencies>
...
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.8</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>verify</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                        <configuration>
                            <includes>
                                <include>com/casha/auth/domain/usecase/**</include>
                                <include>com/casha/auth/domain/service/**</include>
                            </includes>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

> 推薦下一篇: [RBAC](/spec/common/rbac.md)