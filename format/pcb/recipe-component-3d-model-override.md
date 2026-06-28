# 操作配方：单独覆盖某个器件的 3D 模型（不动 footprint / 封装 / 网络）

## 场景

只把**指定单个器件**（例如 SW3、SW4）的 3D 模型换成另一个 LCSC/库模型的 3D 模型，
**不修改** footprint、焊盘、网络、位号、坐标、旋转、层等任何其它属性。
其它使用同一 footprint 的器件（例如 SW1、SW2）不受影响。

典型用途：BOM 贴片元件型号与默认 footprint 自带 3D 模型不一致（外形/高度不同），
需要在 3D 预览/装配图里显示真实外形。

## 为什么不能直接用 API

`IPCB_PrimitiveComponent` 只有 `getState_Model3D()`，**没有 `setState_Model3D()`**；
`pcb_PrimitiveComponent.modify()` 的 property 参数也不含 `model3D` 字段。
因此官方 API 层**无法**单独改 3D 模型，只能走 UI 手动改 或 源码层改。

> 副作用：在 3D 预览视图（`documentType === 15` 即 `PCB_3D_PREVIEW`）下，
> `sys_FileManager.getDocumentSource()` 会抛 `"Cannot destructure property 'message'..."`。
> 操作源码前必须先用 `dmt_EditorControl.openDocument(pcbUuid)` 切回 PCB 2D 编辑视图（`documentType === 3`）。

## 关键结论：覆盖 3D 模型 = 给 COMPONENT 实例加 3 个 attrs + 1 个子 ATTR

当某个器件**没有**覆盖（使用 footprint 默认 3D 模型）时，COMPONENT 的 `attrs` 里**没有**任何 `3D Model*` 键，
3D 模型从 footprint 继承。

一旦在 UI 里"单独指定"了 3D 模型，COMPONENT 实例的 `attrs` 里会**新增**三个键，并新增一条子 ATTR。

## 实测 diff（同板 SW3 已改 / SW4 未改，二者 footprint 相同）

### SW3（已单独指定 3D 模型）

```json
{"type":"COMPONENT","ticket":735,"id":"c77344f107be8ef8"}||{
  "partitionId":"",
  "groupId":0,
  "layerId":1,
  "x":500.9959,
  "y":-3546.3328,
  "angle":0,
  "attrs":{
    "3D Model":"2d8bb60bd5304fe6a91832393c329795|0819f05c4eef4c71ace90d822a990e87",
    "3D Model Title":"KEY-SMD_4P-L12.0-W12.0-P5.00-LS15.1_K2-1103ST-A4SW-04",
    "3D Model Transform":"606.2979849815368,472.44394234657284,287.40100750923153,0,0,0,0,0,0",
    "Unique ID":"UNIQUESW3",
    "Reuse Block":"",
    "Group ID":"",
    "Channel ID":"$4I445",
    "Function":"用户按键1 / KEY1 / ESP32-C6 GPIO18",
    "Codex_Note":"..."
  },
  "locked":true,
  "zIndex":-1
}|
```

外加一条**子 ATTR**（`parentId` 指向该 COMPONENT，默认隐藏）：

```json
{"type":"ATTR","ticket":739,"id":"c77344f107be8ef89507fdcdcd9cb779"}||{
  "partitionId":"",
  "groupID":0,
  "parentId":"c77344f107be8ef8",
  "layerId":3,
  "x":146.6609,
  "y":-3857.5188,
  "key":"3D Model Title",
  "value":"KEY-SMD_4P-L12.0-W12.0-P5.00-LS15.1_K2-1103ST-A4SW-04",
  "keyVisible":false,
  "valueVisible":false,
  "fontFamily":"default",
  "fontSize":45,
  "strokeWidth":6,
  "bold":0,
  "italic":0,
  "origin":"LEFT_BOTTOM",
  "angle":0,
  "reverse":false,
  "expansion":0,
  "mirror":false,
  "locked":false,
  "zIndex":-1,
  "specialColor":"#000000"
}|
```

### SW4（未改，继承 footprint 默认 3D 模型）

```json
{"type":"COMPONENT","ticket":744,"id":"d8f9e0b41841af85"}||{
  "partitionId":"",
  "groupId":0,
  "layerId":1,
  "x":1422.2557,
  "y":-3546.3328,
  "angle":0,
  "attrs":{
    "Unique ID":"UNIQUESW4",
    "Reuse Block":"",
    "Group ID":"",
    "Channel ID":"$4I451",
    "Function":"用户按键2 / KEY2 / ESP32-C6 GPIO19",
    "Codex_Note":"..."
  },
  "locked":true,
  "zIndex":-1
}|
```

**没有** `3D Model` / `3D Model Title` / `3D Model Transform` 键，**也没有** `key="3D Model Title"` 的子 ATTR。

### 完全一致的属性（验证 footprint / 封装 / 网络未变）

两者都有完全相同的：
- `layerId`、`angle`、`locked`、`zIndex`、`partitionId`、`groupId`
- 子 ATTR `key="Footprint"` → `value="f6239897efda0db0"`（同一 footprint）
- 子 ATTR `key="Device"` → `value="0c417d60f8e07626"`（同一 device）
- 四条 `PAD_NET`（焊盘网络映射，KEYx/GND 完全对应）

结论：单独覆盖 3D 模型**只动了** COMPONENT 的 `attrs` 三键 + 一条子 ATTR，其它什么都没动。

## 字段说明

| 位置 | 键 | 格式 | 说明 |
|------|-----|------|------|
| COMPONENT `attrs` | `3D Model` | `"<modelUuid>\|<libraryUuid>"` | 必填，模型 UUID 竖线分隔库 UUID |
| COMPONENT `attrs` | `3D Model Title` | 字符串 | 模型的人类可读名称 |
| COMPONENT `attrs` | `3D Model Transform` | 9 个逗号分隔数字 | 见下表 |
| 子 ATTR | `key="3D Model Title"` | 字符串 | 与 attrs 内同值；UI 属性面板缓存，默认隐藏 |

### `3D Model Transform` 9 个参数顺序

```
sizeX, sizeY, sizeZ, rotZ, rotX, rotY, offX, offY, offZ
```

| 序号 | 名称 | 含义 |
|------|------|------|
| 1 | sizeX | X 轴尺寸（缩放基准） |
| 2 | sizeY | Y 轴尺寸 |
| 3 | sizeZ | Z 轴尺寸，**为 0 表示自动适应高度** |
| 4 | rotZ | 绕 Z 轴旋转角度 |
| 5 | rotX | 绕 X 轴旋转角度 |
| 6 | rotY | 绕 Y 轴旋转角度 |
| 7 | offX | X 轴偏移 |
| 8 | offY | Y 轴偏移 |
| 9 | offZ | Z 轴偏移 |

变换矩阵算法见 [component.md](./component.md) 的 "3D Model Transform 的特殊说明"。
单位为 mil（1mil = 0.0254mm），例如 `590.55` ≈ 15.0mm。

## 如何取得目标模型的 uuid / libraryUuid / Title / Transform

不要凭空构造，从 LCSC 元件库搜出来最可靠：

```javascript
// 以 C2733614 为例
const r = await eda.lib_Device.search("C2733614");
// r[0].model3DUuid          -> "fe3a276f50fb4f7391469018b03b16d8"
// r[0].libraryUuid          -> "0819f05c4eef4c71ace90d822a990e87"  (LCSC 系统库)
// r[0].model3DName          -> "KEY-SMD_4P-L12.0-W12.0-H7.3-LS15.0-P5.0"
// r[0].otherProperty["3D Model Transform"] -> "590.55,472.44,0,..."
```

填入 COMPONENT `attrs["3D Model"]` 时拼成 `"<model3DUuid>|<libraryUuid>"`。
`3D Model Title` 用 `model3DName`，`3D Model Transform` 用元件自带的 transform 字符串。

## 操作步骤（源码层，可批量化）

1. **切到 PCB 2D 编辑视图**（必须在 2D，不能在 3D 预览）：
   ```javascript
   await eda.dmt_EditorControl.openDocument(pcbUuid);
   ```
2. **备份**：先 `getDocumentSource()` 把整份源码保存到本地文件，便于回滚。
3. **读源码**：`const src = await eda.sys_FileManager.getDocumentSource();`
4. **定位目标 COMPONENT 行**：按 `primitiveId`（如 `c77344f107be8ef8`）grep。
5. **改 COMPONENT 的 `attrs`**：在 `attrs` 对象里新增/替换 `3D Model` / `3D Model Title` / `3D Model Transform` 三个键。
6. **新增子 ATTR**：在 COMPONENT 行之后插入一条 `key="3D Model Title"` 的 ATTR（parentId = 该 COMPONENT id）。
7. **写回**：`await eda.sys_FileManager.setDocumentSource(newSrc);`
8. **重载**：`openDocument(pcbUuid)` 重新打开，让画布刷新。
9. **验证**：见下。

> 改源码必须保留 `||...|` 双竖线 + 末尾竖线的包裹结构，且外层 JSON 的 `type/id/ticket` 三键不能丢。
> `setDocumentSource` 是整文档覆盖，任何手抖都会损坏文件 —— **必须先备份**。

## 验证

```javascript
const c = await eda.pcb_PrimitiveComponent.get([targetPrimitiveId]);
const a = c[0].toAsync();
const m = a.getState_Model3D();
// m 应为 { libraryUuid, uuid:"<modelUuid>|<libraryUuid>" }
```

或在 UI 里选中器件看属性面板的"3D 模型"字段，以及 3D 预览里的外形。

## 撤销覆盖（恢复 footprint 默认 3D 模型）

反向操作：从 COMPONENT `attrs` 删除三个 `3D Model*` 键，并删除对应的子 ATTR 行。
然后 `setDocumentSource` 写回 + 重载。

## 风险与注意

- `setDocumentSource` 是**全量覆盖**，改之前**务必备份**整份源码。
- 改完必须在 2D 编辑视图写回，3D 预览视图下 `getDocumentSource`/`setDocumentSource` 行为异常。
- 不同外形（圆按钮 vs 矩形按钮、高度 6mm vs 7.3mm）的模型替换后，**机械结构（外壳开孔、限高）要重新核对**。
- 同一 footprint 的其它器件**不会**自动跟随（这正是"单独覆盖"的目的）。
- 子 ATTR 的 `id` 用一个文档内未使用的随机 hex 即可；ticket 用一个递增未占用的整数。

## 相关 API

- `eda.sys_FileManager.getDocumentSource()` / `setDocumentSource(src)`
- `eda.dmt_EditorControl.openDocument(docUuid)`
- `eda.lib_Device.search(keyword)` —— 取目标模型的 uuid/title/transform
- `eda.pcb_PrimitiveComponent.get([id])` → `.toAsync().getState_Model3D()` —— 验证
