# Java ThreadLocal 原理、内存泄漏与 Spring Security SecurityContext 完整笔记

## 一、ThreadLocal 是什么？

`ThreadLocal<T>` 是 Java 提供的线程本地变量工具，用于让不同线程独立保存和访问各自的数据。

**核心特点：**

1. 线程隔离：不同线程通过同一个 ThreadLocal 对象访问到的 Value 相互独立。
2. 线程内共享：同一线程的不同方法，只要使用同一个 ThreadLocal 对象，就能访问同一份线程本地数据。
3. 不负责线程同步：ThreadLocal 通过隔离数据避免共享，不是通过加锁解决并发问题。
4. 生命周期需要管理：线程结束通常可以释放对应数据，但在线程池中，线程可能长期存活，需要主动清理。

### 1.1 基本用法

```
public class UserContext {

    private static final ThreadLocal<String> LOCAL =
            new ThreadLocal<>();

    public static void set(String userId) {
        LOCAL.set(userId);
    }

    public static String get() {
        return LOCAL.get();
    }

    public static void remove() {
        LOCAL.remove();
    }
}
```

假设两个线程同时执行：

```
// 线程 A
UserContext.set("用户A");
System.out.println(UserContext.get()); // 用户A

// 线程 B
UserContext.set("用户B");
System.out.println(UserContext.get()); // 用户B
```

两个线程使用同一个 ThreadLocal 对象，但获取的是各自的数据。

## 二、ThreadLocal 的底层原理

### 2.1 Thread、ThreadLocalMap、ThreadLocal 的关系

每个 Java Thread 对象内部都有一个 `threadLocals` 字段：

```
ThreadLocal.ThreadLocalMap threadLocals = null;
```

ThreadLocalMap 并不是保存在 ThreadLocal 对象中，而是保存在 Thread 对象内部。

结构如下：

```
Thread A
  └── ThreadLocalMap
        ├── Entry
        │     ├── Key   → ThreadLocal1（弱引用）
        │     └── Value → 用户A（强引用）
        │
        └── Entry
              ├── Key   → ThreadLocal2（弱引用）
              └── Value → 其他数据（强引用）


Thread B
  └── ThreadLocalMap
        └── Entry
              ├── Key   → ThreadLocal1（弱引用）
              └── Value → 用户B（强引用）
```

这里需要区分：

- **Thread**：持有自己的 ThreadLocalMap。
- **ThreadLocalMap**：保存当前线程的多个本地变量。
- **ThreadLocal**：作为 ThreadLocalMap 的 Key，用于定位对应 Value。
- **Value**：当前线程真正保存的数据。

因此，ThreadLocal 本身不是存放线程数据的容器，而是访问线程本地数据的入口。

### 2.2 set() 的执行过程

```
LOCAL.set("张三");
```

简化执行过程：

1. 获取当前线程 `Thread.currentThread()`。
2. 获取当前线程的 `ThreadLocalMap`。
3. 如果 Map 不存在，则创建。
4. 以当前 ThreadLocal 对象作为 Key。
5. 保存 Value 为 `"张三"`。

可理解为：

```
Thread thread = Thread.currentThread();

ThreadLocalMap map = thread.threadLocals;

map.set(LOCAL, "张三"); // 示意代码
```

真正的源码还包括初始化、哈希定位、冲突处理和过期 Entry 清理。

### 2.3 get() 的执行过程

```
String value = LOCAL.get();
```

过程：

```
获取当前线程
      ↓
获取 ThreadLocalMap
      ↓
使用当前 ThreadLocal 作为 Key
      ↓
定位 Entry
      ↓
返回 Value
```

如果不存在对应 Entry，`get()` 会返回 `initialValue()` 的结果，默认是 `null`，并为当前线程建立对应条目。通过 `withInitial()` 创建的 ThreadLocal 则可以提供自定义初始值。

## 三、为什么 ThreadLocalMap 的 Key 使用弱引用？

ThreadLocalMap 的 Entry 继承了 WeakReference：

```
static class Entry extends WeakReference<ThreadLocal<?>> {

    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

这里：

- Key：弱引用 ThreadLocal。
- Value：普通强引用。

### 3.1 强引用和弱引用的区别

**强引用：**

```
ThreadLocal<String> local = new ThreadLocal<>();
```

只要对象存在可达的强引用，GC 就不会回收它。

**弱引用：**

```
WeakReference<Object> weak =
        new WeakReference<>(object);
```

当对象只剩下弱引用时，GC 可以将其回收。

### 3.2 为什么不把 Key 设计成强引用？

假设 ThreadLocalMap 的 Key 是强引用：

```
存活的 Thread
    ↓
ThreadLocalMap
    ↓
Entry
    ↓ 强引用
ThreadLocal
```

即使业务代码已经不再使用 ThreadLocal，只要线程长期存活，它仍然会被 ThreadLocalMap 强引用，可能一直无法回收。

改成弱引用之后：

```
存活的 Thread
    ↓
ThreadLocalMap
    ↓
Entry
    ↓ 弱引用
ThreadLocal
```

如果外部没有其他强引用，ThreadLocal 就可以被 GC 回收。

**弱引用 Key 的目的，是避免 ThreadLocalMap 本身阻止废弃的 ThreadLocal 对象被回收。**

但弱引用并不能自动解决 Value 的滞留问题。

## 四、ThreadLocal 被 GC 回收后为什么可能内存泄漏？

假设：

```
public void test() {
    ThreadLocal<byte[]> local = new ThreadLocal<>();

    local.set(new byte[1024 * 1024]);
}
```

方法结束后：

1. 局部变量 `local` 不再被使用。
2. 如果没有其他强引用，ThreadLocal 对象可以被 GC 回收。
3. ThreadLocalMap 中 Entry 的 Key 变成 `null`。
4. Entry 的 Value 仍然被强引用。
5. 如果线程长期存活，Value 可能继续占用内存。

引用关系：

```
GC 之前：

Thread
  └── ThreadLocalMap
        └── Entry
              ├── Key → ThreadLocal（弱引用）
              └── Value → byte[1MB]（强引用）


GC 之后：

Thread
  └── ThreadLocalMap
        └── Entry
              ├── Key → null
              └── Value → byte[1MB]（强引用）
```

即使 ThreadLocal 被回收，Value 也不会因为 Key 变成 null 而立即释放。

这种 Key 已经失效的 Entry 称为 **Stale Entry（过期条目）**。

### 4.1 JVM 会不会自动清理？

ThreadLocalMap 在执行部分 `get()`、`set()`、`remove()` 操作时，可能顺带清理遇到的过期 Entry。

但是：

- 不是 GC 一发生就立即清理 Value。
- 不保证下一次操作会清理全部过期 Entry。
- 如果线程长期空闲，过期 Entry 可能长期存在。

因此，不能依赖这种被动清理机制。

### 4.2 Key 不被回收就不会泄漏了吗？

不是。

例如：

```
private static final ThreadLocal<byte[]> LOCAL =
        new ThreadLocal<>();

public void test() {
    LOCAL.set(new byte[1024 * 1024]);
}
```

这里 ThreadLocal 长期具有强引用，Key 通常不会被 GC。

但如果线程池中的线程长期存活，又一直不执行 `remove()`，Value 依然可能长期存在。

**ThreadLocal 内存泄漏的核心风险，是不再需要的 Value 被长期存活的线程间接强引用，无法及时释放。**

## 五、ThreadLocal 为什么经常声明为 static final？

### 5.1 static final 创建方式

```
private static final ThreadLocal<String> LOCAL =
        new ThreadLocal<>();
```

引用关系：

```
类的静态字段
    ↓ 强引用
ThreadLocal 对象
    ↑ 弱引用
ThreadLocalMap.Entry
```

`static final` 的作用：

- `static`：整个类共享同一个 ThreadLocal 对象，而不是每创建一个类实例就创建一个。
- `final`：变量引用不能重新赋值。
- 强引用：在类及其静态字段仍然可达期间，ThreadLocal 不会因为 Map 的弱引用而被 GC。

注意：`static final` 不意味着 ThreadLocal 永远不会被回收。例如，相关类加载器被卸载后，对应静态字段可能不再可达。

### 5.2 普通成员变量创建方式

```
public class UserService {

    private final ThreadLocal<String> local =
            new ThreadLocal<>();
}
```

引用关系：

```
UserService 实例
      ↓ 强引用
ThreadLocal 对象
```

只要 UserService 实例仍然可达，ThreadLocal 就会保持可达。

如果 UserService 实例被回收，而 ThreadLocal 又没有其他强引用，则 ThreadLocal 可以被回收。

对于 Spring 单例 Bean，成员 ThreadLocal 通常也会长期存活。

**这种方式并不是错误的。** 只是每个 UserService 实例拥有不同的 ThreadLocal 对象。

### 5.3 方法内部创建方式

```
public void test() {
    ThreadLocal<String> local = new ThreadLocal<>();

    try {
        local.set("张三");
        System.out.println(local.get());
    } finally {
        local.remove();
    }
}
```

这种方式同样合法。

只要业务仍需要访问该 ThreadLocal，并且存在有效的强引用，就可以正常使用。

但是方法结束后，局部引用通常不再存在。如果没有其他强引用，ThreadLocal 可以被回收。

因此：

| 创建方式         | 典型生命周期       | 推荐使用场景           |
| ---------------- | ------------------ | ---------------------- |
| `static final`   | 类长期存活期间     | 请求上下文、用户上下文 |
| 普通实例成员变量 | 所属实例的生命周期 | 实例级线程本地状态     |
| 方法内部局部变量 | 本次使用期间       | 临时线程本地数据       |

**结论：ThreadLocal 并不必须声明成 static final。是否合适取决于需要共享 ThreadLocal 对象的范围。**

## 六、到底什么时候调用 remove()？

这是 ThreadLocal 使用中最重要的问题。

**remove() 的时机由 Value 的业务生命周期决定，而不是由 ThreadLocal 对象或线程的生命周期决定。**

### 6.1 remove() 删除的是什么？

```
LOCAL.remove();
```

清除的是：

```
当前线程的 ThreadLocalMap
    └── 对应 LOCAL 的 Entry
```

不会发生以下事情：

- 不会销毁 ThreadLocal 对象。
- 不会结束当前线程。
- 不会清理其他线程的相同 ThreadLocal 数据。
- 不会删除整个 ThreadLocalMap。

它只清除当前线程中与指定 ThreadLocal 对应的数据。

如果 Value 没有其他强引用，就可以被 GC 回收。

### 6.2 方法级使用

```
public void execute() {
    try {
        LOCAL.set("张三");

        methodA();
        methodB();

    } finally {
        LOCAL.remove();
    }
}
```

只要 methodA、methodB 还需要读取 Value，就不能提前删除。

应当在本次业务使用结束时清理。

### 6.3 线程池中使用

线程池是 ThreadLocal 内存滞留和数据污染的典型场景。

```
executorService.submit(() -> {
    try {
        LOCAL.set("用户A");

        process();

    } finally {
        LOCAL.remove();
    }
});
```

执行顺序：

```
线程 T1
  ↓
处理任务 A
  ↓
LOCAL.set("用户A")
  ↓
任务 A 结束
  ↓
LOCAL.remove()
  ↓
线程 T1 返回线程池
  ↓
接收任务 B
  ↓
LOCAL.set("用户B")
  ↓
任务 B 结束
  ↓
LOCAL.remove()
```

线程一直存活，但每次任务的数据都得到清理。

### 6.4 为什么使用 try-finally？

错误写法：

```
LOCAL.set("张三");

process();

LOCAL.remove();
```

如果 `process()` 抛出异常，`remove()` 可能不会执行。

推荐：

```
try {
    LOCAL.set("张三");
    process();
} finally {
    LOCAL.remove();
}
```

即使发生异常，也能在退出这段业务逻辑时进行清理。

### 6.5 如果多个方法都需要读取怎么办？

例如：

```
Controller
    ↓
Service
    ↓
DAO
```

如果所有方法都需要当前用户的信息，应在外层建立上下文，在整个调用链结束后统一清理。

不能在 Service 读取一次之后就提前 `remove()`，否则后续业务将无法读取。

这也正是 Spring Security 使用过滤器统一管理上下文生命周期的原因。

## 七、Spring Security 怎么使用 ThreadLocal？

### 7.1 SecurityContextHolder 是什么？

Spring Security 通过 `SecurityContextHolder` 管理当前执行上下文中的用户安全信息。

主要结构：

```
SecurityContextHolder
       ↓
SecurityContextHolderStrategy
       ↓
ThreadLocal（默认 Servlet 模式）
       ↓
SecurityContext
       ↓
Authentication
       ├── Principal
       ├── Credentials
       ├── Authorities
       ├── Details
       └── authenticated
```

其中：

**SecurityContextHolder**

提供访问安全上下文的静态入口。

**SecurityContextHolderStrategy**

定义 SecurityContext 的存储和访问策略。

**SecurityContext**

保存当前的 Authentication。

**Authentication**

表示当前用户的认证信息，例如身份、权限、认证状态等。

### 7.2 SecurityContextHolder 的存储策略

Spring Security 支持以下主要模式：

| 模式                        | 特点                                                  |
| --------------------------- | ----------------------------------------------------- |
| MODE_THREADLOCAL            | 默认策略，当前线程独立保存上下文                      |
| MODE_INHERITABLETHREADLOCAL | 使用 InheritableThreadLocal，创建子线程时可继承上下文 |
| MODE_GLOBAL                 | 全局共享上下文，不适合常规 Web 请求隔离               |

在传统 Spring MVC 应用中，默认使用 `MODE_THREADLOCAL`。

Spring Security 6 的 `ThreadLocalSecurityContextHolderStrategy` 内部使用的类型是：

```
private static final ThreadLocal<Supplier<SecurityContext>>
        contextHolder = new ThreadLocal<>();
```

它存放的是能够提供 SecurityContext 的 Supplier，而不是直接将 SecurityContext 对象作为 ThreadLocal 的 Value。

这是为了支持延迟获取上下文等机制。

### 7.3 怎么在代码中获取当前用户？

```
Authentication authentication =
        SecurityContextHolder.getContext()
                             .getAuthentication();
```

获取用户名：

```
String username = authentication.getName();
```

获取权限：

```
Collection<? extends GrantedAuthority> authorities =
        authentication.getAuthorities();
```

实际业务中需要考虑未登录时 Authentication 可能为 null，以及匿名认证的情况。

## 八、Spring Security 什么时候清理 SecurityContext？

Spring Security 会通过安全过滤器链管理当前请求的 SecurityContext。

在 Spring Security 6 的常见默认配置中，`SecurityContextHolderFilter` 负责关联和清理上下文。

### 8.1 请求处理流程

```
HTTP 请求
    ↓
SecurityFilterChain
    ↓
SecurityContextHolderFilter
    │
    ├── 从 SecurityContextRepository
    │   获取延迟加载的上下文
    │
    └── 关联到 SecurityContextHolder
    ↓
认证过滤器
    │
    ├── 验证用户身份
    └── 设置 Authentication
    ↓
Controller
    ↓
Service
    ↓
业务结束
    ↓
SecurityContextHolderFilter finally
    ↓
SecurityContextHolder.clearContext()
    ↓
线程返回 Tomcat 线程池
```

### 8.2 SecurityContextHolderFilter 原理

简化伪代码：

```
try {
    Supplier<SecurityContext> context =
            repository.loadDeferredContext(request);

    strategy.setDeferredContext(context);

    filterChain.doFilter(request, response);

} finally {
    strategy.clearContext();
}
```

其中：

```
strategy.clearContext();
```

默认 ThreadLocal 策略最终通过 `ThreadLocal.remove()` 清理当前线程中的上下文。

这意味着：

- ThreadLocal 对象可以继续存在。
- 当前请求关联的 SecurityContext 被清理。
- Tomcat 工作线程可以继续存活。
- 后续请求不能直接读取到前一个请求残留的上下文。

注意：实际过滤器链的认证、加载和保存过程取决于 Spring Security 配置，以上是核心机制的简化示意。

## 九、清理 SecurityContext 后，为什么下次请求还是登录状态？

因为需要区分：

**当前线程的安全上下文** 和 **跨 HTTP 请求的认证状态**。

ThreadLocal 负责当前执行线程访问上下文，不负责永久保存用户登录状态。

### 9.1 Session 认证模式

典型流程：

```
用户登录
    ↓
认证成功
    ↓
建立 SecurityContext
    ↓
保存到 HttpSession
    ↓
当前请求结束
    ↓
ThreadLocal 被清理

-------------------------

下次请求
    ↓
携带 Session ID
    ↓
找到对应 HttpSession
    ↓
加载 SecurityContext
    ↓
关联到当前线程
    ↓
执行 Controller
    ↓
清理当前线程上下文
```

Spring Security 可以使用 `HttpSessionSecurityContextRepository` 保存和加载 SecurityContext。

但需要注意，Spring Security 6 中，使用 `SecurityContextHolderFilter` 时通常要求认证机制按配置显式保存 SecurityContext。

`SecurityContextHolder.setContext()` 本身并不等价于持久化到 Session。

### 9.2 JWT 无状态认证模式

JWT 模式通常不会依赖服务端 HttpSession 保存 SecurityContext。

流程：

```
请求 1（携带 JWT）
    ↓
验证 JWT
    ↓
构建 Authentication
    ↓
设置 SecurityContext
    ↓
执行业务
    ↓
clearContext()

-------------------------

请求 2（再次携带 JWT）
    ↓
重新验证 JWT
    ↓
重新构建 Authentication
    ↓
设置 SecurityContext
    ↓
执行业务
    ↓
clearContext()
```

因此，每次请求都可以重新建立安全上下文。

对于自定义 JWT 过滤器，可以在认证成功后设置上下文：

```
SecurityContext context =
        SecurityContextHolder.createEmptyContext();

context.setAuthentication(authentication);

SecurityContextHolder.setContext(context);
```

如果过滤器正确集成在 Spring Security 的过滤器链内，通常由外层 SecurityContextHolderFilter 负责请求结束时的清理。

不要在仍有下游业务需要使用 SecurityContext 时提前清除。

## 十、Spring Security 与异步线程的问题

默认的 SecurityContextHolder 使用普通 ThreadLocal。

ThreadLocal **不会自动跨线程传播**。

例如：

```
SecurityContextHolder.getContext()
                     .setAuthentication(authentication);

new Thread(() -> {
    Authentication auth =
            SecurityContextHolder.getContext()
                                 .getAuthentication();
}).start();
```

新线程中的 `auth` 通常为 null。

因为新线程有自己的 ThreadLocalMap。

### 10.1 Spring Security 如何解决？

Spring Security 提供了上下文传播工具，例如：

- `DelegatingSecurityContextRunnable`
- `DelegatingSecurityContextCallable`
- `DelegatingSecurityContextExecutor`

这些工具可以将指定的 SecurityContext 关联到任务执行线程，并在任务完成后恢复或清理原有上下文。

但不建议简单依赖 `InheritableThreadLocal` 在线程池中传播认证信息，因为线程池复用可能导致上下文继承不符合实际请求边界。

另外，WebFlux 响应式应用通常通过 Reactor Context 传播安全上下文，不依赖普通 ThreadLocal 作为主要机制。

## 十一、ThreadLocal 和 SecurityContext 的生命周期对比

| 对象                   | 生命周期                       | 如何清理                                   |
| ---------------------- | ------------------------------ | ------------------------------------------ |
| Thread                 | 由线程创建、结束或线程池管理   | 线程退出                                   |
| ThreadLocalMap         | 通常跟随 Thread                | 线程结束或内部清理机制                     |
| ThreadLocal            | 由强引用可达性决定             | 不再有强引用后可被 GC                      |
| ThreadLocal 中的 Value | 由业务使用和引用关系决定       | `remove()` 解除对应 Entry 引用             |
| SecurityContextHolder  | 框架提供的长期访问入口         | 不需要每个请求销毁                         |
| SecurityContext        | 通常关联一次请求或某个认证会话 | 当前线程清理，持久化副本另行管理           |
| Authentication         | 认证信息对象                   | 不再被 SecurityContext 等对象引用后可被 GC |

注意：`remove()` 或 `clearContext()` 的效果是解除关联，不保证对象立即发生 GC。

## 十二、常见面试追问

### Q1：ThreadLocal 为什么会内存泄漏？

ThreadLocalMap 的 Entry 使用弱引用保存 Key，强引用保存 Value。如果 ThreadLocal 失去外部强引用，Key 被 GC 回收后可能变成 null，但 Value 仍然被 Entry 强引用。

线程长期存活且过期 Entry 没有及时清理时，Value 就可能长期占用内存。

即使 Key 没有被回收，只要 Value 不再需要却始终留在长期存活的线程中，也存在内存滞留风险。

### Q2：为什么 Key 使用弱引用，而 Value 使用强引用？

Key 使用弱引用，是为了避免 ThreadLocalMap 阻止不再使用的 ThreadLocal 对象被回收。

Value 使用强引用，是为了保证线程在正常使用 ThreadLocal 时，数据不会仅仅因为没有其他外部引用而被 GC 回收。

### Q3：为什么不让 Value 也使用弱引用？

因为 ThreadLocal 需要可靠保存线程本地数据。

如果 Value 仅被弱引用持有，即使业务还希望通过 ThreadLocal 读取它，也可能被 GC 回收。

因此 Value 需要强引用维持有效性，业务负责及时清理。

### Q4：ThreadLocal 必须声明为 static final 吗？

不必须。

`static final` 适用于多个方法共享同一个稳定的 ThreadLocal 实例。

实例成员变量或方法局部变量也可以使用 ThreadLocal，只要在需要时保持可达，并正确管理数据生命周期。

### Q5：static final 能防止 ThreadLocal 内存泄漏吗？

不能。

它主要保证 ThreadLocal 对象长期可达，避免 Key 因失去强引用而被 GC。

如果不调用 `remove()`，线程中的 Value 仍可能长期占用内存。

### Q6：为什么线程池容易出现 ThreadLocal 内存泄漏？

线程池会复用工作线程。

线程执行完任务后通常不会立即销毁，它内部的 ThreadLocalMap 也可能继续存在。

如果任务没有及时清理 Value，就可能导致旧数据一直占用内存，甚至污染下一个任务。

### Q7：调用 remove() 会销毁 ThreadLocal 吗？

不会。

`remove()` 只清理当前线程的 ThreadLocalMap 中对应的 Entry。

ThreadLocal 对象仍然可以通过 `set()` 继续使用。

### Q8：Spring Security 为什么使用 ThreadLocal？

因为传统 Servlet 请求通常由某个工作线程执行，同一请求处理链中的不同方法都需要方便地获取当前认证信息。

使用 ThreadLocal 可以让不同请求线程访问各自的 SecurityContext，避免通过方法参数逐层传递 Authentication。

但是异步执行或者线程切换时，需要额外处理上下文传播。

### Q9：Spring Security 为什么必须清理 SecurityContext？

因为 Tomcat 等服务器使用线程池复用工作线程。

如果不清理，上一个请求的 Authentication 可能残留在线程中，使后续请求错误读取前一个用户的认证信息，造成严重的安全问题。

### Q10：SecurityContextHolder.clearContext() 和 remove() 有什么关系？

默认 ThreadLocal 策略下，`clearContext()` 内部会通过 ThreadLocal 的 `remove()` 移除当前线程关联的上下文。

它不会清除 Session 等持久化存储中已经保存的 SecurityContext。

## 十三、最终总结

需要掌握三个不同层面的生命周期：

**第一层：ThreadLocal 对象**

ThreadLocal 是访问线程本地数据的 Key。

可以声明为 `static final`，通过稳定强引用使它长期存在，并被不同线程重复使用。

**第二层：线程本地 Value**

Value 才是真正存储在 ThreadLocalMap 中的数据。

应该在一次业务、任务或请求不再需要它时调用 `remove()`，而不是等待线程结束或 GC。

**第三层：Spring Security 安全上下文**

Spring Security 默认通过 ThreadLocal 策略让当前线程访问 SecurityContext。

在 Servlet 请求中，过滤器负责建立、加载和清理当前执行上下文。跨请求登录状态由 Session、JWT 等机制处理，而不是依赖 ThreadLocal 一直保存。

### 一句话记忆

> **ThreadLocal 负责线程隔离，ThreadLocalMap 负责实际存储，弱引用 Key 避免无用 ThreadLocal 被 Map 长期持有，remove() 负责及时清理线程本地数据。Spring Security 则利用这一机制，在请求期间关联 SecurityContext，在请求结束后清理上下文，避免线程池复用造成认证信息污染。**