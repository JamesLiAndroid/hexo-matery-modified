---
title: 关于maven deploy出现连接超时的情况
top: false
cover: false
toc: true
mathjax: true
date: 2021-06-15 11:01:35
password:
summary:
tags:
categories:
---

# 关于maven deploy命令出现连接超时的情况解决

## 问题描述

在执行maven的编译构建操作后，将项目的公共依赖推送到甲方远端的maven仓库地址中。我首先在idea中进行操作，错误信息如下：

```
Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy (default-deploy) on project basestage-starter-web: Failed to retrieve remote metadata com.company:basestage-starter-web:0.0.1-SNAPSHOT/maven-metadata.xml: Could not transfer metadata com.company:basestage-starter-web:0.0.1-SNAPSHOT/maven-metadata.xml from/to my-company-snapshots (http://maven.company.com:8081/repository/maven-snapshots/): Connect to maven.company.com:8081 [maven.company.com/192.16.0.10] failed: Connection timed out: connect

```

连续推送多次，问题如前。于是我又尝试使用命令行进行推送:

```
mvn deploy -DskipTests

...
...
...

Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy (default-deploy) on project basestage-starter-web: Failed to retrieve remote metadata com.company:basestage-starter-web:0.0.1-SNAPSHOT/maven-metadata.xml: Could not transfer metadata com.company:basestage-starter-web:0.0.1-SNAPSHOT/maven-metadata.xml from/to my-company-snapshots (http://maven.company.com:8081/repository/maven-snapshots/): Connect to maven.company.com:8081 [maven.company.com/192.16.0.10] failed: Connection timed out: connect

```

**注意：**甲方的maven仓库地址为*maven.baddad.com:8081*，我们公司的maven地址为*maven.company.com:8081*。

## 尝试解决：排除网络问题

首先看网络问题，在公司使用vpn的方式进行推送，发现始终连不上甲方的网络环境，无法推送到甲方的nexus中。

其次在甲方的内网环境下，还是存在上述的依赖推送问题。不论是wifi环境下授权还是在内网服务器上操作，始终存在该问题。

## 尝试解决：更换settings.xml配置文件

更换我们自己为甲方定制的settings.xml文件，具体内容如下：

```xml

<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>D:/Dev/Java/repository</localRepository>
  
  <pluginGroups>
        <pluginGroup>org.apache.maven.plugins</pluginGroup>
  </pluginGroups>

  <proxies>

  </proxies>

  <servers>
    <server>
        <id>baddad-cloud-releases</id>
        <username>admin</username>
        <password>admin</password>
    </server>
    <server>
        <id>baddad-cloud-snapshots</id>
        <username>admin</username>
        <password>admin</password>
    </server>
    <server>
        <id>baddad-cloud-public</id>
        <username>admin</username>
        <password>admin</password>
    </server>
  </servers>

  <mirrors>
   
  </mirrors>

  <profiles>
        <profile>
            <id>baddad-cloud-nexus</id>
            <activation>
                <jdk>1.8</jdk>
            </activation>
            <properties>
                <maven.compiler.source>1.8</maven.compiler.source>
                <maven.compiler.target>1.8</maven.compiler.target>
                <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
            </properties>
            <repositories>
                <repository>
                    <id>baddad-cloud-public</id>
                    <url>http://maven.baddad.com:8081/repository/maven-public/</url>
                    <releases><enabled>true</enabled></releases>
                    <snapshots><enabled>false</enabled></snapshots>
                </repository>
                <repository>
                    <id>baddad-cloud-snapshots</id>
                    <url>http://maven.baddad.com:8081/repository/maven-snapshots/</url>
                    <releases><enabled>false</enabled></releases>
                    <snapshots><enabled>true</enabled></snapshots>
                </repository>
                <repository>
                    <id>baddad-cloud-release</id>
                    <url>http://maven.baddad.com:8081/repository/maven-release/</url>
                    <releases><enabled>true</enabled></releases>
                    <snapshots><enabled>false</enabled></snapshots>
                </repository>
            </repositories>

            <pluginRepositories>
                <pluginRepository>
                        <id>baddad-cloud-public</id>
                        <url>http://maven.baddad.com:8081/repository/maven-public/</url>
                        <releases><enabled>false</enabled></releases>
                        <snapshots><enabled>false</enabled></snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>baddad-cloud-nexus</activeProfile>
  </activeProfiles>

</settings>

```

更换后在idea中进行设置，如下：

![](maven在idea中的配置.png)

再进行尝试，问题如前。通过命令行指定settings.xml文件尝试，如下：

```
$ mvn -s 'D:\settings\maven\settings-jiafang.xml' clean install deploy 

...
...
...

```

问题如前。

## 尝试解决：检查项目中是否包含外部的maven仓库地址并改正

到这一步才想起来，前面错误中的nexus依赖仓库地址始终指向的我们公司自己的nexus3地址信息，而在maven的配置信息setttings.xml文件中，并未存在这个地址，在这种情况下推断，项目中存在我们公司自己的nexus3地址，于是查找项目的pom.xml文件如下：

```xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.test</groupId>
    <artifactId>basetest</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>basestage</name>
    <description>测试项目</description>
    <packaging>pom</packaging>


    <modules>
        ........
    </modules>

    <properties>
       .......
    </properties>

    <dependencyManagement>
        .......
    </dependencyManagement>

    <dependencies>
        .......
    </dependencies>

    <build>
        .......
    </build>

    <distributionManagement>
        <repository>
            <id>my-company-releases</id>
            <name>test releases</name>
            <url>http://maven.company.com:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>my-company-snapshots</id>
            <name>test snapshots</name>
            <url>http://maven.company.com:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <repositories>
        <repository>
            <id>my-company-releases</id>
            <name>test releases</name>
            <url>http://maven.company.com:8081/repository/maven-releases/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>false</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
        <repository>
            <id>my-company-snapshots</id>
            <name>test snapshots</name>
            <url>http://maven.company.com:8081/repository/maven-snapshots/</url>
            <releases>
                <enabled>false</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>

    <pluginRepositories>
        <pluginRepository>
            <id>public</id>
            <name>aliyun nexus</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>

</project>

```

显然这个地址指向不正确，于是修改如下：

```xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.test</groupId>
    <artifactId>basetest</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>basestage</name>
    <description>测试项目</description>
    <packaging>pom</packaging>


    <modules>
        ........
    </modules>

    <properties>
       .......
    </properties>

    <dependencyManagement>
        .......
    </dependencyManagement>

    <dependencies>
        .......
    </dependencies>

    <build>
        .......
    </build>

    <distributionManagement>
        <repository>
            <id>my-company-releases</id>
            <name>test releases</name>
            <url>http://maven.baddad.com:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>my-company-snapshots</id>
            <name>test snapshots</name>
            <url>http://maven.baddad.com:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <repositories>
        <repository>
            <id>my-company-releases</id>
            <name>test releases</name>
            <url>http://maven.baddad.com:8081/repository/maven-releases/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>false</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
        <repository>
            <id>my-company-snapshots</id>
            <name>test snapshots</name>
            <url>http://maven.baddad.com:8081/repository/maven-snapshots/</url>
            <releases>
                <enabled>false</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>

    <pluginRepositories>
        <pluginRepository>
            <id>public</id>
            <name>aliyun nexus</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>

</project>

```

这样再次测试deploy操作，得到一个401的结果，如下：

```

[ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy (default-deploy) on project xbnjava: Failed to deploy artifacts: Could not transfer artifact com.github.aliteralmind:xbnjava:pom:0.1.2 from/to sonatype-nexus-staging (https://oss.sonatype.org/service/local/staging/deploy/maven2/): Failed to transfer file: https://oss.sonatype.org/service/local/staging/deploy/maven2/com/github/aliteralmind/xbnjava/0.1.2/xbnjava-0.1.2.pom. Return code is: 401, ReasonPhrase: Unauthorized. -> [Help 1]

```

也就是说，目前地址修改正确了，但是登录信息无法进行对应，导致无法进行推送。

## 尝试解决：检查settings.xml和项目中的pom.xml文件中授权信息是否对应

根据第三步的信息，检查settings.xml文件和项目中pom.xml的对应关系，该对应关系决定着是否能正常从maven远程仓库中获得文件更新，在pom.xml中进行修改如下：

```xml

    <distributionManagement>
        <repository>
            <id>baddad-cloud-releases</id>
            <name>test releases</name>
            <url>http://maven.baddad.com:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>baddad-cloud-snapshots</id>
            <name>test snapshots</name>
            <url>http://maven.baddad.com:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <repositories>
        <repository>
            <id>baddad-cloud-releases</id>
            <name>test releases</name>
            <url>http://maven.baddad.com:8081/repository/maven-releases/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>false</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
        <repository>
            <id>baddad-cloud-snapshots</id>
            <name>test snapshots</name>
            <url>http://maven.baddad.com:8081/repository/maven-snapshots/</url>
            <releases>
                <enabled>false</enabled>
                <updatePolicy>always</updatePolicy>
            </releases>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>

```

将pom文件中的id信息，修改为同settings.xml文件中相同的id信息。这样在访问nexus3镜像仓库时，就能匹配settings.xml中的授权信息，也就能正常登录并进行deploy操作！

## 总结

1. settings.xml中的id信息一定要和pom.xml文件中设置的远程仓库地址的id要一一对应，且内部设置信息相同。重要的是，settings.xml中授权信息的id也要和settings.xml中的repository下的id信息相同！

2. 注意先排查网络情况，一般网络情况不通是最大的问题。