```
@Transactional
```

在srpingMVC中是要使用 @Transactional注解即可开启事务，其中的道理很简单。

不管是hibernate还是Mybatisy又或者是其他的数据库使用，最终都是基于jdbc去实现的

因此spring要做的只是在执行sql的时候给他一个datasource connection连接

通过Aop或者启动时扫描 @Transactional注解

在执行该注解下的方法时，则绑定一个连接到本地线程，该方法的sql操作则始终由该连接完成

ThredLoal的设置的value存储在本地线程中，由该线程独享