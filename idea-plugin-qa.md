---
layout: page
title: IDEA-PLUGIN-WITH-GRADLE-QA
ftitle: IDEA Plugin Development
---

## 问题清单

***

*Q1*：<font color="red"><em>java版本和org.jetbrains.intellij插件不匹配</em></font>

>当org.jetbrains.intellij为`1.3.0`，java版本为`1.8`时，报错需要java 11

a. 修改build.gradle ( 降版本 `1.3.0`->`0.4.21` )

{% highlight js %}

plugins {
    id 'org.jetbrains.intellij' version '0.4.21'
    id 'java'
}

{% endhighlight %}

b. 修改java版本为11

**Settings** `|` **Preferences** `|` **Build, Execution, Deployment** `|` **Build Tools** `|` **Gradle** `|` **JVM Version**

注意：gradle，org.jetbrains.intellij和java三者之间版本要匹配

org.jetbrains.intellij 1.3.0 适用于 Gradle 6.6+

{% highlight js %}

//Gradle升级

./gradlew wrapper --gradle-version=VERSION

{% endhighlight %}

***

*Q2*：<font color="red"><em>org.jetbrains.intellij中对java的支持分包</em></font>

>当导入psi包下PsiJavaClass时，引用缺失

[Solution](https://blog.jetbrains.com/platform/2019/06/java-functionality-extracted-as-a-plugin/)`————`

<cite>
In the latest EAP build of IntelliJ IDEA 2019.2 we’ve extracted Java functionality into a separate plugin. This separates the Java implementation from the “platform” part of IntelliJ IDEA and introduces some flexibility for the future. For example, we could include Java functionality as a plugin to other products, and other plugins such as Gradle can now take optional dependencies on Java, and still work if Java isn’t available.
</cite>

<cite>
The new plugin is not visible in Settings `|` Plugins and cannot be switched off, so there should be no impact on end users. However, if you’re writing a plugin for IntelliJ IDEA you may need to make some small changes to continue working correctly.
</cite>

<cite>
Firstly, if your plugin depends on the Java part of the IntelliJ API, you will need to declare this dependency in your plugin.xml file, by adding the following line:
</cite>

{% highlight js %}

<depends> com.intellij.modules.java </depends>

{% endhighlight %}

<cite>
You might have this line in your plugin.xml already, as this is the recommended way of declaring a dependency on the Java functionality, even for earlier versions of IntelliJ IDEA. However, it is now required, to ensure that your plugin can access classes from the Java plugin at runtime. See the Plugin Compatibility with IntelliJ Platform Products page for more details on dependencies.
</cite>

<cite>
Secondly, if you’re developing a plugin using gradle-intellij-plugin (please make sure you’re using the latest version), you need to tell Gradle about the Java plugin. Add the following to build.gradle in order to include classes from the Java plugin into the compilation classpath, and tell the IDE to load the plugin at runtime:
</cite>


{% highlight js %}

intellij {
  plugins = ['com.intellij.java']
  // ...
}

{% endhighlight %}

<cite>
Please let us know if you have any issues.
</cite>

***

*Q3*：<font color="red"><em> 在插件中如何加载源项目(source project)的类</em></font>

>当调用源项目中方法，由于这部分依赖不在类路径下，会导致Class Not Def错误

a. 扩展ClassLoader

{% highlight js %}

private static class ExtendedClassLoader extends URLClassLoader{
        public ExtendedClassLoader(URL[] urls, ClassLoader parent) {
            super(urls, parent);
        }

        public void addURL(URL url) {
            super.addURL(url);
        }
    }

{% endhighlight %}

首先获取插件项目类加载器(PluginClassLoader)，然后作为父加载器传入自定义加载器
如何获取？只需要获取当前项目任意类加载器即可

{% highlight js %}

ClassLoader threadClassLoader =Thread.currentThread().getContextClassLoader();
ExtendedClassLoader extendedClassLoader=new ExtendedClassLoader(new URL[0], JacoconutApi.class.getClassLoader());

{% endhighlight %}

然后加入源项目类路径(这里同时加入了测试类路径和源代码类路径)

{% highlight js %}

extendedClassLoader.addURL(new URL(("file:"+ProjectParams.PROJECT_ROOT.get()+"/target/classes/").replace("/","\\\\")));

extendedClassLoader.addURL(new URL(("file:"+ProjectParams.PROJECT_ROOT.get()+"/target/test-classes/").replace("/","\\\\")));

{% endhighlight %}

最后替换classloader，记得换回来

{% highlight js %}

Thread.currentThread().setContextClassLoader(extendedClassLoader);

//---------do what you want

Thread.currentThread().setContextClassLoader(threadClassLoader);

{% endhighlight %}

***

*Q4*：<font color="red"><em>jdk9以后版本AppClassLoader不再是URLClassLoader的子类</em></font>

>试图解决*Q3*时采用如下方法

{% highlight js %}

URLClassLoader urlClassLoader = (URLClassLoader) ClassLoader.getSystemClassLoader();

{% endhighlight %}

a. 同2.1.3

b.
{% highlight js %}

object ClassloaderHelper {
  def getURLs(classloader: ClassLoader) = {
    // jdk9+ need to use reflection
    val clazz = classloader.getClass

    val field = clazz.getDeclaredField("ucp")
    field.setAccessible(true)
    val value = field.get(classloader)

    value.asInstanceOf[URLClassPath].getURLs
  }
}

val classpath =
  (
    // jdk8
    // ClassLoader.getSystemClassLoader.asInstanceOf`[`URLClassLoader`]`.getURLs ++
    // getClass.getClassLoader.asInstanceOf`[`URLClassLoader`]`.getURLs

    // jdk9+
    ClassloaderHelper.getURLs(ClassLoader.getSystemClassLoader) ++
    ClassloaderHelper.getURLs(getClass.getClassLoader)
  )

  {% endhighlight %}
  [参见stackoverflow](https://stackoverflow.com/questions/46694600/java-9-compatability-issue-with-classloader-getsystemclassloader)
