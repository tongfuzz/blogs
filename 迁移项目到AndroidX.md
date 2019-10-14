###迁移项目到Androidx

迁移到androidx是早晚的事，对于新特性的添加，谷歌会优先加入到androidx中的，趁着项目不是特别忙，将当前项目迁移到androidx主要有以下几个步骤

1，设置app中的build.gradle中compileSdkVersion和targetSdkVersion在28以上
2，设置project中的build.gradle中的gradle构建版本为3.2.0,如下

```java
com.android.tools.build:gradle:3.2.0
```
3，如果您有任何尚未迁移至 AndroidX 命名空间的 Maven 依赖项，那么当您在 gradle.properties 文件中将以下两个标记设置为 true 时，Android Studio 编译系统也会为您迁移这些依赖项：

```java
android.useAndroidX=true
android.enableJetifier=true
```
    

要迁移未使用任何第三方库但带有需要转换的依赖项的现有项目，可以将 android.useAndroidX 标记设置为 true，并将 android.enableJetifier 标记设置为 false

4，借助 Android Studio 3.2 及更高版本，您可以通过从菜单栏中依次选择 Refactor > Migrate to AndroidX

5，因为低版本的butterknife没有兼容androidx ，修改butterknife的版本到最新的10.1.0

6，重新构建项目，如果仍然找不到相应的类的话，可以通过Edit >  Find > Replace in Path整体查找和修改类和xml文件中的引用，被替换类和替换类对照表参考[类映射表](https://developer.android.com/jetpack/androidx/migrate)
