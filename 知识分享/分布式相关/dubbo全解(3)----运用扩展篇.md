

# 常见的dubbo扩展场景

## 调用相关

dubbo支持单向调用，异步调用，参数回调，事件通知。具体用法见----https://cn.dubbo.apache.org/zh-cn/blog/2018/08/14/dubbo-%e5%85%b3%e4%ba%8e%e5%90%8c%e6%ad%a5/%e5%bc%82%e6%ad%a5%e8%b0%83%e7%94%a8%e7%9a%84%e5%87%a0%e7%a7%8d%e6%96%b9%e5%bc%8f/



## 泛化调用

当消费者不想加载众多依赖时，可以使用泛化调用。可以参照dubbo里的demo项目org.apache.dubbo.demo.consumer.Application#runWithBootstrap

```java
// generic invoke
        GenericService genericService = (GenericService) demoService;
        Object genericInvokeResult = genericService.$invoke("sayHello", new String[] { String.class.getName() },
                new Object[] { "dubbo generic invoke" });
```

## 在RPC调用中传输自定义参数

思路：可以借助Filter链将参数设置在RpcContext里面。


## 使用url与route实现分支隔离

思路：（1）借助自定义Registry注册时附带自定义字段，借助router实现只筛选自定义字段一致的invoker






