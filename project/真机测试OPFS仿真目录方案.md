# 真机测试 OPFS 仿真目录方案

## 元信息

- **性质**：测试环境配方（非施工计划）——记录「浏览器无法授予文件系统权限时如何照常真机测试」的完整做法
- **首次使用**：2026-10-20，[要素管理员提示词升级施工计划](../plans/要素管理员提示词升级施工计划.md) W4 真机回归（十验收点全过）
- **维护约定**：方案本身稳定不常改；每次使用后在文末「使用记录」追加一行

## 适用场景

NovAI 的真机测试需要打开一个真实项目目录（如 `~/note/test-novel`），而网页读写本地目录依赖 File System Access API 的授权面板。在以下环境该面板**弹不出来**，授权请求被自动拒绝并持久化为 `denied`，导致「恢复项目」「打开项目」全部失败：

- ZCode 内置浏览器（IAB）——无文件系统授权 UI（2026-10-20 实测）
- 任何嵌入式 WebView / 无头浏览器——同理（未逐一实测）

注意这是**环境限制，不是应用缺陷**。真实浏览器（Chrome 等）不受影响。

## 诊断步骤（先确诊再动手）

1. 正常路径点击「恢复项目」，页面无反应、无报错 → 怀疑权限层。
2. 在页面上下文探测存储句柄的真实权限状态（从 IndexedDB `novai-projects` 库 `recent-projects` 表取出记录的 handle）：
   - `handle.queryPermission({ mode: 'readwrite' })` 返回 `denied`
   - `handle.requestPermission({ mode: 'readwrite' })` 仍返回 `denied` 且无任何面板出现
3. 两条都 denied 即确诊：该环境无法授予本地目录权限，走本方案。

## 方案原理

OPFS（Origin Private File System，`navigator.storage.getDirectory()`）是浏览器标准 API，返回一个**真实的** `FileSystemDirectoryHandle`——`values()`、`getDirectoryHandle`、`getFileHandle`、`createWritable`、`removeEntry` 全部原生可用，且**天生免授权**（页面自己的沙箱）。

应用侧两个特性让「换柄」可以透明生效：

1. 最近项目记录存在 IndexedDB（`novai-projects` 库），每条记录的 `handle` 字段可整体替换；
2. core 文件层对句柄是鸭子类型调用（无 `instanceof` / `isSameEntry` 硬检查）。

因此：把项目文件副本种进 OPFS，再把记录的句柄换成 OPFS 根，应用的全链路（迁移逻辑、UI、真实 LLM 调用、工具执行、确认链）都在真实代码上运行，**只有存储传输层换了载体**。真实项目目录零改动。

被排除的替代方案（试过或评估过，勿重走）：

| 替代方案 | 不可行原因 |
| :-- | :-- |
| IndexedDB 里存自造的假句柄对象 | 结构化克隆会剥掉对象方法，句柄存不进去（读回是裸对象）；运行时也会 `DataCloneError` |
| 「打开项目」走目录选择器 | IAB 不支持原生文件选择对话框 |
| 等待权限面板 | 该环境根本渲染不出面板 |

## 操作配方

### 第一步：盘外构建种子（shell 侧）

从真实项目目录读出需要的文件（配置、提示词、要素、`.novai/` 下的预设等），打包成一个 JSON（`{ "files": { "<相对路径>": "<文本内容>" } }），放在 `/tmp` 下。二进制文件不进种子（本项目全是文本）。

需要测「存量迁移」时，在这一步向 `novel.config.json` 注入旧字段块（如 `gardener`）——种子是测试数据，可以自由做手脚。

注意：`novel.config.json` 含 API key，种子文件只放 `/tmp`，页面用完即弃；key 在页面内存中运行时使用，不写进任何仓库文件。

### 第二步：页面内播种 OPFS（浏览器自动化 evaluate）

把种子 JSON 作为 evaluate 参数传入页面，在页面上下文执行：

```js
const root = await navigator.storage.getDirectory();
// 1) 清空 OPFS 根
for await (const entry of root.values()) {
  await root.removeEntry(entry.name, { recursive: true }).catch(() => {});
}
// 2) 逐文件写入：按路径逐级建目录、建文件、写内容
for (const [path, content] of Object.entries(seed.files)) {
  const parts = path.split('/');
  const fileName = parts.pop();
  let dir = root;
  for (const part of parts) dir = await dir.getDirectoryHandle(part, { create: true });
  const fh = await dir.getFileHandle(fileName, { create: true });
  const w = await fh.createWritable();
  await w.write(new TextEncoder().encode(content));
  await w.close();
}
```

### 第三步：换柄并恢复

仍在页面上下文，打开 `novai-projects` 库，把 `recent-projects` 表所有记录的 `handle` 字段替换为 OPFS 根（`rootName` 一并置空串），`put` 写回。然后正常点击「恢复项目」——应用按原生流程打开「项目」，全程无感知。

### 附：该环境的输入通道差异

IAB 里 Playwright 合成点击（`locator.click()`，含 `force: true`）会超时**但无实际效果**。可用通道：

- `tab.cua.click({ x, y })` 坐标级真实输入（坐标先用 `getBoundingClientRect()` 量）
- `evaluate` 里 `element.click()` JS 派发（对 Vue `@click` 有效；足够触发应用逻辑，不产生原生授权所需的用户激活——本方案无需授权所以无碍）

## 验证与清理

**验证落盘**：从页面 evaluate 直接遍历 OPFS（或读 `novel.config.json`）核对文件状态，比 UI 目视更硬。

**清理**（测试遗留物三处，一次 evaluate 可全清）：

1. `recent-projects` 表里的测试记录（注意恢复操作会追加新 projectId 的记录，旧记录不会自动消失——会出现两条同名条目，属预期）；
2. OPFS 根的内容（`removeEntry` 逐项）；
3. 磁盘上的落点（供好奇/核对，不建议手工翻）：`~/Library/Application Support/ZCode/session/Partitions/zcode-embedded-browser/File System/`，文件以数字编号存储，人读无意义。

## 边界与注意事项

- 副本会带上测试产生的改动（重命名、写回、新增预设等）——它们只存在于 OPFS，与真实项目无关；
- 用户自己浏览器的站点存储独立存在，本方案不触碰；
- 涉及「写文件弹确认卡」「权限档位」类验收点时，子智能体在 OPFS 上的写操作与真实目录行为一致（走同一运行层路径闸与确认链），可正常验收；
- debug 日志若应用未落盘（如 `.novel/logs` 不存在），消息序列类验收点降级为单测断言。

## 使用记录

| 日期 | 任务 | 结果 |
| :-- | :-- | :-- |
| 2026-10-20 | 要素管理员升级 W4 真机回归（十验收点） | 全过；验收点 5 按计划降级单测；种子注入 gardener 旧块验证迁移 |
