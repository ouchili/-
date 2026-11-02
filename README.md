# Universal Component Runtime — M1

一个独立于具体游戏的第二层通用组件执行内核。当前实现范围严格限制为：

- 组件注册与按 `id + version` 加载
- 输入、参数与目标选择器校验
- 统一执行上下文与确定性 RNG
- Creation、Expansion、Branching 三个最小组件
- 节点数、调用数、深度与逻辑时间预算
- 单组件执行结果与测试

## 环境

- Node.js 20+
- npm

```bash
npm install
npm run typecheck
npm test
npm run demo
```

## 最小执行示例

```ts
import {
  ComponentRegistry,
  ComponentRuntime,
  createDefaultRegistry,
  emptyStructure
} from "./src/index.js";

const registry = createDefaultRegistry();
const runtime = new ComponentRuntime(registry);

const result = runtime.execute({
  instanceId: "create-1",
  componentId: "Creation",
  componentVersion: "1.0.0",
  parameters: {
    nodeType: "plant",
    attrs: { color: "green" }
  },
  target: { kind: "all" }
}, {
  structure: emptyStructure(),
  seed: 12345,
  limits: {
    maxNodes: 100,
    maxCalls: 20,
    maxDepth: 10,
    maxTimeMs: 100
  }
});

console.log(result.status, result.structure);
```

## 注册新组件

实现 `Component` 接口，然后注册：

```ts
registry.register(myComponent);
```

组件定义按 `id + version` 唯一定位。注册后视为不可变；不要在运行过程中修改组件定义。

## 执行 ComponentInstance

通过 `ComponentRuntime.execute(instance, environment)` 执行。执行结果包含状态、执行后的结构、复现记录、错误和资源使用量。

## 当前三个组件的最小语义

原始规格只指定了三个组件名称，没有给出它们各自的精确操作语义。因此本实现采用最小、可测试的临时定义，并在代码中标注为 `TODO/待定`：

- **Creation**：创建一个新节点；若目标选择器恰好选中一个节点，则将新节点挂到该节点之下。
- **Expansion**：对每个目标节点创建指定数量的子节点，并建立 `expansion` 关系。
- **Branching**：对每个目标节点创建指定数量的分支子节点，并建立 `branch` 关系。

这些语义只为跑通 M1 协议，不应视为最终组件库定义。

## 重要实现说明

1. `Structure.nodes` 使用普通对象而非原生 `Map`，便于 JSON 比较、测试与后续序列化；对象划分保持不变。
2. 随机性只通过 `ExecutionContext.rng` 提供的种子 RNG。
3. `usage.timeMs` 当前是**确定性的逻辑时间单位**，不是墙钟耗时。这样同一输入可得到逐字段一致的结果，也能测试时间预算。未来若需要真实耗时监控，应将它作为独立诊断指标，不纳入确定性结果。
4. 每次执行都在结构副本上运行；组件失败或超限时不会污染调用方传入的原始结构。结构入口会检查节点、关系引用、层级父子关系、多父节点与循环。
5. `conditions` 暂存于实例中，本阶段不解释、不执行。
6. `target` 当前支持 `all`、`ids`、`type` 三种选择方式；条件表达式筛选留待后续阶段。
7. `maxDepth` 以目标节点当前层级深度加本次创建层级计算；没有目标父节点时，新节点深度为 0。
8. `seed` 必须是安全整数；四项资源限制必须是非负安全整数。非法环境输入返回 `error`，不会启动组件。
