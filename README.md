经常在github上看到通过类似的语句引进别人的库：
```groovy
repositories { 
    jcenter()
}
dependencies {
    compile 'com.flipboard:bottomsheet-core:1.5.0' 
    compile 'com.flipboard:bottomsheet-commons:1.5.0' // optional
}
```
不得不说，使用起来真方便。那如果自己有好的库，如何分享给其他人，或者放在Bintray，方便自己使用呢？

###发布第一个Android库(想想还是有点小激动)

先到[https://bintray.com](https://bintray.com/)注册,注册之后获取API-Key:


![api-key.png](http://upload-images.jianshu.io/upload_images/759172-322f15a9692bc3ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

必须要特别感谢这位作者:Ashraff Hathibelagal

[http://code.tutsplus.com/zh-hans/tutorials/creating-and-publishing-an-android-library--cms-24582](http://code.tutsplus.com/zh-hans/tutorials/creating-and-publishing-an-android-library--cms-24582)
基本上都是按照他的教程做的。
也参考了XRecyclerView的gradle,也表示感谢.

文章虽好,但也有坑....
于是自己又整理了一下

主要分两步：

### 1，像往常一样写一个library

### 2，发布到Bintray

这里需要修改两个gradle。

#### Project build.gradle

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.0.0'

        /*
        第 1 步：添加必要的插件
        为了在 Android Studio 里与 Bintray 交互，你应该把 Bintray 插件引入到项目的 build.gradle 文件的 dependencies 里。
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.2'
        因为你要把库上传到 Maven 仓库，你还应该像下面这样添加 Maven 插件。
        classpath "com.github.dcendents:android-maven-gradle-plugin:1.3"
         */
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.2'
        classpath "com.github.dcendents:android-maven-gradle-plugin:1.3"
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

```

### Module build.gradle

```groovy
apply plugin: 'com.android.library'
/*
第 2 步：应用插件
打开您的库模块的 build.gradle 文件并添加以下代码，以应用我们在上一步中添加的插件。
 */
apply plugin: 'com.jfrog.bintray'
apply plugin: 'com.github.dcendents.android-maven'
/*
第 3 步: 指定 POM 详细信息
在上传库时，Bintray 插件会寻找 POM 文件。 即使 Maven 插件为你生成了它，你也应该自己指定
 groupId 标签和 version 标签的值。 要这样做，请使用 gradle文件中的group 和version 的变量。
 */
version = "1.0.0"
group = "com.example"


def siteUrl = 'https://github.com/heinika/MyHelloWorldLibrary'
def gitUrl = 'https://github.com/heinika/MyHelloWorldLibrary.git'

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.3"

    defaultConfig {
        minSdkVersion 15
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.3.0'
}

/*第 4 步: 生成源 JAR
为了遵守 Maven 标准，你的库也应该有一个包含了库的源文件的 JAR 文件。
 为了生成 JAR 文件，需要创建一个新的 Jar任务，
 generateSourcesJar，并且使用 from 功能指定的源文件的位置。*/
task generateSourcesJar(type: Jar) {
    from android.sourceSets.main.java.srcDirs
    classifier 'sources'
}

/*
第 5 步: 生成 Javadoc JAR
我们同样推荐，在你的库里有一个包含 Javadocs 的 JAR 文件。
 因为目前你还没有任何 Javadocs，需要创建一个新的 Javadoc 任务，generateJavadocs，来生成它们。
  使用 source 变量来指定源文件的位置。 你还应该更新 classpath 变量，以便该任务可以找到属于 Android SDK 的类。
 你可以通过把 android.getBootClasspath 方法的返回值添加给他，来这么做。
 */
task generateJavadocs(type: Javadoc) {
    source = android.sourceSets.main.java.srcDirs
    classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
}

/*下一步，要从 Javadocs 生成 JAR，需要创建 Jar 任务，generateJavadocsJar
，并把 generateJavadocs 的 destinationDir 属性传递给它的 from 功能。
为了确保在 generateJavadocsJar 任务只在 generateJavadocs 任务完成后才开始，
需要添加下面的代码片段，它使用了 dependsOn 方法来决定任务的顺序：
您的新任务应如下所示:*/
task generateJavadocsJar(type: Jar, dependsOn: generateJavadocs) {
    from generateJavadocs.destinationDir
    classifier 'javadoc'
}

/*
 *第 6 步: 引入生成的 JAR 文件
 *为了把源和 Javadoc JAR 文件导入到 artifacts 的列表里，你应该把他们的任务的名字添加到 configuration 里，
 * 称为 archives，artifacts 列表将被上传到 Maven 仓库。 使用下面的代码片段来完成:
 */
artifacts {
    archives generateJavadocsJar
    archives generateSourcesJar
}

/*第 7 步: 运行任务
现在是运行我们在前几步里创建的任务的时候了。 打开 Gradle Projects 窗口，搜索名为 install 的任务。*/

/*
第 8 步: 配置 Bintray 插件
要配置插件，你应该使用 Gradle 文件中的 bintray 闭包。 首先，使用与你的 Bintray 用户名和 API 密钥对应的 user 和 key 变量进
行身份验证。在 Bintray，你的库会被放置在 Bintray package 里。 你应该使用 pkg 闭包里命名直观的 repo、 name、licenses 和
vcsUrl 参数，提供详细的相关信息， 如果这个包不存在，会为你自动创建。当你将文件上传到 Bintray 时，他们会与 Bintray 包里的一
个版本相关联。 因此，pkg 必须包含一个 version 闭包，闭包的 name 属性要设为独一无二的名称。 另外，你还可以使用 desc，
released 和 vcsTag参数来提供描述、 发布日期和 Git 标签。
最后，为了指定应该上传的文件，要把 configuration 参数的值设为 archives。
bintray {
    user = 'test-user'
    key = '01234567890abcdef01234567890abcdef'
    pkg {
        repo = 'maven'
        name = 'com.github.hathibelagal.mylittlelibrary'

        version {
            name = '1.0.1-tuts'
            desc = 'My test upload'
            released  = new Date()
            vcsTag = '1.0.1'
        }

        licenses = ['Apache-2.0']
        vcsUrl = 'https://github.com/hathibelagal/LibraryTutorial.git'
        websiteUrl = 'https://github.com/hathibelagal/LibraryTutorial'
    }
    configurations = ['archives']
}
 */
Properties properties = new Properties()
properties.load(project.rootProject.file('local.properties').newDataInputStream())
bintray {
    user = properties.getProperty("BINTRAY_USER")
    key = properties.getProperty("BINTRAY_KEY")
    configurations = ['archives']
    pkg {
        repo = "maven"
        name = "MyHelloWorldLibrary"    //发布到JCenter上的项目名字

        version {
            name = '1.0.1-tuts'
            desc = 'My test upload'
            vcsTag = '1.0.0'
        }

        websiteUrl = siteUrl
        vcsUrl = gitUrl
        licenses = ["Apache-2.0"]
        publish = true
    }
}
/*
第 9 步: 使用 Bintray 插件上传文件
再次打开 Gradle Projects 窗口，搜索 bintrayUpload 任务。 双击它，启动上传文件。*/
/*
一旦任务完成，你就能打开浏览器来访问你的 Bintray 包的详细信息页面。 你会看到一个通知，
说你有四个未发布的文件。 如果要发布这些文件，单击 Publish 链接。 */
/*
* 使用 Bintray 里的库
你的库现在已经可以作为 Bintray 包使用了。 只要你分享了你的 Maven 仓库的 URL，加上 group ID、artifact ID 和 version number，
任何开发人员都可以访问你的库。 例如，为了使用我们刚才创建的库，开发人员必须引入下面这个代码片段:
repositories {
    maven {
        url 'https://dl.bintray.com/eruzza/maven'
    }
}
dependencies {
    compile 'com.github.hathibelagal.librarytutorial:mylittlelibrary:1.0.1@aar'
}
注意，在把库添加为 compile 依赖之前，
开发人员必须显式地在 repositories 列表里，引入你的仓库。*/

/*
 * Bintray的用户手册:https://bintray.com/docs/usermanual/
 */
```
![上传成功](http://upload-images.jianshu.io/upload_images/759172-49dda98e33d12ae2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 将库添加到 JCenter

默认情况下，Android Studio 会搜索一个名为 **JCenter** 的仓库里面的库。 如果你把自己的库引入到了 JCenter 存储库，开发人员就不必向他的 `repositories` 里添加任何东西了。

要将你的库添加到 JCenter，需要打开浏览器并访问你的 Bintray 的包的详细信息页面。 单击 **Add to JCenter** 按钮

接着你进入一个页面，让你填写一些信息。 你可以用 **Comments** 区域来提及任何关于这个库的细节。
单击 Send 按钮，启动 Bintray 的审查过程。 在一两天以内，Bintray 的工作人员会把你的库链接到 JCenter 仓库，这样你就将能在你的包的详细信息页面上，看到指向 JCenter 的链接了。
任何开发人员现在都可以使用你的库，而无需更改 `repositories` 列表。

![通过后点击JCenter如图](http://upload-images.jianshu.io/upload_images/759172-8daa4ea703561a51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 使用
以后用到这个库的时候，只需要下面一句就可以了。是不是很爽，哈哈哈哈。
```groovy
compile 'com.example:myfristlibrary:1.0.0'
```