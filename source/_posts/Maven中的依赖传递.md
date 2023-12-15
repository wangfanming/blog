---
title: Maven中的依赖传递
date: '2018/4/28 14:53:31'
categories:
  - Web
tags:
  - maven

---

## 简介

Maven 是一款基于 Java 平台的项目管理和整合工具，它将项目的开发和管理过程抽象成一个项目对象模型（POM）。开发人员只需要做一些简单的配置，Maven 就可以自动完成项目的编译、测试、打包、发布以及部署等工作。

Maven 是使用 Java 语言编写的，因此它和 Java 一样具有跨平台性，这意味着无论是在 Windows ，还是在 Linux 或者 Mac OS 上，都可以使用相同的命令进行操作。

Maven 使用标准的目录结构和默认构建生命周期，因此开发者几乎不用花费多少时间就能够自动完成项目的基础构建工作。

Maven 能够帮助开发者完成以下任务：

- 构建项目
- 生成文档
- 创建报告
- 维护依赖
- 软件配置管理
- 发布
- 部署

总而言之，Maven 简化并标准化了项目构建过程。它将项目的编译，生成文档，创建报告，发布，部署等任务无缝衔接，构建成一套完整的生命周期。

## Maven 的目标

Maven 的主要目标是为为开发人员提供如下内容：

- 一个可重复使用，可维护且易于理解的项目综合模型
- 与此模型进行交互的工具和插件

<!--More-->

## Maven 的目标

Maven 的主要目标是为为开发人员提供如下内容：

- 一个可重复使用，可维护且易于理解的项目综合模型
- 与此模型进行交互的工具和插件

## 约定优于配置

约定优于配置（Convention Over Configuration）是 Maven 最核心的涉及理念之一 ，Maven对项目的目录结构、测试用例命名方式等内容都做了规定，凡是使用 Maven 管理的项目都必须遵守这些规则。

Maven 项目构建过程中，会自动创建默认项目结构，开发人员仅需要在相应目录结构下放置相应的文件即可。

例如，下表显示了项目源代码文件，资源文件和其他配置在 Maven 项目中的默认位置。 



| 文件         | 目录               |
| ------------ | ------------------ |
| Java 源代码  | src/main/java      |
| 资源文件     | src/main/resources |
| 测试源代码   | src/test/java      |
| 测试资源文件 | src/test/resources |
| 打包输出文件 | target             |
| 编译输出文件 | target/classes     |

## Maven 的特点

Maven 具有以下特点：

1. 设置简单。
2. 所有项目的用法一致。
3. 可以管理和自动进行更新依赖。
4. 庞大且不断增长的资源库。
5. 可扩展，使用 Java 或脚本语言可以轻松的编写插件。
6. 几乎无需额外配置，即可立即访问新功能。
7. 基于模型的构建：Maven 能够将任意数量的项目构建为预定义的输出类型，例如 JAR，WAR。
8. 项目信息采取集中式的元数据管理：使用与构建过程相同的元数据，Maven 能够生成一个网站（site）和一个包含完整文档的 PDF。
9. 发布管理和发行发布：Maven 可以与源代码控制系统（例如 Git、SVN）集成并管理项目的发布。
10. 向后兼容性：您可以轻松地将项目从旧版本的 Maven 移植到更高版本的 Maven 中。
11. 并行构建：它能够分析项目依赖关系，并行构建工作，使用此功能，可以将性能提高 20%-50％。
12. 更好的错误和完整性报告：Maven 使用了较为完善的错误报告机制，它提供了指向 Maven Wiki 页面的链接，您将在其中获得有关错误的完整描述。



## 依赖范围scope
>Maven因为执行一系列编译、测试、和部署等操作，在不同的操作下使用的classpath不同，**依赖范围就是控制依赖与三种classpath(编译classpath、测试classpath、运行classpath)的关系**。

一共有5种，compile(编译)、test(测试)、runtime(运行时)、provided、system不指定，则范围默认为compile。

 - compile：编译依赖范围，在编译、测试、运行时都需要。    
 - test：测试依赖范围，测试时需要。编译和运行不需要。
 - runtime：运行时以来范围，测试和运行时需要，编译不需要。
 - provided：已提供依赖范围，编译和测试时需要。运行时不需要。
 - system：系统依赖范围。本地依赖，不在maven中央仓库。

---
## 依赖的传递
A->B(compile)   a依赖b      
    >b是A的编译依赖范围，在A编译、测试、运行时都依赖b

B->C(compile)   B依赖C      
    >c是B的编译依赖范围，在B编译、测试、运行时都依赖c

当在A中配置

```xml
<dependency>  
        <groupId>com.B</groupId>  
        <artifactId>B</artifactId>  
        <version>1.0</version>  
</dependency>
```

则会自动导入c包。关系传递如下表：


由上表不难看出，项目A具体会不会导入B依赖的包c，取决于第一和第二依赖，但是依赖的范围不会超过第一直接依赖，即具体会不会引入c包，要看第一直接依赖的依赖范围。

## 依赖冲突的调节   

    A -> B -> C -> X(1.0)
    A -> D -> X(2.0)
由于只能引入一个版本的包，此时Maven按照最短路径选择导入X(2.0).

    A -> B -> X(1.0)
    A -> D -> X(2.0)
路径长度一致，则优先选择第一个，此时导入X(1.0).

## 排除依赖

    A -> B -> C(1.0)

此时在A项目中，不想使用C(1.0)，而使用C(2.0)

则需要使用exclusion排除B对C(1.0)的依赖。并在A中引入C(2.0).

 

```xml
pom.xml中配置
<!--排除B对C的依赖-->

<dependency>  
        <groupId>B</groupId>  
        <artifactId>B</artifactId>  
        <version>0.1</version>  
        <exclusions>
             <exclusion>
                <groupId>C</groupId>  
             <artifactId>C</artifactId><!--无需指定要排除项目的                                         版本号-->
             </exclusion>
        </exclusions>
</dependency> 

<!---在A中引入C(2.0)-->
<dependency>  
        <groupId>C</groupId>  
        <artifactId>C</artifactId>  
        <version>2.0</version>  
</dependency> 
```




## 依赖关系的查看

cmd进入工程根目录，执行  mvn dependency:tree

会列出依赖关系树及各依赖关系

**mvn dependency:analyze    分析依赖关系**   

​    

