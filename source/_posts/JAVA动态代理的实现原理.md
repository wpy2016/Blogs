title: JAVA动态代理的实现原理
date: 2020-1-9 01:39:04
tags:
  - JAVA
  - 设计模式
---



## 定义

代理：对象结构型模式，为其他对象提供一种代理其控制对这个对象的访问

动态代理的特点：代理接口可以在运行时改变，代理控制方式（InvocationHandler）可以在运行时改变，代理的对象可以在运行时改变，代理者和被代理者的关系是完全动态的

实现方式：JAVA实现， CGLIB实现。前者主要依托于Proxy.newProxyInstance()， 后者底层使用ASM字节码框架实现。

## 优势缺点及应用场景

动态代理使用场景：通过代理可以扩展现有实现类的功能（符合开闭原则），可以将所有函数的公共部分抽取出来，简化代码。

例如：

1. RPC接口调用中，都需要对调用者的权限进行控制，因此使用动态代理包装RPC实例，在InvocationHandler.invoke方法中，对访问者进行权限检查，再调用具体PRC接口。
2. 需要统计某个实例每个方法的调用耗时时间，调用次数，可以使用动态代理

## 实例演示

下面的例子功能：查询用户是否已经存在。类图后面展示。

### 查询用户接口

```java
public interface IQuery {
    boolean isUserExist(String username);
}
```

### 查询用户接口实现

```java
public class QueryImpl implements IQuery {
    @Override
    public boolean isUserExist(String username) {
        System.out.println("query " + username + " is exist or not?");
        if (username.equals("xiaobai")) {
            return true;
        } else {
            return false;
        }
    }
}
```

### InvocationHandle实现接口

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class QueryInvocationHandle implements InvocationHandler {

    private IQuery query;

    public QueryInvocationHandle(IQuery query) {
        this.query = query;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before method invoke");
        System.out.println("invoke method name is " + method.getName());
        boolean isExist = (boolean)method.invoke(query, args);
        System.out.println("after method invoke");
        return isExist;
    }
}
```

### Main

```java
import java.lang.reflect.Proxy;

public class Main {

    public static void main(String[] args) {
        Exec();
    }

    public static void Exec(){
        QueryImpl query = new QueryImpl();
        QueryInvocationHandle queryInvocationHandle = new QueryInvocationHandle(query);
        IQuery proxyInstance = (IQuery)Proxy.newProxyInstance(query.getClass().getClassLoader(), query.getClass().getInterfaces(), queryInvocationHandle);
        System.out.println(proxyInstance.isUserExist("xiaobai"));
        System.out.println("finish");
    }
}
```

### 输出

```
before method invoke
invoke method name is isUserExist
query xiaobai is exist or not?
after method invoke
true
finish
```



## 实现原理

基于上面的例子，我们探究下其底层的实现原理：

上面这个例子类图如下：

![java_dynamic_png1](https://github.com/wpy2016/wpy2016.github.io/blob/master/imgs/java_dynamic_proxy/java_dynamic_png1.png?raw=true)



其中类$Proxy0是动态代理实现原理的关键，由ProxyClassFactory生成。对应时序图如下：

![java_dynamic_png2](https://github.com/wpy2016/wpy2016.github.io/blob/master/imgs/java_dynamic_proxy/java_dynamic_png2.png?raw=true)



下面将$Proxy0动态代理类的字节码反编译，其代码实现为：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class Proxy0 extends Proxy implements IQuery {
  private static Method m1;
  
  private static Method m2;
  
  private static Method m3;
  
  private static Method m0;
  
  public Proxy0(InvocationHandler paramInvocationHandler) { super(paramInvocationHandler); }
  
  public final boolean equals(Object paramObject) {
    try {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  public final String toString() {
    try {
      return (String)this.h.invoke(this, m2, null);
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  public final boolean isUserExist(String paramString) {
    try {
      return ((Boolean)this.h.invoke(this, m3, new Object[] { paramString })).booleanValue();
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  public final int hashCode() {
    try {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    } catch (Error|RuntimeException error) {
      throw null;
    } catch (Throwable throwable) {
      throw new UndeclaredThrowableException(throwable);
    } 
  }
  
  static  {
    try {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m3 = Class.forName("IQuery").getMethod("isUserExist", new Class[] { Class.forName("java.lang.String") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    } catch (NoSuchMethodException noSuchMethodException) {
      throw new NoSuchMethodError(noSuchMethodException.getMessage());
    } catch (ClassNotFoundException classNotFoundException) {
      throw new NoClassDefFoundError(classNotFoundException.getMessage());
    } 
  }
}
```

由此可以看到Proxy.newProxyInstance返回实例的调用，本质上如下的调用关系：

![java_dynamic_png3](https://github.com/wpy2016/wpy2016.github.io/blob/master/imgs/java_dynamic_proxy/java_dynamic_png3.png?raw=true)



结束。

补充：

生成$Proxy0代理类字节码的方法。

```java
//proxyName, interfaces, accessFlags
private byte[] generateClassFile() {
    this.addProxyMethod(hashCodeMethod, Object.class);
    this.addProxyMethod(equalsMethod, Object.class);
    this.addProxyMethod(toStringMethod, Object.class);
    Class[] var1 = this.interfaces;
    int var2 = var1.length;

    int var3;
    Class var4;
    for(var3 = 0; var3 < var2; ++var3) {
        var4 = var1[var3];
        Method[] var5 = var4.getMethods();
        int var6 = var5.length;

        for(int var7 = 0; var7 < var6; ++var7) {
            Method var8 = var5[var7];
            this.addProxyMethod(var8, var4);
        }
    }

    Iterator var11 = this.proxyMethods.values().iterator();

    List var12;
    while(var11.hasNext()) {
        var12 = (List)var11.next();
        checkReturnTypes(var12);
    }

    Iterator var15;
    try {
        this.methods.add(this.generateConstructor());
        var11 = this.proxyMethods.values().iterator();

        while(var11.hasNext()) {
            var12 = (List)var11.next();
            var15 = var12.iterator();

            while(var15.hasNext()) {
                ProxyGenerator.ProxyMethod var16 = (ProxyGenerator.ProxyMethod)var15.next();
                this.fields.add(new ProxyGenerator.FieldInfo(var16.methodFieldName, "Ljava/lang/reflect/Method;", 10));
                this.methods.add(var16.generateMethod());
            }
        }

        this.methods.add(this.generateStaticInitializer());
    } catch (IOException var10) {
        throw new InternalError("unexpected I/O Exception", var10);
    }

    if (this.methods.size() > 65535) {
        throw new IllegalArgumentException("method limit exceeded");
    } else if (this.fields.size() > 65535) {
        throw new IllegalArgumentException("field limit exceeded");
    } else {
        this.cp.getClass(dotToSlash(this.className));
        this.cp.getClass("java/lang/reflect/Proxy");
        var1 = this.interfaces;
        var2 = var1.length;

        for(var3 = 0; var3 < var2; ++var3) {
            var4 = var1[var3];
            this.cp.getClass(dotToSlash(var4.getName()));
        }

        this.cp.setReadOnly();
        ByteArrayOutputStream var13 = new ByteArrayOutputStream();
        DataOutputStream var14 = new DataOutputStream(var13);

        try {
            var14.writeInt(-889275714);
            var14.writeShort(0);
            var14.writeShort(49);
            this.cp.write(var14);
            var14.writeShort(this.accessFlags);
            var14.writeShort(this.cp.getClass(dotToSlash(this.className)));
            var14.writeShort(this.cp.getClass("java/lang/reflect/Proxy"));
            var14.writeShort(this.interfaces.length);
            Class[] var17 = this.interfaces;
            int var18 = var17.length;

            for(int var19 = 0; var19 < var18; ++var19) {
                Class var22 = var17[var19];
                var14.writeShort(this.cp.getClass(dotToSlash(var22.getName())));
            }

            var14.writeShort(this.fields.size());
            var15 = this.fields.iterator();

            while(var15.hasNext()) {
                ProxyGenerator.FieldInfo var20 = (ProxyGenerator.FieldInfo)var15.next();
                var20.write(var14);
            }

            var14.writeShort(this.methods.size());
            var15 = this.methods.iterator();

            while(var15.hasNext()) {
                ProxyGenerator.MethodInfo var21 = (ProxyGenerator.MethodInfo)var15.next();
                var21.write(var14);
            }

            var14.writeShort(0);
            return var13.toByteArray();
        } catch (IOException var9) {
            throw new InternalError("unexpected I/O Exception", var9);
        }
    }
}
```



## Plant UML绘图代码

以上图采用PlantUML绘制，对应代码如下：

类图：

```
@startuml
title: 例子类图
abstract class Proxy
class $Proxy0
interface IQuery
class QueryImpl
interface InvocationHandler
class QueryInvocationHandle

IQuery <|.. QueryImpl
InvocationHandler <|.. QueryInvocationHandle
QueryImpl <.. QueryInvocationHandle
Proxy <|-- $Proxy0
IQuery <|.. $Proxy0
QueryInvocationHandle <.. $Proxy0
ProxyClassFactory <-- Proxy
ProxyGenerator <.. ProxyClassFactory

class IQuery {
isUserExist()
}

class QueryImpl {
isUserExist()
}

class QueryInvocationHandle {
invoke()
QueryImpl
}

class InvocationHandler {
invoke()
}

class $Proxy0 {
Method m1
Method m2
Method m3
Method m4
equals()
toString()
isUserExist()
hashCode()
}

class Proxy {
newProxyInstance()
defineClass0()
proxyClassCache
}

class ProxyClassFactory {
proxyClassNamePrefix
apply()
}

class ProxyGenerator {
generateProxyClass()
}
@enduml
```

获取动态代理时序图：

```
@startuml
title: 获取动态代理实例时序图
Main -> Proxy: newProxyInstance
Proxy -> Proxy: getProxyClass0
Proxy -> WeakCache: get
WeakCache -> WeakCache.Factory: get
WeakCache.Factory -> ProxyClassFactory: apply
ProxyClassFactory -> ProxyGenerator: generateProxyClass
ProxyClassFactory -> ProxyClassFactory: defineClass0
ProxyClassFactory -> WeakCache.Factory: save $Proxy to map & return
WeakCache.Factory -> WeakCache: return $Proxy
WeakCache -> Proxy: return $Proxy
Proxy -> Proxy: $Proxy.newInstance
Proxy -> Main: return $Proxy instance
@enduml
```

动态代理实例的调用流程:

```
@startuml
title: 动态代理实例的调用流程
Main -> Proxy: newProxyInstance
Proxy -> Main: return $Proxy Instance
Main -> $Proxy0: isUserExist
$Proxy0 -> QueryInvocationHandle: invoke
QueryInvocationHandle -> QueryImpl: 反射调用isUserExist
 Proxy <|.. $Proxy0
 IQuery <|.. $Proxy0
@enduml
```

