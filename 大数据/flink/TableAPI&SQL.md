# Table API & SQL

## 动态表 & 连续查询(Continuous Query)

*动态表* 是 Flink 的支持流数据的 Table API 和 SQL 的核心概念。与表示批处理数据的静态表不同，动态表是随时间变化的。

查询动态表将生成一个 *连续查询* 。一个连续查询永远不会终止，结果会生成一个动态表。查询不断更新其(动态)结果表，以反映其(动态)输入表上的更改。本质上，动态表上的连续查询非常类似于定义物化视图的查询。

下图显示了流、动态表和连续查询之间的关系:

![image-20220301111820591](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220301111820591.png)

1. 将流转换为动态表。
2. 在动态表上计算一个连续查询，生成一个新的动态表。
3. 生成的动态表被转换回流。

**注意：** 动态表首先是一个逻辑概念。在查询执行期间不一定(完全)物化动态表。

## Source表

```sql
CREATE TABLE employee_information (
    emp_id INT,
    name VARCHAR,
    dept_id INT
) WITH ( 
    'connector' = 'filesystem',
    'path' = '/path/to/something.csv',
    'format' = 'csv'
);
```

## Sink表

```sql
SELECT 
   dept_id,
   COUNT(*) as emp_count 
FROM employee_information 
GROUP BY dept_id;
```



查看jar包内是否有类

```
jar -vtf /path/of.jar | grep ''
```



## 模板

To build an application JAR that contains all dependencies required for declared connectors and libraries, you can use the following shade plugin definition:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.1.1</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <artifactSet>
                            <excludes>
                                <exclude>com.google.code.findbugs:jsr305</exclude>
                                <exclude>org.slf4j:*</exclude>
                                <exclude>log4j:*</exclude>
                            </excludes>
                        </artifactSet>
                        <filters>
                            <filter>
                                <!-- Do not copy the signatures in the META-INF folder.
                                Otherwise, this might cause SecurityExceptions when using the JAR. -->
                                <artifact>*:*</artifact>
                                <excludes>
                                    <exclude>META-INF/*.SF</exclude>
                                    <exclude>META-INF/*.DSA</exclude>
                                    <exclude>META-INF/*.RSA</exclude>
                                </excludes>
                            </filter>
                        </filters>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>my.programs.main.clazz</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

