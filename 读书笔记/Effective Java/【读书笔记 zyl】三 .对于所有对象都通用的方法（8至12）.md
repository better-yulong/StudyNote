Object所有的非final方法(equals、hashCode、toString、clone和finalize)都有明确的通用约定，因为他们被设计成是可覆盖的；故覆盖这些方法有责任遵守约定。

### 第8条：覆盖equals时请遵守通用约定
- equals方法看似简单但容易出错且后果严重，尽量避免覆盖equqls方法。
1. 类的每个实例本质上都是唯一的。对于代表活动实体而不是值(value）的类来说确实如此，如Thread。Object类提供的默认equals实现对于这些类来说正是正确的行为，故不应该覆盖。
2. 不关心类是否提供了"逻辑相等"(logical equality)的测试功能。
3. 超类已经覆盖了equals，从超类继承过来的行为对子类也是合适的。如大多数Set实现都是AbstraceSet继承Set实现，List实现从AbstractList继承equals实现，Map实现从AbstractMap继承equals实现。另外发现个新知识点，Set、
```language

```

4. 