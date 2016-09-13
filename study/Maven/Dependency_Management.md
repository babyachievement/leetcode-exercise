# Maven 依赖管理


## 1. Maven的依赖机制

Maven的依赖管理机制是Maven被人熟知的一大特点，也是Maven擅长的方面之一。依赖管理对于只有一个模块的项目来说没有什么难处，但对于拥有多个模块，甚至成百上千个模块的项目来说确实是个难点。



### 1.1 传递依赖

传递依赖是Maven2.0中的一个新特性。这个特性避免了开发者需要发现指定依赖所需要的库，Maven会自动将它们引入包含进来。该特性通过从远程仓库读取项目依赖项的项目文件实现。“[In general, all dependencies of those projects are used in your project, as are any that the project inherits from its parents, or from its dependencies, and so on](http://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html).”

Maven对于依赖的层级数并没有限制，只会在发现循环依赖时引发问题。使用传递依赖，包含的库的总体大小可能会快速增长。针对这个问题，有一些额外的特性可以用来限制哪些依赖才能被引入：

* 依赖调解（Dependency mediation）-这个特性能够决定在一个artifact有多个版本满足条件时使用哪个版本。当前，Maven2.0只支持使用距离最近版本，也就是说，会使用与项目依赖树中距离最近的版本。例如：A->B->C->D2.0, A->E->D1.0, 项目会选择D1.0引入。当然，可以在pom中明确指定所用版本。

* 依赖管理（Dependency Management)-该特性允许项目作者在artifact中碰见传递依赖或者在依赖中没有版本指定时，直接指定将使用的artifacts的版本。

* 依赖范围（Denpendency Scope）-该特性允许只引入与当前构建阶段相符的依赖。

* 排除依赖（Excluded dependencies）-如果项目X依赖项目Y，项目Y依赖项目Z，项目X的作者可以明确的在exclusion元素中排除对Z的依赖。

* 可选依赖（Optional dependencies）-如果项目Y依赖项目Z，项目Y的作者使用optional元素将Z标记为可选依赖。当项目X依赖Y时，X将只依赖Y不依赖Z。X项目的作者可以明确的将Z项目添加为依赖。

### 1.2 依赖范围

依赖范围用于限制依赖的传递性，同样对于多个构建任务的classpath也会有影响。

由6个范围可用：

* compile

范围的默认值。编译依赖在项目的所有classpath中都可用。而且，这些依赖还会传播到依赖项目中。

* provided

与compile很像，但是需要JDK或者容器在项目运行时提供依赖项。比如，当构建一个web程序时，需要设置Servlet API和相应的Java EE  API的依赖的范围为<code>provided</code>，因为web 容器会提供这些类。这个范围只在编译和测试的classpath可用，并且是不传递的。

* runtime

这个范围意味着对于编译来说不需要，但是对于运行是需要的。在运行和测试的classpath中，但不在编译的classpath中。

* test

此范围的依赖对于程序的正常运行并不是需要的，只对测试的编译和执行阶段可用。此范围依赖不传递。

* system

与provided范围相似，除了需要明确地提供包含依赖的JAR包。artifact总是可用的并且不再仓库中查找。

* import

此范围只对pom的<code>depenencyManagement</code>节可用。意思是指定的POM将被POM的<dependencyManagement>节中的依赖所替代。因为是替换的，带有import范围的依赖不会首到传递依赖的限制。

### 1.3 依赖管理

依赖管理节是集中管理依赖信息的机制。如果一些列项目继承一个通用的项目，那么将所有相关依赖的信息放在公用的POM中是可以的，并且在子POM中能更方便地引用。这个机制通过一些例子能够更好地说明。

项目A：
```xml
    <project>
      ...
      <dependencies>
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-a</artifactId>
          <version>1.0</version>
          <exclusions>
            <exclusion>
              <groupId>group-c</groupId>
              <artifactId>excluded-artifact</artifactId>
            </exclusion>
          </exclusions>
        </dependency>
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-b</artifactId>
          <version>1.0</version>
          <type>bar</type>
          <scope>runtime</scope>
        </dependency>
      </dependencies>
    </project>
```

项目B:

```xml
    <project>
      ...
      <dependencies>
        <dependency>
          <groupId>group-c</groupId>
          <artifactId>artifact-b</artifactId>
          <version>1.0</version>
          <type>war</type>
          <scope>runtime</scope>
        </dependency>
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-b</artifactId>
          <version>1.0</version>
          <type>bar</type>
          <scope>runtime</scope>
        </dependency>
      </dependencies>
    </project>
```
着两个项目拥有一个共同的依赖，并且每个都有一个不同的依赖。这些依赖项可以放在一个父POM文件中，如下：

```xml
    <project>
      ...
      <dependencyManagement>
        <dependencies>
          <dependency>
            <groupId>group-a</groupId>
            <artifactId>artifact-a</artifactId>
            <version>1.0</version>
     
            <exclusions>
              <exclusion>
                <groupId>group-c</groupId>
                <artifactId>excluded-artifact</artifactId>
              </exclusion>
            </exclusions>
     
          </dependency>
     
          <dependency>
            <groupId>group-c</groupId>
            <artifactId>artifact-b</artifactId>
            <version>1.0</version>
            <type>war</type>
            <scope>runtime</scope>
          </dependency>
     
          <dependency>
            <groupId>group-a</groupId>
            <artifactId>artifact-b</artifactId>
            <version>1.0</version>
            <type>bar</type>
            <scope>runtime</scope>
          </dependency>
        </dependencies>
      </dependencyManagement>
    </project>
```

两个子POM文件将变得更简单：

```xml
    <project>
      ...
      <dependencies>
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-a</artifactId>
        </dependency>
     
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-b</artifactId>
          <!-- This is not a jar dependency, so we must specify type. -->
          <type>bar</type>
        </dependency>
      </dependencies>
    </project>
```

```xml
    <project>
      ...
      <dependencies>
        <dependency>
          <groupId>group-c</groupId>
          <artifactId>artifact-b</artifactId>
          <!-- This is not a jar dependency, so we must specify type. -->
          <type>war</type>
        </dependency>
     
        <dependency>
          <groupId>group-a</groupId>
          <artifactId>artifact-b</artifactId>
          <!-- This is not a jar dependency, so we must specify type. -->
          <type>bar</type>
        </dependency>
      </dependencies>
    </project>
```

### 1.4 导入依赖

1.3节中介绍了怎么通过继承实现依赖的管理，但在大型的项目中，通过从单个父节点继承完成依赖管理不太可能，为了适应这个目标，项目可以从其他项目导入管理的依赖，这可以通过声明一个import范围的依赖实现。

项目A:
```xml
    <project>
     <modelVersion>4.0.0</modelVersion>
     <groupId>maven</groupId>
     <artifactId>A</artifactId>
     <packaging>pom</packaging>
     <name>A</name>
     <version>1.0</version>
     <dependencyManagement>
       <dependencies>
         <dependency>
           <groupId>test</groupId>
           <artifactId>a</artifactId>
           <version>1.2</version>
         </dependency>
         <dependency>
           <groupId>test</groupId>
           <artifactId>b</artifactId>
           <version>1.0</version>
           <scope>compile</scope>
         </dependency>
         <dependency>
           <groupId>test</groupId>
           <artifactId>c</artifactId>
           <version>1.0</version>
           <scope>compile</scope>
         </dependency>
         <dependency>
           <groupId>test</groupId>
           <artifactId>d</artifactId>
           <version>1.2</version>
         </dependency>
       </dependencies>
     </dependencyManagement>
    </project>
```

项目B：
```xml
    <project>
      <modelVersion>4.0.0</modelVersion>
      <groupId>maven</groupId>
      <artifactId>B</artifactId>
      <packaging>pom</packaging>
      <name>B</name>
      <version>1.0</version>
      <dependencyManagement>
        <dependencies>
          <dependency>
            <groupId>maven</groupId>
            <artifactId>A</artifactId>
            <version>1.0</version>
            <type>pom</type>
            <scope>import</scope>
          </dependency>
          <dependency>
            <groupId>test</groupId>
            <artifactId>d</artifactId>
            <version>1.0</version>
          </dependency>
        </dependencies>
      </dependencyManagement>
      <dependencies>
        <dependency>
          <groupId>test</groupId>
          <artifactId>a</artifactId>
          <version>1.0</version>
          <scope>runtime</scope>
        </dependency>
        <dependency>
          <groupId>test</groupId>
          <artifactId>c</artifactId>
          <scope>runtime</scope>
        </dependency>
      </dependencies>
    </project>
```

### 1.4 系统依赖

## dependencyManagement Import 
