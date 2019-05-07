Object所有的非final方法(equals、hashCode、toString、clone和finalize)都有明确的通用约定，因为他们被设计成是可覆盖的；故覆盖这些方法有责任遵守约定。

### 第8条：覆盖equals时请遵守通用约定
- equals方法看似简单但容易出错且后果严重，尽量避免覆盖equqls方法。
1. 类的每个实例本质上都是唯一的。对于代表活动实体而不是值(value