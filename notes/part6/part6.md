## Spark 中的 --files 参数与 ConfigFactory 工厂方法

## 1.1 使用配置库

**配置文件解析利器-Config库**

 Typesafe的Config库，纯Java写成、零外部依赖、代码精简、功能灵活、API友好。支持Java properties、JSON、JSON超集格式HOCON以及环境变量。
 
```
ublic class Configure {
    private final Config config;
 
    public Configure(String confFileName) {
        config = ConfigFactory.load(confFileName);
    }
 
    public Configure() {
        config = ConfigFactory.load();
    }
 
    public String getString(String name) {
        return config.getString(name);
    }
}
```
ConfigFactory.load()会加载配置文件，默认加载classpath下的application.conf,application.json和application.properties文件。当然也可以调用ConfigFactory.load(confFileName)加载指定的配置文件。

配置内容即可以是层级关系，也可以用”.”号分隔写成一行:

```
host{
  ip = 127.0.0.1
  port = 2282
}

```
或者

```
host.ip = 127.0.0.1
host.port = 2282
```
即json格式和properties格式。貌似:

 - *.json只能是json格式
 - *.properties只能是properties格式
 - *.conf可以是两者混合

而且配置文件只能是以上三种后缀名

如果多个config 文件有冲突时，解决方案有:
- 1. a.withFallback(b) //a和b合并，如果有相同的key，以a为准 
- 2. a.withOnlyPath(String path) //只取a里的path下的配置
- 3. a.withoutPath(String path) //只取a里出path外的配置



