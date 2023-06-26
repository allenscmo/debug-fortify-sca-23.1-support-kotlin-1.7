# Issue about Fortify SCA 23.1 supporting kotlin-gradle-plugin 1.7 

Content:
- Background
- How the issue happen
- Root Cause
- Workaround
- Final solution?

## Background

We find that Fortify SCA 23.1 works for Kotlin project that use kotlin-gradle-plugin 1.6, but not work for Kotlin project that use kotlin-gradle-plugin 1.7+.

The software environment:
- Fortify SCA version: 23.1.0.0136
- OpenJDK 11.0.19
- Ubuntu 18.04.6 LTS

## How the issue happen

We find a Kotlin hello-world project as example: [HelloKotlinAndroid](https://github.com/kousen/HelloKotlinAndroid.git). [This project use Kotlin and kotlin-gradle-plugin 1.7.20](https://github.com/kousen/HelloKotlinAndroid/blob/master/gradle/libs.versions.toml#L2)

The follow steps take us to the issue:
```
git clone https://github.com/kousen/HelloKotlinAndroid.git

cd HelloKotlinAndroid

fortifyupdate

sourceanalyzer -Xmx12000M -b kotlin-1.7-debug -clean

# Translate the kotlin project
# Add -debug-verbose option for debug log
sourceanalyzer -debug-verbose -Xmx12000M -b kotlin-1.7-debug ./gradlew assembleRelease
```

It throws the following exception when translating the project:
```
* Where:
Initialization script '/root/.fortify/sca23.1/build/kotlin-1.7-debug/init-script14145138810832154397.gradle' line: 126
 
* What went wrong:
Execution failed for task ':app:kaptGenerateStubsReleaseKotlin'.
> Could not find method getSource() for arguments [] on task ':app:kaptGenerateStubsReleaseKotlin' of type org.jetbrains.kotlin.gradle.internal.KaptGenerateStubsTask.
```

## Root Cause

We have checked the Gradle script ``Tools/init-script-tplt.gradle`` under Fortify SCA 23.1 installation folder. It uses the ``getSource()`` method of class ``org.jetbrains.kotlin.gradle.tasks.KotlinCompile``.

``` 
120                         if (DEBUG) System.out.println("---- classpath length= " + classpathLen);
121
122                         srcFiles = task.getSource().getFiles();
123                 } else {
124                         if (DEBUG) System.out.println(task.getName() + " NOT instance of either JavaCompile or KotlinCompile or CCompile or CppCompile: " + task.getClass());
```
The class ``org.jetbrains.kotlin.gradle.tasks.KotlinCompile`` is imported from kotlin-gradle-plugin.

**For kotlin-gradle-plugin 1.6, class KotlinCompile do have a getSource() method**:
https://github.com/JetBrains/kotlin/blob/v1.6.20/libraries/tools/kotlin-gradle-plugin/src/main/kotlin/org/jetbrains/kotlin/gradle/tasks/Tasks.kt#L83
 
**But for kotlin-gradle-plugin 1.7, the getSource() method has been removed**:
https://github.com/JetBrains/kotlin/blob/v1.7.20/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/Tasks.kt#L230

**The substitute is the sources variable**: https://github.com/JetBrains/kotlin/blob/v1.7.20/libraries/tools/kotlin-gradle-plugin/src/common/kotlin/org/jetbrains/kotlin/gradle/tasks/Tasks.kt#L92-L95


## Workaround

Update ``Tools/init-script-tplt.gradle``, replace ``getSource()`` with ``sources``.
Because for kotlin-gradle-plugin 1.7, the ``getSource()`` method has been removed, and replaced by ``sources`` variable.

Before update:
```
120                         if (DEBUG) System.out.println("---- classpath length= " + classpathLen);
121
122                         srcFiles = task.getSource().getFiles();
123                 } else {
124                         if (DEBUG) System.out.println(task.getName() + " NOT instance of either JavaCompile or KotlinCompile or CCompile or CppCompile: " + task.getClass());
```

After update:
```
120                         if (DEBUG) System.out.println("---- classpath length= " + classpathLen);
121
122                         srcFiles = task.sources.getFiles();
123                 } else {
124                         if (DEBUG) System.out.println(task.getName() + " NOT instance of either JavaCompile or KotlinCompile or CCompile or CppCompile: " + task.getClass());
```

After this update, Fortify SCA works as expectation.


## Final solution?

- First, Fortify SCA 23.1 claims that it supports Kotlin 1.7: https://www.microfocus.com/documentation/fortify-core-documents/2310/Fortify_Sys_Reqs_23.1.0/index.htm#SCA/SCA_SupportedLangs.htm?TocPath=Fortify%2520Static%2520Code%2520Analyzer%2520Requirements%257CLanguages%257C_____0 . kotlin-gradle-plugin 1.7 is the build suite of Kotlin 1.7, so Fortofy SCA 23.1 must also support kotlin-gradle-plugin 1.7.
- Second, Fortify SCA 23.1 ``Tools/init-script-tplt.gradle`` script uses internal class and method of kotlin-gradle-plugin to translate the Kotlin code.
- Third, Fortify SCA 23.1 uses an already removed method of kotlin-gradle-plugin 1.7: the getSource() mothed of org.jetbrains.kotlin.gradle.tasks.KotlinCompile class.

Base on the above analysis, it's a bug of Fortify SCA 23.1, and it must be fixed. 
