# Safe Java Review

Java 代码安全审查 - 结合 safe-java-code skill 和通用代码审查。

## 使用方法

```
/skill safe-java-code
```

然后运行审查：
```
/review [文件路径]
```

## 审查内容

### 并发安全
- 分布式锁与事务顺序
- 锁顺序问题
- volatile/ThreadLocal 误用

### 竞态条件
- Check-Then-Act 模式
- 双重检查锁定
- 不可变对象误用

### 数据库安全
- 更新结果未检查
- 乐观锁处理
- N+1 查询

### 基础问题
- 空指针风险
- 数组越界
- 异常未记录

### 安全漏洞
- SQL 注入
- IDOR
- 输入验证缺失
