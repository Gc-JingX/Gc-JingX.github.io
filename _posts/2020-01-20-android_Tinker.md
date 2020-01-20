---
layout: post
title:  "Tinker 热修复"
categories: Tinker
tags:  android
author: GaryLiang
---

* content
{:toc}


## 1.准备工作

Tinker 官网  http://tinkerpatch.com/   

花几分钟注册个账号  新建一个APP  APP名称建议和工程名称一样，方便管理

https://img-blog.csdn.net/20180313180508257

## 2.SDK接入
> 1）、配置Tinker版本信息
> 
>     我们使用配置文件去配置版本信息，易于统一版本和后面更换版本
> 
>      编辑根目录的gradle.properties，加入
> 
>   * TINKER_VERSION=1.9.2
>   * TINKERPATCH_VERSION=1.2.2
 

>2）.配置app目录下的build.gradle文件

```css
apply plugin: 'com.android.application'
 
apply from: 'tinkerpatch.gradle'
android {
    compileSdkVersion 26
    defaultConfig {
        applicationId "com.feijin.sleep"
        minSdkVersion 19
        targetSdkVersion 26
        versionCode 8
        versionName "0.0.8"
        multiDexEnabled true
        multiDexKeepProguard file("tinkerMultidexKeep.pro")
        //keep specific classes using proguard syntax
    }
    signingConfigs {
        release {//发布版本的签名配置
            storeFile file('../keystore/gary.jks')
            keyAlias 'gary'
            storePassword '12345678'
            keyPassword '12345678'
        }
        debug {//调试版本的签名配置
            storeFile file('../keystore/gary.jks')
            keyAlias 'gary'
            storePassword '12345678'
            keyPassword '12345678'
        }
    }
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            signingConfig signingConfigs.debug
 
            proguardFiles 'proguardRules.pro', getDefaultProguardFile('proguard-android.txt')
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
    }
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
}
 
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:26.1.0'
    implementation 'com.android.support.constraint:constraint-layout:1.0.2'
    testImplementation 'junit:junit:4.12'
    implementation "com.android.support:multidex:1.0.2"
    //若使用annotation需要单独引用,对于tinker的其他库都无需再引用
    annotationProcessor("com.tinkerpatch.tinker:tinker-android-anno:${TINKER_VERSION}") {
        changing = true
    }
    compileOnly("com.tinkerpatch.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    implementation("com.tinkerpatch.sdk:tinkerpatch-android-sdk:${TINKERPATCH_VERSION}") {
        changing = true
    }
 
}


```

>3） 配置 根目录下的build.gradle 文件

```css
// Top-level build file where you can add configuration options common to all sub-projects/modules.
buildscript {
    repositories {
        //mavenLocal()
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.1'
        //无需再单独引用tinker的其他库
        classpath "com.tinkerpatch.sdk:tinkerpatch-gradle-plugin:${TINKERPATCH_VERSION}"
    }
}
 
allprojects {
    repositories {
        //mavenLocal()
        google()
        jcenter()
    }
}
// See http://blog.joda.org/2014/02/turning-off-doclint-in-jdk-8-javadoc.html.
if (JavaVersion.current().isJava8Compatible()) {
    allprojects {
        tasks.withType(Javadoc) {
            options.addStringOption('Xdoclint:none', '-quiet')
        }
    }
}
 
subprojects {
    tasks.withType(JavaCompile) {
        sourceCompatibility = JavaVersion.VERSION_1_7
        targetCompatibility = JavaVersion.VERSION_1_7
    }
}
 
task clean(type: Delete) {
    delete rootProject.buildDir
}


```

>4） 在项目根目录新建tinkerpatch.gradle文件 代码如下


```css
apply plugin: 'tinkerpatch-support'
 
/**
 * TODO: 请按自己的需求修改为适应自己工程的参数
 */
def bakPath = file("${buildDir}/bakApk/")
def baseInfo = "app-0.0.8-0313-16-40-30"
def variantName = "release"
 
/**
 * 对于插件各参数的详细解析请参考
 * http://tinkerpatch.com/Docs/SDK
 */
tinkerpatchSupport {
    /** 可以在debug的时候关闭 tinkerPatch **/
    /** 当disable tinker的时候需要添加multiDexKeepProguard和proguardFiles,
        这些配置文件本身由tinkerPatch的插件自动添加，当你disable后需要手动添加
        你可以copy本示例中的proguardRules.pro和tinkerMultidexKeep.pro,
        需要你手动修改'tinker.sample.android.app'本示例的包名为你自己的包名, com.xxx前缀的包名不用修改
     **/
    tinkerEnable = true
    reflectApplication = true
    /**
     * 是否开启加固模式，只能在APK将要进行加固时使用，否则会patch失败。
     * 如果只在某个渠道使用了加固，可使用多flavors配置
     **/
    protectedApp = false
    /**
     * 实验功能
     * 补丁是否支持新增 Activity (新增Activity的exported属性必须为false)
     **/
    supportComponent = true
 
    autoBackupApkPath = "${bakPath}"
 
    appKey = "0d36f0530a76e76c"
 
    /** 注意: 若发布新的全量包, appVersion一定要更新 **/
    appVersion = "0.0.8"
 
    def pathPrefix = "${bakPath}/${baseInfo}/${variantName}/"
    def name = "${project.name}-${variantName}"
 
    baseApkFile = "${pathPrefix}/${name}.apk"
    baseProguardMappingFile = "${pathPrefix}/${name}-mapping.txt"
    baseResourceRFile = "${pathPrefix}/${name}-R.txt"
 
    /**
     *  若有编译多flavors需求, 可以参照： https://github.com/TinkerPatch/tinkerpatch-flavors-sample
     *  注意: 除非你不同的flavor代码是不一样的,不然建议采用zip comment或者文件方式生成渠道信息（相关工具：walle 或者 packer-ng）
     **/
}
 
/**
 * 用于用户在代码中判断tinkerPatch是否被使能
 */
android {
    defaultConfig {
        buildConfigField "boolean", "TINKER_ENABLE", "${tinkerpatchSupport.tinkerEnable}"
    }
}
 
/**
 * 一般来说,我们无需对下面的参数做任何的修改
 * 对于各参数的详细介绍请参考:
 * https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
 */
tinkerPatch {
    ignoreWarning = false
    useSign = true
    dex {
        dexMode = "jar"
        pattern = ["classes*.dex"]
        loader = []
    }
    lib {
        pattern = ["lib/*/*.so"]
    }
 
    res {
        pattern = ["res/*", "r/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
        ignoreChange = []
        largeModSize = 100
    }
 
    packageConfig {
    }
    sevenZip {
        zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
//        path = "/usr/local/bin/7za"
    }
    buildConfig {
        keepDexApply = false
    }
}





```

>5） 配置application 文件

https://img-blog.csdn.net/20180313180726163


## 3、生成基准包

https://img-blog.csdn.net/20180313180817581

双击assembleRelease 生成一个文件  也就是基准包，通俗点来说 就是当前发布出去的包，而且还带有bug的 而这个文件在哪里可以获取到呢，看下图

https://img-blog.csdn.net/20180313180901984

在本地电脑里 找到这个文件 安装到手机 运行一次 然后杀掉


## 4、添加修复代码 随便写一些就好了，做一下测试，然后修改 tinkerpatch.gradle 类 

```css
/**
 * TODO: 请按自己的需求修改为适应自己工程的参数
 */
def bakPath = file("${buildDir}/bakApk/")
def baseInfo = "app-0.0.8-0313-16-40-30"
def variantName = "release"
/** 注意: 若发布新的全量包, appVersion一定要更新 **/
appVersion = "0.0.8"
 

```

注意的是  appVersion 要和versionName 一样 然后执行最后一步 如下图

https://img-blog.csdn.net/20180313180926990

点击thinkerPathRelease  等待片刻后 会生成一个补丁包如下图

https://img-blog.csdn.net/20180313181109812

##  5.发布补丁包

https://img-blog.csdn.net/20180313181210134

https://img-blog.csdn.net/20180313181133844



此时我们只要找到刚才那个patch_signed_7zip.apk 文件放进去就好了。

等待片刻 作者运气可能好 1分钟后，彻底杀掉APP 再次打开就可以看到打印log 拉取文件信息了，如果没看到拉取信息，有个很神奇的操作，就是点击屏幕，会一直打印log 

最后我们来看看最终效果

 



