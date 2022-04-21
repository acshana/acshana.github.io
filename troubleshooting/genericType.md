# 泛型

## JSON

## spring

### 1. 携带泛型的Event无法被EventListener注解订阅

#### 1.1 结论

 由于类型擦除的存在，会导致**Event无法按照真正的内部对象类型来分发事件**  由于类型擦除的存在，会导致**Event无法按照真正的内部对象类型来分发事件**

 Spring实现Event分发的源码在**ApplicationListenerMethodAdapter.java的processEvent方法**中，其中**调用resolveArguments时就会调用event的getResolvableType方法**来作为分发判断条件之一。 



#### 1.2 解决方法

 新版本的Spring提供了一个巧妙的办法，把**真正的类型带到运行期** —— 实现 **ResolvableTypeProvider 接口** 。

```java
import lombok.Getter;
import org.springframework.core.ResolvableType;
import org.springframework.core.ResolvableTypeProvider;

@AllArgsConstructor
public class MyEvent<T> implements ResolvableTypeProvider {

    @Getter
    private String type

    @Getter
    private T data;

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(),
                ResolvableType.forInstance(data));
    }
}
```



### 参考文献

> [[Java杂技] Spring Events泛型使用方法] https://code2life.top/2019/10/31/0046-spring-event/

## restTemplate

