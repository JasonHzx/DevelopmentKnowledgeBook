# 使用Gradle生成和分析依赖树

```java
compile - Compile dependencies for 'main' sources (deprecated: use 'implementation' instead).
+--- com.android.support:multidex:1.0.1
+--- com.jakewharton:butterknife:7.0.1
+--- io.reactivex:rxandroid:1.1.0
|    \--- io.reactivex:rxjava:1.1.0 -> 1.2.3
+--- io.reactivex:rxjava:1.1.0 -> 1.2.3
+--- com.squareup.retrofit2:adapter-rxjava:2.0.0
|    +--- com.squareup.retrofit2:retrofit:2.0.0 -> 2.0.2
|    |    \--- com.squareup.okhttp3:okhttp:3.2.0 -> 3.4.1
|    |         \--- com.squareup.okio:okio:1.9.0
|    \--- io.reactivex:rxjava:1.1.1 -> 1.2.3
+--- com.squareup.retrofit2:retrofit:2.0.0 -> 2.0.2 (*)
+--- com.squareup.retrofit2:converter-gson:2.0.0 -> 2.0.2
|    +--- com.squareup.retrofit2:retrofit:2.0.2 (*)
|    \--- com.google.code.gson:gson:2.6.1
+--- com.squareup.okhttp3:logging-interceptor:3.4.1
|    \--- com.squareup.okhttp3:okhttp:3.4.1 (*)
+--- com.bigkoo:convenientbanner:2.0.5
|    \--- com.android.support:appcompat-v7:21.0.3 -> 26.1.0
|         +--- com.android.support:support-annotations:26.1.0
|         +--- com.android.support:support-v4:26.1.0
|         |    +--- com.android.support:support-compat:26.1.0
|         |    |    +--- com.android.support:support-annotations:26.1.0
|         |    |    \--- android.arch.lifecycle:runtime:1.0.0
|         |    |         +--- android.arch.lifecycle:common:1.0.0
|         |    |         \--- android.arch.core:common:1.0.0
|         |    +--- com.android.support:support-media-compat:26.1.0
|         |    |    +--- com.android.support:support-annotations:26.1.0
|         |    |    \--- com.android.support:support-compat:26.1.0 (*)
|         |    +--- com.android.support:support-core-utils:26.1.0
|         |    |    +--- com.android.support:support-annotations:26.1.0
|         |    |    \--- com.android.support:support-compat:26.1.0 (*)
|         |    +--- com.android.support:support-core-ui:26.1.0
|         |    |    +--- com.android.support:support-annotations:26.1.0

```

## 生成项目依赖树
* 方式一：
直接Terminal输入命令：

```java
 ./gradlew :app:dependencies 
```
 
* 方式二：
Android Studio右侧图形化界面的gradle菜单内点击：

```java
app -> Tasks -> android -> androidDependencies
```

* 方式三：

```java
Android Studio Plugin: GradleView
```
View ->Tool Window ->Gradle View，开启GradleView则自动分析给出报告。相比前面两种的优点是一次安装就能重复使用，不用每次都输入一长串命令；缺点是不支持搜索；

### 存在的问题：

虽然上面的命令能达到目的，但是它会将gradle执行的各个步骤下的依赖树全部打印出来，包括 debugApk、debugCompile、releaseApk、releaseCompile、compile 等全部打印出来，不仅耗费时间长，而且结果冗余，不利于查找。
但实际上只需要 compile 时期的依赖树就行了，可以在命令后配置一个参数.
参考 gradle dependencies -configuration <configuration>
例 gradle dependencies -configuration compile => 查看编译时依赖
例 gradle dependencies -configuration runtime => 查看运行时依赖

所以改进为：

```
./gradlew :app:dependencies --configuration compile 
```

因为生成的结果很长，在Terminal内可能会显示不全。所以需要将结果输出到文件内查看。

继续改进为：

```
./gradlew :app:dependencies --configuration compile > dependenciesTree.txt
```

备注：目前不支持生成implementation方式的依赖树，现在想到办法是分析前改为complie，分析后改回去。。。


## 分析项目依赖树

符号含义说明

```
-> 用于指向版本冲突中获胜的依赖项。
```
由于默认会优先版本高的依赖,这个时候你想使用版本低的依赖的话需要排除掉高的依赖。


```
(*) 表示这个依赖被忽略了
```
这是因为其他顶级依赖中也依赖了相同的版本，Gradle会自动分析下载最合适的依赖。


##FAQ
* 	版本冲突时十分常见的，比如下面的例子

```java
// 库 a 传递性依赖库 b-1.2，与添加的b-1.1冲突
dependencies {
    compile 'a:a:1.0'
    compile 'b:b:1.1'
}
```
Gradle解决冲突有以下几种方式

```
(1) 最近版本策略（默认）：上例将忽略b-1.1，而下载b-1.2 
(2) 冲突失败策略：发生冲突时，编译失败（有些新版本库并不兼容之前的，因此这个库可以让开发者主动作出选择）
(3) 强制指定版本策略：发生冲突时，使用开发者指定的版本
```

设置冲突失败策略

```
configurations.all {
    resolutionStrategy 
      {
         failOnVersionConflict() 
      }
}
```


* 什么是传递性依赖：

```
简单说就是依赖的依赖
```

局部性的关闭传递依赖

```
 compile('com.android.support:support-v4:23.1.1'){
 // 为本依赖关闭依赖传递特性
        transitive = false
 }
```

全局性的关闭传递依赖

```
configurations.all {
   transitive = false
}
```
 
* 排除依赖

有些时候你可能需要排除一些传递性依赖中的某个模块，此时便不能靠单纯的关闭依赖传递特性来解决了。这时exclude就该登场了，如果说@jar彻底的解决了传递问题，那么exclude则是部分解决了传递问题.
可以通过configuration配置或者在依赖声明时添加exclude的方式来排除指定的引用。
exclude可以接收group和module两个参数，这两个参数可以单独使用也可以搭配使用,举个例子：

局部性的排除依赖

```java
compile('com.jason98k:DragSortList:1.0.2'){ 
//com.android.support:appcompat-v7:23.1.1 
// 依据构建名称排除 
exclude module: 'appcompat-v7' 
// 依据组织名称排除 
exclude group: 'com.android.support' 
// 依据组织名称+构件名称排除 
exclude group: 'com.android.support', module: 'appcompat-v7' 
}  
```

全局性的排除依赖

```java
configurations {
   all*.exclude group: 'com.android.support', module: 'appcompat-v7'
}
```


* 强制指定版本策略

有时候你可能仅仅是需要强制使用某个统一的依赖版本，而不是排除他们，那么此时force就该登场了。指定`force = true`属性可以冲突时优先使用该版本进行解决。

```java
compile('com.jason98k:DragSortList:1.0.2'){
// 冲突时优先使用该版本 
        force = true
 }
```

全局配置强制使用某个版本的依赖来解决依赖冲突中出现的依赖

```java
configurations.all {
   resolutionStrategy {
       force 'com.jason98k:DragSortList:1.0.2'
   }
}
```

* 谨慎使用动态依赖

在一些情形中，你可能想使用最新的依赖包在构建你的app或者library的时候。实现他的最好方式是使用动态版本。几种不同的动态控制版本方式：

```java
dependencies {
       //我们想得到最新的minor版本，并且其最小的版本号是23.0.
       compile 'com.android.support:appcompat-v7:23.0.+' 
       //得到最新的生产版本
       compile 'com.android.support:recyclerview-v7:+'
}
```

你应该谨慎的去使用动态版本，如果当你允许gradle去依赖最新版本，可能依赖的版本并不是稳定版，以及没有经过严谨测试包含bug的版本，这将导致你的主项目存在未知的风险。


* 使用依赖配置implementation不向上传递依赖

依赖指令说明：

```
api：等同compile
implementation：依赖不会传递
compileOnly：gradle 添加依赖到编译路径，编译时使用。（不会打包到APK）
runtimeOnly：gradle 添加依赖只打包到 APK，运行时使用。（不会添加到编译路径）
```

当你切换到新的 Android gradle plugin 3.x.x，你应用使用 implementation 替换所有的 compile 依赖配置。然后尝试编译和测试你的应用。如果没问题那样最好，如果有问题说明你的依赖或使用的代码现在是私有的或不可访问。

如果你是一个lib库的维护者，对于所有需要公开的 API 你应该使用 api 依赖配置，测试依赖或不让最终用户使用的依赖使用 implementation 依赖配置。

使用implementation会使编译速度有所增快。

