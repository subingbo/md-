# RuoYi-Vue-Plus 启动异常排查总结

------

## 一、问题背景

使用 **RuoYi-Vue-Plus 5.6.0** 开发时，最初项目可以正常运行，但在切换 JDK 版本后出现一系列问题。

------

## 二、问题现象

1. 编译失败（`setUpdateBy` / `BaseMapperPlus` 等找不到）
2. Maven 打包失败（`Unable to find main class`）
3. IDEA 启动失败（`Could not find or load main class`）
4. 运行时报错（`ClassFormatError: Duplicate method name "convert"`）

------

## 三、根本原因

**一切的根因只有一个：使用了不兼容的 JDK 24。**

RuoYi-Vue-Plus 5.6.0 基于 JDK 17 开发和测试，使用 JDK 24 会导致一条完整的错误链：

```
使用 JDK 24 编译
→ RuoYi 自身代码编译失败（setUpdateBy / BaseMapperPlus 等找不到）
→ 手动修复后切换 JDK，但 target 残留 JDK 24 的脏 class 文件
→ MapStruct 生成类字节码冲突（ClassFormatError）
→ Spring 上下文初始化失败，应用无法启动
```

------

## 四、关键认知

**"刚刚还能运行"不代表没问题。**

IDEA 直接运行使用的是旧的编译缓存，编译已经失败但没有立刻暴露。执行 `mvn clean` 清掉缓存后，才强制暴露了真实的编译错误。

```
能运行 ≠ 正确
clean 后还能运行才是正确
```

------

## 五、解决方法

1. 将 JDK 统一切换为 **JDK 17**（IDEA、Maven、运行配置三处保持一致）
2. 清理并重新编译问题模块：

```bash
mvn clean -pl ruoyi-modules/ruoyi-system
mvn compile -pl ruoyi-modules/ruoyi-system -am
```

1. 在 IDEA 中 `Build → Rebuild Project`，重启应用

------

## 六、预防措施

每次切换 JDK 或拉取新代码后执行：

```bash
mvn clean install -DskipTests
```

确保编译产物始终干净。

------

## 七、一句话结论

**JDK 版本不兼容是根因，编译失败是表象，target 脏文件是导火索。**