# 编译Flink代码

Date: 2022/10/06 Author: KenG

## Environment

1.  Java11/8

1.  Maven@3.2.5  or Maven@3.8.5 (注意使用该较新版本的Maven编译时，部分模块可能报错，可以参考Troubleshooting2的步骤进行解决)

1.  macOS 10.15

## Step

```bash
git clone https://github.com/apache/flink
mvn clean install -DskipTests -Dfast

# faster by skipping web ui and parallel building
mvn clean install -DskipTests -Dfast -Pskip-webui-build -T 1C
```

## Troubleshooting

1.  在编译flink-runtime-web常常会报`npm ci --cache-max=0 --no-save`失败的错误

    解决方案：

    -   修改flink/flink-runtime-web/pom.xml中的`<npm.proxy>--registry http://172.17.0.1:8888/repository/npm/</npm.proxy>` 为`<npm.proxy>--registry https://registry.npm.taobao.org</npm.proxy>` 使用淘宝npm源；

    -   尝试运行`mvn clean install -DskipTests -Dfast -rf :flink-runtime-web` 继续编译；如果报相同错误， cd flink/flink-runtime-web 然后npm install --cache-max=0 --no-save --registry https://registry.npm.taobao.org --force观察是否成功

    -   如果上述手动npm install的操作仍然失败，尝试更新npm版本(Mac上可以`brew upgrade npm`)后再次尝试；npm install成功后可再次返回flink项目根目录，继续编译。

    -   如果不需要WebUI功能，mvn命令添加`-Pskip-webui-build`.

1.  使用较新版本的maven编译flink-connector-hive时报错: `[ERROR] Failed to execute goal on project flink-connector-hive_2.12: Could not resolve dependencies for project org.apache.flink:flink-connector-hive_2.12:jar:1.16-SNAPSHOT: Failed to collect dependencies at org.apache.hive:hive-exec:jar:2.3.9 -> org.pentaho:pentaho-aggdesigner-algorithm:jar:5.1.5-jhyde: Failed to read artifact descriptor for org.pentaho:pentaho-aggdesigner-algorithm:jar:5.1.5-jhyde: Could not transfer artifact org.pentaho:pentaho-aggdesigner-algorithm:pom:5.1.5-jhyde from/to maven-default-http-blocker (http://0.0.0.0/): Blocked mirror for repositories: [conjars (http://conjars.org/repo, default, releases+snapshots), apache.snapshots (http://repository.apache.org/snapshots, default, snapshots)] -> [Help 1]`

    解决方案：

    -   方案一：参考Ref3修改该module的pom.xml添加conjars

    -   方案二：为方便之后提交代码，建议使用较低的Maven版本(e.g. 3.2.5)进行编译。Flink相关开发人员封装了一个 mvnw方便进行该较低的Maven版本进行操作，可以直接`./mvnw clean install -DskipTests -Dfast` 继续编译。或者通过[mvnvm](https://mvnvm.org/)来维护多个版本的Maven环境。

## Reference

1.  [Building Flink from Source | Apache Flink](https://nightlies.apache.org/flink/flink-docs-master/docs/flinkdev/building/)

1.  [Flink1.13.2源码编译 - 知乎](https://zhuanlan.zhihu.com/p/411559435)

1.  [[FLINK-27894] Build flink-connector-hive failed using Maven@3.8.5 - ASF JIRA](https://issues.apache.org/jira/browse/FLINK-27894)



Reference:

