# APK瘦身

[TOC]

## 一、理论

### 1.1 为什么要瘦身

* 安装包变大，导致很多用户不愿意安装更新；
* 安装包变大，导致新用户不愿意下载；
* 安装包变大，流量使用增多，增加其它边际成本。

### 1.2 `APK`组成内容

| 属性名                | 类型                                                         |
| --------------------- | ------------------------------------------------------------ |
| `lib/`                | 存放`so`文件，有`armeabi`、`armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64`、`mips`，一般只需要支持`armeabi`与`x86`的架构即可 |
| `res/`                | 存放编译后的资源文件，例如：`drawable`、`layout`等等         |
| `assets/`             | 应用程序的资源，应用程序可以使用`AssetManager`来检索该资源   |
| `META-INF`            | 该文件夹一般存在于已经签名的`APK`中，它包含了`APK`中所有文件的签名摘要等信息 |
| `classes(n).dex`      | `classes`文件是`Java Class`，被`DEX`编译后可供`Dalvik/ART`虚拟机所理解的文件格式 |
| `resources.arsc`      | 编译后的二进制资源文件                                       |
| `AndroidManifest.xml` | `Android`的清单文件，格式为`AXML`，用于描述应用程序的名称、版本、所需权限、注册的四大组件 |

### 1.3 `APK`存储格式

 `aapt`

```bash
aapt l -v xxx.apk
```

![image](https://github.com/tianyalu/NeAppThining/raw/master/show/aapt_format.png)

## 二、 资源优化

### 2.1 图片资源的优化

#### 2.1.1 图片选择顺序

图片选择顺序如下：

> 1. 首先选择`VD`；
> 2. 其次选择`WebP`；
> 3. 再次如果有透明通道等选择`PNG`；
> 4. 最后选择`JPG`.

![image](https://github.com/tianyalu/NeAppThining/raw/master/show/picture_choose_sequence.png)

#### 2.1.2 优化方式

* **使用矢量图**：①矢量图片只需要放置一份；②图片太大绘制时间长，制作复杂度高；
* **使用`WebP`**：`WebP`体积更小，`4.2.1+`支持透明度（）；
* **使用`PNG`**：有透明度、渐变、阴影的情况下选择`PNG`。

#### 2.1.3 `WebP`转换方式

> 1. 在`AndroidStudio`中 右键-->`Convert to WebP`即可；
> 2. 下载[`WebP`转换工具](https://developers.google.com/speed/webp/docs/precompiled) 进行图片统一压缩。

#### 2.1.4 `PNG`压缩

采用压缩工具对`png`进行压缩(透明通道)：

> 1. 可以采用`ImageOptim`或者`Pngyu`对`png`进行压缩；
> 2. `AAPT`会使用内置的压缩算法来优化`res/drawable/`目录下的`PNG`图片，但也可能会导致本来已经优化过的图片体积变大，这里禁用`aapt`优化`PNG`图片。

```groovy
aaptOptions {
  cruncherEnabled = false
}
```

#### 2.1.5 `Jpg`图片压缩

采用压缩工具对`JPG`压缩：

> 1. 使用`packJPG`或者`guetzli`压缩`jpg`图片。

#### 2.1.6 其它图片优化

* 纯色图片代码实现

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
       android:shape="rectangle">
	<corners android:radius="5dp"/>
  <solid android:color="#FF0000"/>
</shape>
```

* 减少图片资源放置份数：放置`xhdpi`、`xxhdpi`如果只放一份会有什么问题？
* 能用代码实现的图片尽量采用代码实现：圆形图片、环形、渐变等都可以采用代码实现。

### 2.2 资源压缩

#### 2.2.1 开启资源压缩

开启代码混淆与去除无用资源：

```groovy
android {
  //...
  buildTypes {
    release {
      shrinkResources true
      minifyEnabled true
      proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
  }
}
```

#### 2.2.2 进一步资源混淆

微信开源了 [`AndResGuard`](https://github.com/shwenzhang/AndResGuard) 工具，对资源进一步混淆：

```groovy
buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath 'com.tencent.mm:AndResGuard-gradle-plugin:1.2.20'
  }
}
```

```groovy
andResGuard {
  // mappingFile = file("./resource_mapping.txt")
  // mappingFile用于增量更新，保持本次混淆与上次混淆结果一致
  mappingFile = null
  // uss7zip为true时，useSign必须为true
  use7zip = true
  // useSign为true时，需要配置signConfig
  useSign = true
  // 打开这个开关，会keep住所有资源的元素路径，只混淆资源的名字
  keepRoot = false
  
  // It will merge the duplicated resources, but don't rely on this feature too much.
  // it's always better to remove duplicated resource from repo
  mergeDuplicatedRes = true
  // whiteList添加在代码内部需要动态获取的资源id，不混淆这部分
  whiteList = [
    // your icon
    "R.drawable.icon",
    // for fabric
    "R.string.com.crashlytics.*",
    // for google-services
    "R.string.google_app_id",
    "R.string.gcm_defaultSenderId",
    "R.string.default_web_client_id",
    "R.string.ga_trackingId",
    "R.string.firebase_database_url",
    "R.string.google_api_key",
    "R.string.google_crash_reporting_api_key",
    "R.string.project_id",
  ]
  compressFilePattern = [
    "*.png",
    "*.jpg",
    "*.jpeg",
    "*.gif",
  ]
  sevenzip {
    artifact = 'com.tencent.mm:SevenZip:1.2.20'
    //path = "/usr/local/bin/7za"
  }

  /**
    * Optional: if finalApkBackupPath is null, AndResGuard will overwrite final apk
    * to the path which assemble[Task] write to
    **/
  // finalApkBackupPath = "${project.rootDir}/final.apk"

  /**
    * Optional: Specifies the name of the message digest algorithm to user when digesting the entries of JAR file
    * Only works in V1signing, default value is "SHA-1"
    **/
  // digestalg = "SHA-256"
}
```

### 2.3 其它优化

#### 2.3.1 冗余代码优化

* 为什么会有冗余代码？

> 1. `Ctrl+C`、`Ctrl+V`；
> 2. 对项目不了解。

* 冗余代码的定义：

> 1. 完全一致的代码或者只修改了空格和评论；
> 2. 结构上和句法上一致的代码，例如只是修改了变量名；
> 3. 插入和删除了部分代码；
> 4. 功能和逻辑上一致的代码，语义上的拷贝。

* 用什么工具检测？

> 1. 使用工具，如`PMD`；
> 2. `PMD`下载地址： [https://github.com/pmd/pmd](https://github.com/pmd/pmd) .

* 怎么检测？

​	下载后输入：

```bash
./run.sh cpdgui
pmd -d /usr/src -R rulesets/java/quickstart.xml -f text
```

#### 2.3.2 `Lint`大法

* 检测未使用资源并删除：

```bash
Android Studio --> Analyze --> Run inspection by Name --> unused resource
```

* 无用代码优化：

```bash
Analyze --> Run Inspection by Name --> unused declaration --> Module app --> ok
```

* 其它`lint`优化

#### 2.3.3 其它优化

* 压缩存储文件：

> 1. 导入`7zip`，压缩存储预制资源；
> 2. 使用时，解压到本地，比如`assets`中的资源。

* 语言资源优化：

​	在`build`配置中配置`resConfigs`指定需要的语言类型：

```groovy
defaultConfig {
  resConfigs "zh", "en"
}
```

* `Splits`根据不同的`ABI`以及不同的屏幕密度分别打包：

  https://developer.android.com/studio/build/configure-apk-splits.html

* 重复的`string`、`color`优化：

> 1. 过滤重复的`string`；
> 2. 定义唯一的`color`名。

* 减少`Enum`使用：

> 1. 减少一个`ENUM`文件可以减少 1K 左右的大小；
> 2. 采用常量定义。

* 优化引用的库：

> 1. 去除不再使用的库；
> 2. 优化过时的库；
> 3. 仅仅提取使用的代码；
> 4. 选用更小的外部库。

* 音频资源压缩：

> 1. 采用音频压缩工具，压缩音频，降低音频采样率，通道数等，在不明显影响效果的前提下压缩音频。

* `SO`动态下发：

> 1. `SO`可以采用动态下发的方式加载 --> 下载失败了怎么办？
> 2. 仅仅只需要加载对应`abi`下的`so`。

* 指定`abi`：

  `Gradle`中指定`abiFilter`：

  ```groovy
  ndk {
    abiFilters "armeabi-v7a", "x86"
  }
  ```

  