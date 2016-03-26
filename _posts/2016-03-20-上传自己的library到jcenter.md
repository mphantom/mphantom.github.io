---
layout: post
title: Library 上传到Jcenter
---

首先你先需要一个bintray的帐号，申请之类的不在这篇文章的范围之内。
对我自己而言，我上传到Jcenter一般是直接在android studio上传的。  
而上传到jcenter需要两个插件  

    classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.0'
    classpath 'com.github.dcendents:android-maven-gradle-plugin:1.3'

接下来在你要上传的项目中使用这两个插件  

    apply plugin: 'com.github.dcendents.android-maven'
    apply plugin: 'com.jfrog.bintray'


最后填写一些配置信息(有些信息相对重要请填写到local.properties中)  

    def siteUrl = 'https://github.com/xxxx'   // 项目的主页
    def gitUrl = 'git@github.com:xxxx.git'   // Git仓库的url
    group = "xxx.xxx.xxx            // 一般填你唯一的包名
    install {
        repositories.mavenInstaller {
            // This generates POM.xml with proper parameters
            pom {
                project {
                    packaging 'aar'
                    name 'XXX'     //项目的描述 你可以多写一点
                    url siteUrl
                    // Set your license
                    licenses {
                        license {
                            name 'The Apache Software License, Version 2.0'
                            url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                        }
                    }
                    developers {
                        developer {
                            id 'xxx'        //填写的一些基本信息
                            name 'xxxx'
                            email 'xxx'
                        }
                    }
                    scm {
                        connection gitUrl
                        developerConnection gitUrl
                        url siteUrl
                    }
                }
            }
        }
    }
    task sourcesJar(type: Jar) {
        from android.sourceSets.main.java.srcDirs
        classifier = 'sources'
    }
    task javadoc(type: Javadoc) {
        source = android.sourceSets.main.java.srcDirs
        classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
    }
    task javadocJar(type: Jar, dependsOn: javadoc) {
        classifier = 'javadoc'
        from javadoc.destinationDir
    }
    artifacts {
        archives javadocJar
    archives sourcesJar
    }
        Properties properties = new Properties()
        properties.load(project.rootProject.file('local.properties').newDataInputStream())
        bintray {
            user = properties.getProperty("bintray.user")
            key = properties.getProperty("bintray.apikey")
            configurations = ['archives']
            pkg {
                repo = "maven"
                name = "ControlBtn"    //发布到JCenter上的项目名字
                websiteUrl = siteUrl
                vcsUrl = gitUrl
                licenses = ["Apache-2.0"]
                publish = true
            }
    }

上传到自己的Bintray使用  

    ./gradlew install
    ./gradlew bintrayUpload