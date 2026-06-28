# 操作配方：机械孔边缘加镀锡层（与周边 GND 隔断）

## 适用场景

PCB 上有 NPTH（非金属化）机械螺丝孔（例如 M3 过孔 3.4mm），需要在孔边缘加一圈**露铜喷锡**镀层（增加耐磨、提供接地接触面），但**不能**与周边大面积 GND 铺铜连通。

## 为什么不能走官方 API

`pcb_PrimitivePad.get([padId])` 在独立（非 component 子焊盘）的机械孔 pad 上会抛：

```
Cannot read properties of null (reading 'length')
```

无法 `toAsync().setState_Pad()`。这是 EasyEDA API 的缺陷，所以只能走**源码层**。

## 原理：扩大 NPTH 焊盘铜环 + 阻焊开窗

机械孔在源码里是一个 `type:"PAD"` 图元。原本铜径 = 孔径（无铜环）。只要把铜径扩大到比孔大，并设置阻焊扩展为 0（露铜），就形成一圈镀锡铜环。

由于 NPTH 是 `plated:false`、无网络（`netName:""`），GND pour 会自动按安全间距避让这圈铜 → **天然与 GND 隔断**，无需额外画 keepout。

## PAD 源码字段表

| 字段（PAD body）| 改不改 | 值 | 说明 |
|------|:---:|------|------|
| `hole.width` / `hole.height` | ❌ 不动 | 133.8583 mil (3.4mm) | M3 过孔 |
| `defaultPad.width` / `defaultPad.height` | ✅ **改** | 255.9055 mil (6.5mm) | 扩大铜环 |
| `plated` | ❌ 不动 | false | NPTH |
| `netName` | ❌ 不动 | "" | 无网络 |
| `topSolderExpansion` | ✅ **改** | 0 | 露铜 = 整个铜径 |
| `bottomSolderExpansion` | ✅ **改** | 0 | 双面露铜 |
| `connectMode` | ✅ 统一 | null | 不参与连接规则 |
| `centerX` / `centerY` | ❌ 不动 | 原坐标 | 位置不变 |
| `layerId` | ❌ 不动 | 12 (MULTI) | 多层 |
| `padAngle` | ❌ 不动 | 0 | 不旋转 |

### 铜环尺寸计算

```
铜环宽度 = (padWidth - holeWidth) / 2
         = (255.9 - 133.9) / 2
         ≈ 61 mil = 1.55mm（每边）
```

### 单位换算

```
1mm = 39.37 mil
3.4mm ≈ 133.86 mil
6.5mm ≈ 255.9 mil
1.55mm ≈ 61 mil
```

### 阻焊字段含义

| 值 | 含义 |
|------|------|
| `null` | 按工程默认（通常缩进 4mil） |
| `0` | 阻焊开窗 = 铜径尺寸，即整圈露铜 |
| 正数 | 阻焊从铜边再扩大 N mil |

## 操作步骤

1. **切到 PCB 2D 编辑视图**（3D 预览视图下源码 API 失效）：

   ```javascript
   await eda.dmt_EditorControl.openDocument(pcbUuid);
   ```

2. **备份**整份源码到本地文件。

3. **取当前源码**：

   ```javascript
   const src = await eda.sys_FileManager.getDocumentSource();
   ```

4. **手术式修改**：按 `primitiveId` 定位每个机械孔的 `"type":"PAD"` 行，仅改三个字段：
   - `defaultPad.width = defaultPad.height = 255.9055`
   - `topSolderExpansion = 0`
   - `bottomSolderExpansion = 0`
   - 其余字段（hole / plated / netName / 坐标 / 旋转 / layer）**一字不改**。

5. **写回**：

   ```javascript
   await eda.sys_FileManager.setDocumentSource(newSrc);  // 返回 true 才算成功
   ```

6. **重载**：`openDocument(pcbUuid)` 重新打开。

7. **验证**：

   ```javascript
   const p = await eda.pcb_PrimitivePad.get([mhPadId]);
   const a = p[0].toAsync();
   // pad: ["ELLIPSE", 255.9, 255.9]
   // hole: ["ROUND", 133.8583]（未变）
   // plated: false（未变）
   // solder.topSolderMask: 0
   ```

8. **保存**（关键，容易忘）：

   > `setDocumentSource` 只写入画布内存，不等于保存到工程。

   ```javascript
   await eda.pcb_Document.save(pcbUuid);
   ```

9. **重新铺铜**：UI 里 `工具 → 更新所有铺铜`，让 GND pour 按新铜环避让。不重铺铜会导致 DRC 报 `Copper Region(Filled) to TH Pad` 间距错误。

10. **跑 DRC**：确认 GND 与铜环之间无短路 / 间距违规，且 MH 铜环附近无信号走线 / 过孔冲突。

## DRC 检查

跑 `eda.pcb_Drc.check(true, true, true)`（第三个参数 `true` 返回明细数组）。

扩大铜环后预期的 DRC 错误类型：

| DRC 类型 | 原因 | 修复方式 |
|----------|------|---------|
| `Copper Region(Filled) to TH Pad` | GND 铺铜缓存未更新 | 重新铺铜 |
| `TH Pad to Track` | 信号走线物理重叠新铜环 | 绕线 |
| `TH Pad to Via` | 过孔物理重叠新铜环 | 移过孔或缩铜环 |
| `Hole to TH Pad` | 过孔的孔重叠新铜环 | 移过孔或缩铜环 |

> **重要**：扩大铜环前应先检查目标位置附近是否有信号走线 / 过孔。密集布线区域的机械孔不适合做大铜环。

## 撤销

反向操作：把 `defaultPad.width/height` 改回孔径值（133.86），阻焊字段改回 null，重铺铜即可。

## 注意事项

- 铜环宽度建议 ≥ 0.8mm（约 31.5 mil），低于此喷锡挂不住。
- 铜环外径要小于螺丝垫片直径，避免垫片压到铜环导致接地短路（除非本就希望接地）。
- 改完后**必须重新铺铜**，否则 DRC 报间距错误且制造文件里 GND 会覆盖铜环。
- `topSolderExpansion = 0` 不是"无阻焊"，而是"阻焊开窗 = 铜径尺寸"，即整圈露铜。
- 密集布线区域的机械孔（如靠近主控 IC / 编码器）做大铜环前，**必须先检查周边走线/过孔间距**，否则会产生大量 DRC 冲突。

## 实测案例（Sonos 控制器 屏幕排线版本 主板）

MH1-MH5 五个 M3 机械孔，全部从 3.4mm 无铜环改为 6.5mm 铜环 + 双面露铜：

- 源码 diff：精确 5 行变更（每行一个 MH pad），无任何意外改动。
- 写回 + 重载 + `pcb_Document.save()` 全部成功。
- DRC 结果：29 个 MH 相关错误
  - 15 个 `Copper Region(Filled) to TH Pad` → GND 铺铜未重铺（重铺即修复）
  - 14 个集中在 MH4（位于 ESP32 模组与旋转编码器之间，信号走线密集），包括 `TH Pad to Track` / `TH Pad to Via` / `Hole to TH Pad` → 需绕线或缩小 MH4 铜环
- **教训**：扩大铜环前必须检查周边走线密度，密集区域不适合大铜环。

## 相关 API

- `eda.sys_FileManager.getDocumentSource()` / `setDocumentSource(src)`
- `eda.dmt_EditorControl.openDocument(docUuid)`
- `eda.pcb_PrimitivePad.get([id])` → `.toAsync().getState_Pad()` —— 验证（注意独立 pad 上可能崩）
- `eda.pcb_Document.save(pcbUuid)` —— 保存到工程
- `eda.pcb_Drc.check(true, true, true)` —— DRC 明细
