---
title: jenkins迁移记录
top: false
cover: false
toc: true
mathjax: true
date: 2020-06-12 17:30:27
password:
summary:
tags:
categories:
---
# jenkins备份和迁移

## 前言

## jenkins导出镜像

```
$ docker commit jenkins_official jenkins/jenkins:2.208

$ docker export jenkins_official > jenkins_2.208.img

$ scp -P 15555 jenkins_2.208.img centos@192.168.88.180:~/

```
用上述方式导出后，发送到新的机器上时，然后尝试启动，发现不成功，如下：

```
// 在另外一台机器上执行
$ docker import - jenkins/jenkins:2.208 < jenkins_2.208.img

$ docker run -d --name jenkins_official -v /jenkins_home:/home/centos/docker/jenkins_home -v /data/maven:/home/centos/maven/apache-maven-3.6.2 -p 50001:50000 -p 9090:8080 --restart=always jenkins/jenkins:2.208
docker: Error response from daemon: No command specified.
See 'docker run --help'.

```

出现上述问题后，查看整个镜像的描述信息，如下:

```
$ docker inspect b437
[
    {
        "Id": "sha256:b43728bd35394ed1fd5a982fbaad893293cb926f2db00e395ff970a3f40a7653",
        "RepoTags": [
            "jenkins/jenkins:2.208"
        ],
        "RepoDigests": [],
        "Parent": "",
        "Comment": "Imported from -",
        "Created": "2020-05-28T02:00:19.9158622Z",
        "Container": "",
        "ContainerConfig": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": null,
            "Cmd": null,
            "Image": "",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": null
        },
        "DockerVersion": "19.03.5",
        "Author": "",
        "Config": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": null,
            "Cmd": null,
            "Image": "",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": null
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 742766101,
        "VirtualSize": 742766101,
        "GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/d890ad5267d6e09d4c4e6d9264b85a282ffdf619b0f399de94ae685c17f9ba10/merged",
                "UpperDir": "/var/lib/docker/overlay2/d890ad5267d6e09d4c4e6d9264b85a282ffdf619b0f399de94ae685c17f9ba10/diff",
                "WorkDir": "/var/lib/docker/overlay2/d890ad5267d6e09d4c4e6d9264b85a282ffdf619b0f399de94ae685c17f9ba10/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:613397f3accc61821ab378b654c7f09f84dad36d603bd121a355a2d53c569335"
            ]
        },
        "Metadata": {
            "LastTagTime": "2020-05-28T10:00:20.275661145+08:00"
        }
    }
]

```
可以看到镜像的cmd部分，命令为null，En导致启动失败。因此需要更换镜像的导出方式。

在执行docker commit命令后，使用其它的方式导出信息。

```
// 在之前的机器上执行

$ docker save c1746bb41658 > jenkins.tar

$ scp -P 15555 jenkins.tar centos@192.168.88.180:~/

```

传输完成后，在后续机器上导入

```
// 在另一台机器上执行
$ docker load < jenkins.tar
e4b20fcc48f4: Loading layer [==================================================>]  105.6MB/105.6MB
91ecdd7165d3: Loading layer [==================================================>]   24.1MB/24.1MB
73bfa217d66f: Loading layer [==================================================>]  8.005MB/8.005MB
5f3a5adb8e97: Loading layer [==================================================>]  146.4MB/146.4MB
9a11244a7e74: Loading layer [==================================================>]   10.1MB/10.1MB
b18043518924: Loading layer [==================================================>]  3.584kB/3.584kB
2ee490fbc316: Loading layer [==================================================>]  205.6MB/205.6MB
60fe639d301d: Loading layer [==================================================>]  338.9kB/338.9kB
167c0ea1e843: Loading layer [==================================================>]  3.584kB/3.584kB
37e1fe95171e: Loading layer [==================================================>]  9.728kB/9.728kB
65c4c6a6feb6: Loading layer [==================================================>]  868.9kB/868.9kB
811d818689d8: Loading layer [==================================================>]  62.76MB/62.76MB
06e095115ef5: Loading layer [==================================================>]  3.584kB/3.584kB
ed68bc14f08b: Loading layer [==================================================>]  9.728kB/9.728kB
581bcef05992: Loading layer [==================================================>]   5.12kB/5.12kB
94dc46c85541: Loading layer [==================================================>]  3.072kB/3.072kB
87e68b9bebe3: Loading layer [==================================================>]  7.168kB/7.168kB
557f852b13c6: Loading layer [==================================================>]  13.82kB/13.82kB
d9caa2edb930: Loading layer [==================================================>]  199.9MB/199.9MB
Loaded image ID: sha256:c1746bb416581b8a9b81fee02ebb355a681752ffe9d38ba7a5adba02b89d3fc3

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              c1746bb41658        24 hours ago        750MB
gitlab/gitlab-ce    12.2.5_101          3925ebbdc4d6        25 hours ago        1.75GB
gitlab/gitlab-ce    12.2.5              a574a1869b14        4 months ago        1.75GB

$ docker tag  c1746bb41658 jenkins/jenkins:2.208

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
jenkins/jenkins     2.208               c1746bb41658        24 hours ago        750MB
gitlab/gitlab-ce    12.2.5_101          3925ebbdc4d6        25 hours ago        1.75GB
gitlab/gitlab-ce    12.2.5              a574a1869b14        4 months ago        1.75GB

$ docker inspect c1746bb41658
[
    {
        "Id": "sha256:c1746bb416581b8a9b81fee02ebb355a681752ffe9d38ba7a5adba02b89d3fc3",
        "RepoTags": [
            "jenkins/jenkins:2.208"
        ],
        "RepoDigests": [],
        "Parent": "",
        "Comment": "",
        "Created": "2020-05-27T02:18:31.310227817Z",
        "Container": "cbbc3907c1bcbd577722d0dbed4459a9af5b5360e3bedbc03094528b3a0d2841",
        "ContainerConfig": {
            "Hostname": "cbbc3907c1bc",
            "Domainname": "",
            "User": "jenkins",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "50000/tcp": {},
                "8080/tcp": {}
            },
            "Tty": true,
            "OpenStdin": true,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/openjdk-8/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "LANG=C.UTF-8",
                "JAVA_HOME=/usr/local/openjdk-8",
                "JAVA_VERSION=8u232",
                "JAVA_BASE_URL=https://github.com/AdoptOpenJDK/openjdk8-upstream-binaries/releases/download/jdk8u232-b09/OpenJDK8U-jdk_",
                "JAVA_URL_VERSION=8u232b09",
                "JENKINS_HOME=/var/jenkins_home",
                "JENKINS_SLAVE_AGENT_PORT=50000",
                "REF=/usr/share/jenkins/ref",
                "JENKINS_VERSION=2.208",
                "JENKINS_UC=https://updates.jenkins.io",
                "JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental",
                "JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals",
                "COPY_REFERENCE_FILE_LOG=/var/jenkins_home/copy_reference_file.log"
            ],
            "Cmd": null,
            "Image": "jenkins/jenkins",
            "Volumes": {
                "/var/jenkins_home": {}
            },
            "WorkingDir": "",
            "Entrypoint": [
                "/sbin/tini",
                "--",
                "/usr/local/bin/jenkins.sh"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "DockerVersion": "19.03.2",
        "Author": "",
        "Config": {
            "Hostname": "cbbc3907c1bc",
            "Domainname": "",
            "User": "jenkins",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "50000/tcp": {},
                "8080/tcp": {}
            },
            "Tty": true,
            "OpenStdin": true,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/openjdk-8/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "LANG=C.UTF-8",
                "JAVA_HOME=/usr/local/openjdk-8",
                "JAVA_VERSION=8u232",
                "JAVA_BASE_URL=https://github.com/AdoptOpenJDK/openjdk8-upstream-binaries/releases/download/jdk8u232-b09/OpenJDK8U-jdk_",
                "JAVA_URL_VERSION=8u232b09",
                "JENKINS_HOME=/var/jenkins_home",
                "JENKINS_SLAVE_AGENT_PORT=50000",
                "REF=/usr/share/jenkins/ref",
                "JENKINS_VERSION=2.208",
                "JENKINS_UC=https://updates.jenkins.io",
                "JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental",
                "JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals",
                "COPY_REFERENCE_FILE_LOG=/var/jenkins_home/copy_reference_file.log"
            ],
            "Cmd": null,
            "Image": "jenkins/jenkins",
            "Volumes": {
                "/var/jenkins_home": {}
            },
            "WorkingDir": "",
            "Entrypoint": [
                "/sbin/tini",
                "--",
                "/usr/local/bin/jenkins.sh"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 749645095,
        "VirtualSize": 749645095,
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/14e74494b7d5e6f6edade6abcd59118440346fa48d151642f4d6ca84a2763899/diff:/var/lib/docker/overlay2/160ca4b02111a4e91dd7cb72c8d903da556aa85b79121994c71d0f25dafecb6b/diff:/var/lib/docker/overlay2/768dac08d447e865a9456ea862b6fe8242e15bfc127e645e6136665b6e1e50ab/diff:/var/lib/docker/overlay2/5e63415ab060f1df0a4436a609cc277a68fb582ee04ac45479e45502703dbd1f/diff:/var/lib/docker/overlay2/1b3ddec07d1159647ab55af7dc1f413d314a7dfb140ab3cb3a10e73e6547707a/diff:/var/lib/docker/overlay2/d6d40054166e07dc1b81b01caeb373e27ccb2fabd1c438172a026f0b2fd14cee/diff:/var/lib/docker/overlay2/c8a667802affe38898c86cee24c17c911698536de7690cd6c00ba94d6b5a0e32/diff:/var/lib/docker/overlay2/46de4645b03414a8345932dac971c155077749f159bd95121bd94a9c61dbc332/diff:/var/lib/docker/overlay2/8820f1a5bd612cb450024dd06838d53e90bfc6b3905835333c5d086ae4718152/diff:/var/lib/docker/overlay2/1c45197624d037e0c5304be33a89b15f399f8aef5427f0052bd98c8f4e846e4a/diff:/var/lib/docker/overlay2/58108e3d56074702831819adc017cd76bf48c3e381c1926b5506c72d724d5619/diff:/var/lib/docker/overlay2/5aae50ad403940eb6665853bacc95f3b26fd107de435418460dbe5e7aeb2314a/diff:/var/lib/docker/overlay2/cc94aec27d7388468a38efc7d6a9fae4979e4fb9e8d095a4abe24d0b9545366b/diff:/var/lib/docker/overlay2/52e8cfa0b28387eb79384858333e4889fe7fe652c9c0ad11b81840aa9078057d/diff:/var/lib/docker/overlay2/538fe5e1f97fd5db5730c092d7c92cb5e7985d0cceb11fe5c488b935dfc3ec6d/diff:/var/lib/docker/overlay2/3cdfcac471d7bb04d6f1936f704d8c034ce36643f8da440791c1b940c7c0e639/diff:/var/lib/docker/overlay2/81604d5d12d8b8f266f6ec5e4fe17bb09c4be1b8f182737d80b6490b2bbbe602/diff:/var/lib/docker/overlay2/e38dbbc004152e1f7e6a3c325193ae1e9fe3388c853eecf46df8e26c730b6cec/diff",
                "MergedDir": "/var/lib/docker/overlay2/7a8f041bc7472f86d7a15a6faf72dd565b89535f93551470e3ad1084941724e4/merged",
                "UpperDir": "/var/lib/docker/overlay2/7a8f041bc7472f86d7a15a6faf72dd565b89535f93551470e3ad1084941724e4/diff",
                "WorkDir": "/var/lib/docker/overlay2/7a8f041bc7472f86d7a15a6faf72dd565b89535f93551470e3ad1084941724e4/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:e4b20fcc48f4a225fd29ce3b3686cc51042ce1f076d88195b3705b5bb2f38c3d",
                "sha256:91ecdd7165d37f7ac4fb6051632936b95da73973f0261785e162b555458f4592",
                "sha256:73bfa217d66f35520cce481c1fbde51cbdba48113f549c41b68b33eca7bc1f0c",
                "sha256:5f3a5adb8e97eab0570fd2d83a112f270cb2d0b6e32cbb6b6770ebe7a3e9678e",
                "sha256:9a11244a7e74c1be586f2029f6ca6d9359afd5657403eec6fe6cf6658907427d",
                "sha256:b18043518924251be33af630922e6357383dc7e723864c79ba2e347197e5ce95",
                "sha256:2ee490fbc316cec390e0e2409b26d698f4825d673e14cf41055412accd902f1f",
                "sha256:60fe639d301de3b394e6adb800378832c86091e72ac712a5a4415a2bc1fff884",
                "sha256:167c0ea1e843e48675d88bd4d7ead1bf07c859e02c5b81384af4bad31ebcbd35",
                "sha256:37e1fe95171e3eb1b41b50ce512552635aed8879562bb55569ef10a494712cf0",
                "sha256:65c4c6a6feb60e4277a6d46ec7217ca5e89e68eeaa4bf20be2b4a38c58f441d6",
                "sha256:811d818689d836d0d7f247ee56d26518b000738a49685f87b39b104991dc2b05",
                "sha256:06e095115ef52045c10ad6d99d735f941eea9987fc01cd695da9589e66f93985",
                "sha256:ed68bc14f08ba0dc6c4b318819b10ca57145c63ac25cc9d95864bd0b3a9952bb",
                "sha256:581bcef0599240aa369b87716e81be3f70487b08476da5b1850e5fd303c955a4",
                "sha256:94dc46c85541c011448c5871401d83bec2589ebf5a4d28f6e00c70bea18d7af0",
                "sha256:87e68b9bebe3ec7ca616656bdba5c24b17ea1ac4795a86c5bf64e4001ed24e96",
                "sha256:557f852b13c6f82a43724259d13b6c36dff30f29b1b9493f586be1a061820ac2",
                "sha256:d9caa2edb9309e9e78275238d80c17ccce4d0a74dc9c2cef623f3f8d57945e7e"
            ]
        },
        "Metadata": {
            "LastTagTime": "2020-05-28T10:18:14.372683313+08:00"
        }
    }
]

```

这样导出再导入的时候，Entrypoint就不为null了，重新启动jenkins尝试，如下：

```
$ docker run -d --name jenkins_official -v /jenkins_home:/home/centos/docker/jenkins_home -v /data/maven:/home/centos/maven/apache-maven-3.6.2 -p 50001:50000 -p 9090:8080 --restart=always jenkins/jenkins:2.208

```

## 查看jenkins容器的启动命令

使用插件查看

启动命令
docker run -d --name jenkins_official -v /jenkins_home:/home/centos/docker/jenkins_home -v /data/maven:/home/centos/maven/apache-maven-3.6.2 -p 50001:50000/tcp -p 9090:8080/tcp --restart always -e 'LANG=C.UTF-8' jenkins/jenkins

## 创建运行的文件夹

maven组件、创建文件夹，拷贝

## 导入镜像到新机器中并启动



## 初始化设置

按照之前的jenkins设置为一致，不安装插件，直接跳过。

配置插件安装地址

安装thinbackup插件

初始化设置thinbackup

## 备份原有镜像内容

使用thinbackup的备份方式，点击BackupNow，将备份后的文件夹进行压缩，传输到目标机器中。

## 重新覆盖新机器中的镜像信息

通过thinbackup进行恢复，拷贝文件到文件夹中

文件解压，回到页面上点击restore

**注意：** 在jenkins恢复时，在网页上无法判断是否恢复完毕，甚至存在浏览器不停转圈的情况。

面对这种情况，可以直接从docker镜像的日志中查看恢复进度，如下：

```
$ docker logs -f --tail=100 jenkins_official

2020-06-29 01:25:13.774+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'ssh-credentials'.
2020-06-29 01:25:13.779+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'workflow-job'.
2020-06-29 01:25:13.784+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'workflow-basic-steps'.
2020-06-29 01:25:13.788+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'jsch'.
2020-06-29 01:25:13.789+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'github-api'.
2020-06-29 01:25:13.790+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'gitlab-merge-request-jenkins'.
2020-06-29 01:25:13.795+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: Restore plugin 'workflow-cps'.
2020-06-29 01:25:13.796+0000 [id=71]    INFO    o.j.h.p.t.restore.HudsonRestore#installPlugin: R

.................................


2020-06-29 01:29:25.522+0000 [id=76]    INFO    h.model.UpdateCenter$DownloadJob#run: Starting the installation of pubsub-light on behalf of admin
2020-06-29 01:29:26.737+0000 [id=76]    INFO    h.m.UpdateCenter$UpdateCenterConfiguration#download: Downloading pubsub-light
2020-06-29 01:29:27.046+0000 [id=71]    INFO    o.j.h.p.t.ThinBackupMgmtLink#doRestore: Restore finished.

```

直到日志中出现*Restore finished.*即为恢复完成！

## 迁移后的剩余问题

1. jenkins的URL地址进行变更，变更为目标服务器的

重新设置jenkins的URL地址

2. 账号凭据等失效的问题

重建账号信息

3. 本地maven工具的失效的问题

重新拷贝maven工具到容器中，然后在jenkins中重新配置

4. pom文件执行错误的问题:构建时，使用了项目根目录下的pom.xml文件，并不是构建的对应目录下的文件

报错信息如下：

```
09:51:02 Parsing POMs
09:51:02 using settings config with name Maven Local Settings
09:51:02 Replacing all maven server entries not found in credentials list is true
09:51:02 ERROR: Failed to parse POMs
09:51:02 org.apache.maven.project.ProjectBuildingException: Some problems were encountered while processing the POMs:
09:51:02 [ERROR] Non-resolvable import POM: Failure to transfer org.springframework.cloud:spring-cloud-dependencies:pom:Hoxton.SR1 from https://repo.maven.apache.org/maven2 was cached in the local repository, resolution will not be reattempted until the update interval of central has elapsed or updates are forced. Original error: Could not transfer artifact org.springframework.cloud:spring-cloud-dependencies:pom:Hoxton.SR1 from/to central (https://repo.maven.apache.org/maven2): Connect to repo.maven.apache.org:443 [repo.maven.apache.org/151.101.40.215] failed: Connection timed out (Connection timed out) @ com.testsoft:test-utils:1.0.0-SNAPSHOT, /var/jenkins_home/workspace/test-dict-develop/pom.xml, line 82, column 25
09:51:02 [ERROR] Non-resolvable import POM: Failure to transfer com.alibaba.cloud:spring-cloud-alibaba-dependencies:pom:2.2.1.RELEASE from https://repo.maven.apache.org/maven2 was cached in the local repository, resolution will not be reattempted until the update interval of central has elapsed or updates are forced. Original error: Could not transfer artifact com.alibaba.cloud:spring-cloud-alibaba-dependencies:pom:2.2.1.RELEASE from/to central (https://repo.maven.apache.org/maven2): Connect to repo.maven.apache.org:443 [repo.maven.apache.org/151.101.40.215] failed: Connection timed out (Connection timed out) @ com.testsoft:test-utils:1.0.0-SNAPSHOT, /var/jenkins_home/workspace/test-dict-develop/pom.xml, line 89, column 25
09:51:02 [ERROR] 'dependencies.dependency.version' for org.springframework.cloud:spring-cloud-commons:jar is missing. @ com.testsoft:test-data-dictionary-server:[unknown-version], /var/jenkins_home/workspace/test-dict-develop/test-data-dictionary-server/pom.xml, line 29, column 21
09:51:02 [ERROR] 'dependencies.dependency.version' for org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:jar is missing. @ com.testsoft:test-data-dictionary-server:[unknown-version], /var/jenkins_home/workspace/test-dict-develop/test-data-dictionary-server/pom.xml, line 81, column 21
09:51:02 [ERROR] 'dependencies.dependency.version' for com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-config:jar is missing. @ com.testsoft:test-data-dictionary-server:[unknown-version], /var/jenkins_home/workspace/test-dict-develop/test-data-dictionary-server/pom.xml, line 85, column 21
09:51:02 [ERROR] 'dependencies.dependency.version' for org.springframework.cloud:spring-cloud-starter-sleuth:jar is missing. @ com.testsoft:test-utils:1.0.0-SNAPSHOT, /var/jenkins_home/workspace/test-dict-develop/pom.xml, line 74, column 21
09:51:02 
09:51:02 	at org.apache.maven.project.DefaultProjectBuilder.build(DefaultProjectBuilder.java:383)
09:51:02 	at hudson.maven.MavenEmbedder.buildProjects(MavenEmbedder.java:370)
09:51:02 	at hudson.maven.MavenEmbedder.readProjects(MavenEmbedder.java:340)
09:51:02 	at hudson.maven.MavenModuleSetBuild$PomParser.invoke(MavenModuleSetBuild.java:1329)
09:51:02 	at hudson.maven.MavenModuleSetBuild$PomParser.invoke(MavenModuleSetBuild.java:1126)
09:51:02 	at hudson.FilePath.act(FilePath.java:1075)
09:51:02 	at hudson.FilePath.act(FilePath.java:1058)
09:51:02 	at hudson.maven.MavenModuleSetBuild$MavenModuleSetBuildExecution.parsePoms(MavenModuleSetBuild.java:987)
09:51:02 	at hudson.maven.MavenModuleSetBuild$MavenModuleSetBuildExecution.doRun(MavenModuleSetBuild.java:691)
09:51:02 	at hudson.model.AbstractBuild$AbstractBuildExecution.run(AbstractBuild.java:504)
09:51:02 	at hudson.model.Run.execute(Run.java:1853)
09:51:02 	at hudson.maven.MavenModuleSetBuild.run(MavenModuleSetBuild.java:543)
09:51:02 	at hudson.model.ResourceController.execute(ResourceController.java:97)
09:51:02 	at hudson.model.Executor.run(Executor.java:427)
```

处理方式：在maven命令部分，添加**-U*参数，确保拉取时拉取最新的jar包

5. 迁移以后，jenkins中的ssh连接失效

报错信息：

```
com.jcraft.jsch.JSchException: SSH_MSG_DISCONNECT: 2 Too many authentication failures 
13:18:37 	at com.jcraft.jsch.Session.read(Session.java:1004)
13:18:37 	at com.jcraft.jsch.UserAuthPassword.start(UserAuthPassword.java:91)
13:18:37 	at com.jcraft.jsch.Session.connect(Session.java:470)
13:18:37 	at org.jvnet.hudson.plugins.CredentialsSSHSite.createSession(CredentialsSSHSite.java:132)
13:18:37 	at org.jvnet.hudson.plugins.CredentialsSSHSite.executeCommand(CredentialsSSHSite.java:208)
13:18:37 	at org.jvnet.hudson.plugins.SSHBuilder.perform(SSHBuilder.java:104)
13:18:37 	at hudson.tasks.BuildStepMonitor$1.perform(BuildStepMonitor.java:20)
13:18:37 	at hudson.model.AbstractBuild$AbstractBuildExecution.perform(AbstractBuild.java:741)
13:18:37 	at hudson.maven.MavenModuleSetBuild$MavenModuleSetBuildExecution.build(MavenModuleSetBuild.java:946)
13:18:37 	at hudson.maven.MavenModuleSetBuild$MavenModuleSetBuildExecution.doRun(MavenModuleSetBuild.java:896)
13:18:37 	at hudson.model.AbstractBuild$AbstractBuildExecution.run(AbstractBuild.java:504)
13:18:37 	at hudson.model.Run.execute(Run.java:1853)
13:18:37 	at hudson.maven.MavenModuleSetBuild.run(MavenModuleSetBuild.java:543)
13:18:37 	at hudson.model.ResourceController.execute(ResourceController.java:97)
13:18:37 	at hudson.model.Executor.run(Executor.java:427)
13:18:37 Build step 'Execute shell script on remote host using ssh' marked build as failure

```

处理方式：SSH密钥过期的问题，重新添加密钥即可

## 关于jenkins中容器过大的问题调整

backup内容过大的问题-->删除过多的备份调整备份策略    31G  --> 1.1G

1. 修改定时任务

0 23 * * 1

每周一23点执行一次备份

2.  删除多余的备份

jobs-->产出物的积累问题，产出物jar包              7.9G   --> 7.9G

1. 产出物jar包占用大小，后续构建完成后推送到nexus3镜像，并且删除本地的jar包

2. 构建物的保留策略，需要重新制定，[参考链接](https://blog.csdn.net/liliwang90/article/details/104690491)

workspace--> 排序，查看最大与最小的内容           14G    --> 9.6G

1. 项目拆分，存在重复的情况

2. 前端依赖问题，先备份前端依赖，再逐步删除依赖信息

3. 快速批量删除Jenkins构建清理磁盘空间并按参数保留最近构建

## 总结

jenkins备份相对不那么好做，由于历史的原因，一些配置的设定上并没有那么合理，需要在迁移完成后重新进行设置，并且一定构建一支工程进行测试，确保测试无误后，投入使用。

## 参考链接

* [快速批量删除Jenkins构建清理磁盘空间并按参数保留最近构建](https://blog.csdn.net/liliwang90/article/details/104690491)

## 问题

1. 时间更新

2. 统一构建账户的调整

3. 全局变量的参数化配置