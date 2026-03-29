# 测试反模式

**加载此参考当：** 编写或更改测试、添加 mock、或试图向生产代码添加仅测试用的方法时。

## 概述

测试必须验证真实行为，而不是 mock 行为。Mock 是隔离的手段，而不是被测试的东西。

**核心原则：** 测试代码做什么，而不是 mock 做什么。

**遵循严格的 TDD 可以防止这些反模式。**

## 铁律

```
1. 永远不要测试 mock 行为
2. 永远不要向生产类添加仅测试用的方法
3. 永远不要在不理解依赖的情况下 mock
```

## 反模式 1：测试 Mock 行为

**违规：**
```typescript
// ❌ 错误：测试 mock 是否存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么这是错的：**
- 你在验证 mock 是否工作，而不是组件是否工作
- Mock 存在时测试通过，不存在时失败
- 完全没有说明真实行为

**你的搭档的纠正：** "我们是在测试 mock 的行为吗？"

**修复：**
```typescript
// ✅ 好的：测试真实组件或不 mock 它
test('renders sidebar', () => {
  render(<Page />);  // 不要 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果必须为隔离而 mock sidebar：
// 不要对 mock 断言 - 测试在有 sidebar 时 Page 的行为
```

### 门槛函数

```
在断言任何 mock 元素之前：
  问："我是在测试真实组件行为还是仅仅测试 mock 的存在？"

  如果是测试 mock 存在：
    停止 - 删除断言或取消 mock 组件

  改为测试真实行为
```

## 反模式 2：生产代码中的仅测试方法

**违规：**
```typescript
// ❌ 错误：destroy() 仅在测试中使用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试中
afterEach(() => session.destroy());
```

**为什么这是错的：**
- 生产类被仅测试代码污染
- 如果在生产中意外调用会很危险
- 违反 YAGNI 和关注点分离
- 混淆对象生命周期与实体生命周期

**修复：**
```typescript
// ✅ 好的：测试工具处理测试清理
// Session 没有 destroy() - 在生产中是无状态的

// 在 test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试中
afterEach(() => cleanupSession(session));
```

### 门槛函数

```
在向生产类添加任何方法之前：
  问："这仅被测试使用吗？"

  如果是：
    停止 - 不要添加它
    改为放在测试工具中

  问："这个类是否拥有此资源的生命周期？"

  如果不是：
    停止 - 这个方法放错了类
```

## 反模式 3：不理解依赖就 Mock

**违规：**
```typescript
// ❌ 错误：Mock 破坏了测试逻辑
test('detects duplicate server', () => {
  // Mock 阻止了测试依赖的配置写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛出 - 但不会！
});
```

**为什么这是错的：**
- Mock 的方法有测试依赖的副作用（写入配置）
- 过度 mock"为了安全"破坏了实际行为
- 测试因错误原因通过或神秘失败

**修复：**
```typescript
// ✅ 好的：在正确的层级 mock
test('detects duplicate server', () => {
  // Mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只需 mock 慢的服务器启动

  await addServer(config);  // 配置写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 门槛函数

```
在 mock 任何方法之前：
  停止 - 先不要 mock

  1. 问："真实方法有什么副作用？"
  2. 问："这个测试是否依赖这些副作用中的任何一个？"
  3. 问："我完全理解这个测试需要什么吗？"

  如果依赖副作用：
    在更低层级 mock（实际的慢/外部操作）
    或者使用保留必要行为的测试替身
    而不是测试依赖的高级方法

  如果不确定测试依赖什么：
    先用真实实现运行测试
    观察实际需要什么
    然后在正确的层级添加最小化的 mock

  红旗：
    - "我会 mock 这个以确保安全"
    - "这可能很慢，最好 mock 它"
    - 不理解依赖链就 mock
```

## 反模式 4：不完整的 Mock

**违规：**
```typescript
// ❌ 错误：部分 mock - 只有你需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的 metadata
};

// 之后：当代码访问 response.metadata.requestId 时崩溃
```

**为什么这是错的：**
- **部分 mock 隐藏了结构假设** - 你只 mock 了你知道的字段
- **下游代码可能依赖你未包含的字段** - 静默失败
- **测试通过但集成失败** - Mock 不完整，真实 API 完整
- **虚假信心** - 测试无法证明真实行为

**铁律：** Mock 完整的**真实存在的数据结构**，而不仅仅是你 immediate 测试使用的字段。

**修复：**
```typescript
// ✅ 好的：镜像真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### 门槛函数

```
在创建 mock 响应之前：
  检查："真实 API 响应包含什么字段？"

  操作：
    1. 从文档/示例检查实际 API 响应
    2. 包含系统可能在下游消费的所有字段
    3. 验证 mock 与真实响应模式完全匹配

  关键：
    如果你在创建 mock，你必须理解整个结构
    当代码依赖被忽略的字段时，部分 mock 会静默失败

  如果不确定：包含所有文档化的字段
```

## 反模式 5：集成测试是后记

**违规：**
```
✅ 实现完成
❌ 没有写测试
"准备好测试了"
```

**为什么这是错的：**
- 测试是实现的一部分，不是可选的后续
- TDD 会发现这个问题
- 没有测试不能说完成

**修复：**
```
TDD 周期：
1. 写失败的测试
2. 实现使测试通过
3. 重构
4. 然后才能说完成
```

## 当 Mock 变得太复杂时

**警告信号：**
- Mock 设置比测试逻辑还长
- Mock 一切让测试通过
- Mock 缺少真实组件有的方法
- 当 mock 改变时测试失败

**你的搭档的问题：** "我们这里需要使用 mock 吗？**

**考虑：** 使用真实组件的集成测试通常比复杂 mock 更简单

## TDD 防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 迫使你思考你实际在测试什么
2. **看它失败** → 确认测试测试真实行为，而不是 mock
3. **最小化实现** → 不会有仅测试的方法混入
4. **真实依赖** → 在 mock 之前你看到测试实际需要什么

**如果你在测试 mock 行为，你违反了 TDD** — 你在先看测试对真实代码失败之前就添加了 mock。

## 快速参考

| 反模式 | 修复 |
|--------------|-----|
| 对 mock 元素断言 | 测试真实组件或取消 mock |
| 生产中的仅测试方法 | 移到测试工具 |
| 不理解就 mock | 先理解依赖，最小化 mock |
| 不完整的 mock | 完全镜像真实 API |
| 测试是后记 | TDD - 测试优先 |
| 过度复杂的 mock | 考虑集成测试 |

## 红旗

- 断言检查 `*-mock` test ID
- 方法仅在测试文件中调用
- Mock 设置占测试的 50% 以上
- 当你移除 mock 时测试失败
- 无法解释为什么需要 mock
- "只是为了安全"而 mock

## 底线

**Mock 是隔离的工具，不是被测试的东西。**

如果 TDD 显示你在测试 mock 行为，你就走错了。

修复：测试真实行为或质疑你为什么 mock。