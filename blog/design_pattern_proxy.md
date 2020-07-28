# 设计模式：代理模式

## 代理模式

```java
public interface UserService {
    String getASingleUser(String username);
}

public class OkHttpUserService implements UserService {
    @Override
    public String getASingleUser(String username) {
        System.out.println(String.format(Locale.US, "OkHttp do call: getASingleUser(%s)", username));
        return String.join("-", "UserInfo", username);
    }
}

public class RetrofitUserService implements UserService {
    UserService service;
    RetrofitUserService(UserService obj) {
        this.service = obj;
    }
    @Override
    public String getASingleUser(String username) {
        System.out.println(String.format(Locale.US, "Proxy do call: getASingleUser(%s)", username));
        String userParam = username.toUpperCase(Locale.US);
        System.out.println(String.format(Locale.US, "Proxy do something before OkHttp call, args: %s -> %s", username, userParam));
        String userIfo = service.getASingleUser(userParam);
        String response = userIfo.toUpperCase(Locale.US);
        System.out.println(String.format(Locale.US, "Proxy do something after OkHttp call, ret: %s -> %s", userIfo, response));
        return response;
    }
}

OkHttpUserService okHttpUserService = new OkHttpUserService();
RetrofitUserService proxy = new RetrofitUserService(okHttpUserService);
String userInfo = proxy.getASingleUser("google");
System.out.println(String.format(Locale.US, "Final result: %s", userInfo));
```

```text
Proxy do call: getASingleUser(google)
Proxy do something before OkHttp call, args: google -> GOOGLE
OkHttp do call: getASingleUser(GOOGLE)
Proxy do something after OkHttp call, ret: UserInfo-GOOGLE -> USERINFO-GOOGLE
Final result: USERINFO-GOOGLE
```

代理对象和被代理的对象实现相同的接口，同时代理对象会持有被代理对象，完成对被代理对象的控制和访问  
这种传统的代理模式也被称为静态代理  

## 动态代理

还有一种代理模式可以在运行时生成代理类和代理对象，这种代理模式也被称为动态代理  
代理桥梁通过 `InvocationHandler` 实现，如：  

```java
public interface UserService {
    String getASingleUser(String username);
}

InvocationHandler handler = new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(String.format(Locale.US, "Proxy do invoke: invoke(proxy, %s, %s)", method.getName(), Arrays.toString(args)));
        String username = (String) args[0];
        String userParam = username.toUpperCase(Locale.US);
        System.out.println(String.format(Locale.US, "Proxy do something before OkHttp call, args: %s -> %s", username, userParam));
        System.out.println(String.format(Locale.US, "OkHttp do call: %s(%s)", method.getName(), userParam));
        String userIfo = String.join("-", "UserInfo", userParam);
        String response = userIfo.toUpperCase(Locale.US);
        System.out.println(String.format(Locale.US, "Proxy do something after OkHttp call, ret: %s -> %s", userIfo, response));
        return response;
    }
};
UserService proxy = (UserService) Proxy.newProxyInstance(OkHttpUserService.class.getClassLoader(),
        new Class<?>[] { UserService.class }, handler);
String userInfo = proxy.getASingleUser("google");
System.out.println(String.format(Locale.US, "Final result: %s", userInfo));
```

```text
Proxy do invoke: invoke(proxy, getASingleUser, [google])
Proxy do something before OkHttp call, args: google -> GOOGLE
OkHttp do call: getASingleUser(GOOGLE)
Proxy do something after OkHttp call, ret: UserInfo-GOOGLE -> USERINFO-GOOGLE
Final result: USERINFO-GOOGLE
```

可以看到我们不需要书写代理类，所有对代理对象的方法调用就会被转发到 `InvocationHandler` 的 `invoke` 回调中  
只需要通过 `Proxy` 的静态方法 `newProxyInstance` 就可以获取代理对象，那究竟怎么生成代理对象的类并创建对象的呢？  
当然是通过反射，先继承 `Proxy` 并实现 `UserService` :  

```java
visit(V14, accessFlags, dotToSlash(className), null,
    JLR_PROXY, typeNames(interfaces));
```

然后添加对象的 `hashCode()`, `equals()`, `toString()` 方法：  

```java
addProxyMethod(hashCodeMethod);
addProxyMethod(equalsMethod);
addProxyMethod(toStringMethod);
```

然后添加给定接口中的所有方法：  

```java
for (Class<?> intf : interfaces) {
    for (Method m : intf.getMethods()) {
        if (!Modifier.isStatic(m.getModifiers())) {
            addProxyMethod(m, intf);
        }
    }
}
```

然后添加参数为 `InvocationHandler` 的构造器，以及把方法都声明成静态常量并用静态代码快初始化就完成代理类的生成了：  

```java
generateConstructor();
for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
    for (ProxyMethod pm : sigmethods) {
        // add static field for the Method object
        visitField(Modifier.PRIVATE | Modifier.STATIC, pm.methodFieldName,
                LJLR_METHOD, null, null);

        // Generate code for proxy method
        pm.generateMethod(this, className);
    }
}
generateStaticInitializer();
```

最后调用刚生成的代理类的构造器就可以创建代理对象了：  

```java
return cons.newInstance(new Object[]{h});
```

现在即使我们没有看见生成的代理类是什么样子，我们也清楚了它是什么结构了，它继承了 `Proxy` 并实现了我们给定的接口，它的构造器有 `InvocationHandler` 参数，它重写了 `hashCode()`, `equals()`, `toString()` 方法并把它们都转发给 `InvocationHandler` 的 `invoke` 处理  
它实现了我们给定接口的所有方法，并把它们都转发给 `InvocationHandler` 的 `invoke` 处理  
所以代理类肯定类似于这样：

```java
public final class $1 extends Proxy implements UserService {
    private static Method m0;
    private static Method m1;
    private static Method m2;
    private static Method m3;
    static {
        m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
        m2 = Class.forName("java.lang.Object").getMethod("toString");
        m3 = Class.forName("com.frank.UserService").getMethod("getASingleUser");
    }
    public $1(InvocationHandler var1) {
        super(var1);
    }
    public final int hashCode() {
        return ((Integer) super.h.invoke(this, m0, (Object[]) null)).intValue();
    }
    public final boolean equals(Object var1) {
        return ((Boolean) super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
    }
    public final String toString() {
        return (String) super.h.invoke(this, m2, (Object[]) null);
    }
    public final String getASingleUser(String var1) {
        return (String) super.h.invoke(this, m3, new String[]{var1});
    }
}
```

可见动态代理有时候比静态代理更加的灵活，我们不需要手动实现代理类，也不需要手动创建代理对象，就可以实现代理的功能，但是动态代理利用了反射技术，所以性能没有静态代理那么好  
总之动态代理技术让灵活实现代理成为可能，Retrofit 也正是利用了动态代理技术完成了对 HTTP 请求的优雅封装  
