---
title: Gradle发布jar到Gitea的Package Registry
date: 2023-07-14 16:23:17
updated: 2023-07-14 16:23:17
tags:
  - git
  - gradle
  - gitea
---

>版本说明 Gradle 7.6 ，Gitea 1.19.3，SpringBoot 3.x，Java 17

## 创建一个 AccessToken

进入到【个人设置】》【应用】中，在【管理 Access Token】中创建一个 `AccessToken` ，授权范围选中 `package` ，然后点击“生成令牌”即可，令牌只显示一次，记得保存好。

![image-20230714162703559](assets/image-20230714162703559.png)

![image-20230714162719086](assets/image-20230714162719086.png)

![image-20230714162810762](assets/image-20230714162810762.png)

## Cargo 注册中心索引初始化

进入到【个人设置】》【软件包】页面，第一次操作需要在页面中【Cargo 注册中心索引】点击“初始化索引”，后续也可以点击“重建索引”。

![image-20230714163027306](assets/image-20230714163027306.png)

索引创建成功后，在我们的个人仓库列表中会自动生成一个仓库名为 `_cargo-index` 的仓库。

![image-20230714163157461](assets/image-20230714163157461.png)

## Gradle 配置

Maven的配置可参考 [官方说明文档](https://docs.gitea.com/packages/usage/packages/maven) 

在我们的 `build.gradle` 文件中增加如下配置

```groovy
plugins {
    // 发布普通Java项目时引入
    // id 'java-library'
    // 发布一个 dependencies pom 依赖jar时候引入，它不可与 Java 项目同时使用
    // id 'java-platform'
    // 签名插件，在把项目发布到【maven中央仓库】的时候用的到，需要对 jar 进行签名，本项目并未用到。
    // 需要创建一个公钥和私钥，然后把公钥上传到公共密钥服务器
    // https://zhuanlan.zhihu.com/p/164446166
    // https://central.sonatype.org/publish/requirements/gpg/
    // id 'signing'
    // 项目发布插件，在把项目发布到maven仓库的时候需要用到这个插件，具体的用法请看 ${project.dir}/buildSrc/publishing.gradle
    id 'maven-publish'
}
// 定义项目的git仓库地址
def gitRepo = "github.com/houkunlin/xxxxx.git"
publishing {
    // 发布一个 dependencies pom 依赖jar时候引入，它不可与 Java 项目同时使用
    // publications {
    //     gradlePlatform(MavenPublication) {
    //         from components.javaPlatform
    //         artifactId "${group}.dependencies"
    //     }
    // }
    // 发布普通Java项目的配置
    publications {
        library(MavenPublication) {
            from components.java
            pom {
                name = project.name
                packaging = 'jar'
                description = project.description
                url = "https://${gitRepo}"
                // properties = []
                licenses {
                    license {
                        name = 'Mulan Permissive Software License，Version 2'
                        url = 'https://license.coscl.org.cn/MulanPSL2'
                    }
                }
                developers {
                    developer {
                        id = 'houkunlin'
                        name = 'HouKunLin'
                        email = 'houkunlin@aliyun.com'
                    }
                }
                scm {
                    connection = "scm:git://${gitRepo}"
                    developerConnection = "scm:git://${gitRepo}"
                    url = "git://${gitRepo}"
                }
            }
        }
    }
    repositories {
        maven {
            name = "gitea"
            credentials(HttpHeaderCredentials) {
                name = "Authorization"
                // 这里未直接定义 Access Token 令牌值，防止令牌泄露
                // 而是通过把 Gitea Access Token 存入到环境变量中，或者执行命令参数中，或者存入 gradle.properties 文件中
                value = "token ${findProperty("GITEA_TOKEN") ?: System.getenv("GITEA_TOKEN")}"
            }
            authentication {
                header(HttpHeaderAuthentication)
            }
            // 使用 HTTP 是需要设置这个参数
            setAllowInsecureProtocol(true)
            // http://gitea.domain.com/api/packages/{owner}/maven 这个 owner 改成自己的用户名或者组织名
            url "http://gitea.domain.com/api/packages/houkunlin/maven"
        }
    }
}
// jar签名设置
// signing {
//     // 使用 gradle.properties 设置参数，或者在命令行中增加 -Pgpg_private_key= -Pgpg_password= 设置参数
//     // 或者在环境变量中设置相应的环境变量
//     String signingKey = findProperty("gpg_private_key") ?: System.getenv("gpg_private_key")
//     if (signingKey != null) {
//         String signingPassword = findProperty("gpg_password") ?: System.getenv("gpg_password")
//         useInMemoryPgpKeys(signingKey, signingPassword)
//     }
//     sign publishing.publications
//     // sign configurations.archives
// }
```

加入以上配置后，刷新 Gradle 项目，然后就可以通过 `./gradlew build publishing` 编译并发布项目jar了

## 查看已经推送到 Gitea 的 jar

进入到【个人首页】》【软件包】就可以看到刚刚推送的jar包

![image-20230714170719360](assets/image-20230714170719360.png)

点击进去可查看相关的jar信息和maven配置

![image-20230714170905047](assets/image-20230714170905047.png)

![image-20230714170940184](assets/image-20230714170940184.png)

在包的设置中，可以把jar包关联到某个仓库，然后就可以在【仓库】》【软件包】中看到这个jar包了

![image-20230714171008045](assets/image-20230714171008045.png)

