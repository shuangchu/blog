---
title: "Maven 高级用法"
tag:
    - Maven
---
# 1.依赖范围

1.依赖的jar默认情况可以在任何地方使用，通过scope标签设定
起作用范围:
    1.主要程序范围有效（main文件夹范围）
    2.测试程序范围有效（test文件范围内）
    3.是否参与打包（package指令范围内）

|     scope     | 主代码 | 测试代码 | 打包 |
| :-----------: | :----: | :------: | :--: |
| complie(默认) |   Y   |    Y    |  Y  |
|     test     |        |    Y    |      |
|   provided   |   Y   |    Y    |      |
|    runtime    |        |          |  Y  |

```
	<!-- scope 使用在dependency下，表示以来的范围	-->>
	<dependencies>
		<dependency>
			<groupId></groupId>
        		<artifactId></artifactId>
        		<version></version>
        		<scope></scope>
		</dependency>
	</dependencies>
```

# 2.聚合

* 在一个模块中，只有pom.xml文件。将该模块指定为父工程，管理其他所有模块.
* 在父工程执行Maven命令，子工程会按照依赖关系，顺序执行命令。
* packaging标签定义该工程用于进行构建管理。
* modules标签中添加具体子工程的名称，用于管理各个子工程。

```
<packaging>pom<packaging>

<modules>
    <!--module里面写子工程的具体名称-->
    <module></module>
    <module></module>
           ...
</modules>
```

# 3.继承

* 在父工程中定义整个项目要用到的所有依赖。
* 在子工程中，若要使用父工程添加的依赖，需要使用parent标签指定父工程。
* 子工程使用父工程的依赖，不需要指定版本号。
* 插件也可以这么使用。

```
<!--父工程-->
<dependencyManagement>
    <dependencies>
        <dependency>
            ...
        </dependenct>
    </dependencies>
</dependencyManagement>

<!--子工程-->
<parent>
    <groupId></groupId>
    <artifactId></artifactId>
    <version></version>
    <relativePath></relativePath>
</parent>

```

# 4.属性

* 本质就是键值对，使用properties标签定义值，EL表达式取值。
* 只能在同一文件中取值。

```
<properties>
    <spring.version>5.1.7</spring.version>
</properties>

<dependency>
    ....
    <version>${spring.version}</version>
</dependency>
```

# 5.资源加载属性值

配置方法如下

```
//jdbc配置文件
...
jdbc.url = ${jdbc.url}
...
...
```

```
<properties>
    <jdbc.url>jdbc:mysql://localhost:3306/test<jdbc.url>
</proterties>

<build>
    ...
    <resources>
        <resource>
            <directory><!--模块的resource目录位置--></directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>

```

# 6.多环境配置

配置方法如下

```
<profiles>
    <profile>
    <!--生产环境-->
      <id>pro_env</id>
      ...
      <properties>
        <jdbc.url>...</jdbc.url>
      </properties>
      ...
    </profile>
    <profile>
    <!--开发环境-->
      <id>dep_env</id>
      ...
      <properties>
        <jdbc.url>...</jdbc.url>
      </properties>
      ...
    </profile>
  </profiles>

```
