# 自定义脚本

本目录存放 MXU「自定义脚本」功能使用的 YAML 脚本。**每个脚本一个文件夹**：

```
custom_scripts/
  示例-随机点击/          ← 文件夹名即脚本名称（唯一来源）
    script.yaml           ← 脚本正文
    image_1.png           ← 脚本引用的模板图片，与 script.yaml 放在一起
```

- 在 MXU「设置 → 自定义脚本」中可以录制、编辑、导入、导出、重命名、删除脚本。
- 脚本名称只由文件夹名决定，`script.yaml` 内不写 `name`。在录制器标题栏修改名称并保存会重命名文件夹，任务列表中引用该脚本的任务会自动同步。
- 保存时会清理文件夹中不再被脚本引用的图片。
- 在任务列表「添加任务 → 自定义脚本」分组中添加脚本任务，可设置执行次数（0 表示使用脚本内 `repeat`）。
- 直接修改本目录下的文件后，MXU 会在几秒内自动刷新列表；脚本每次执行时都从磁盘重新读取。
- 旧版平铺布局（`<名称>.yaml` + `<名称>.assets/`）会在 MXU 打开时自动迁移为文件夹布局。
- 录制器生成的步骤统一使用展开（块）写法，并插入到编辑器光标所在位置：光标停在 `then:` / `else:` 行或分支内的步骤上时，新步骤会插入到该分支中。

## 文件结构

```yaml
description: 描述
repeat: 1          # 默认执行次数
retry:             # 整轮失败重试（可选）
  times: 1
  interval: 2s
resolution:        # 录制分辨率，执行时按实际截图分辨率等比缩放坐标
  width: 1280
  height: 720
steps:             # 步骤列表，自上而下执行
  - click:
      x: 640
      y: 360
      r: 30
```

## 失败重试

- **步骤级**：在任意步骤上加同级的 `retry`，该步骤失败（含超时）时按次数重试：

  ```yaml
  - click_text:
      text: 挑战
      timeout: 30s
    retry:
      times: 3       # 失败后最多再试 3 次
      interval: 1s   # 每次重试前等待，默认 1s
  ```

  也可简写为 `retry: 3`。

- **整轮级**：脚本顶层的 `retry`，本轮任一步骤失败时从第一步重新执行本轮。
- 手动停止任务与 `fail` 步骤不会触发重试。

## 时长写法

数字表示毫秒；也可带单位：`300ms`、`1.5s`、`2m`。

## 坐标与区域

| 写法 | 含义 |
|---|---|
| `x` `y` | 固定点 |
| `x` `y` `r` | 圆形区域，执行时在圆内均匀随机取点 |
| `x` `y` `w` `h`（也可写 `width` / `height`） | 矩形区域，执行时在矩形内（偏向中心）随机取点 |

```yaml
- click:
    x: 466
    y: 384
    r: 84
```

## 步骤一览

| 步骤 | 说明 |
|---|---|
| `click` | 点击（圆内 / 矩形内随机） |
| `swipe` | 滑动，`from` / `to` 为区域，`duration` 为时长 |
| `wait` | 固定等待，如 `wait: 1s` |
| `random_wait` | 在 `min` ~ `max` 之间随机等待 |
| `wait_text` / `wait_image` | **条件阻塞**：循环检测，检测到目标后才执行下一步（`text` 支持正则） |
| `wait_text_gone` / `wait_image_gone` | 循环检测，直到目标消失 |
| `click_text` / `click_image` | 循环检测直到目标出现，然后在其范围内随机点击 |
| `if_text` / `if_image` | 条件判断，命中执行 `then`，否则执行 `else` |
| `repeat` | 重复执行 `steps` `times` 次 |
| `until_text` / `until_image` | 循环执行 `steps`，直到目标出现 |
| `run_task` | 调用资源包中已有的 pipeline 节点 |
| `log` | 输出日志 |
| `stop` | 结束脚本（视为成功） |
| `fail` | 结束脚本并标记任务失败 |

识别类步骤的通用参数：

- `roi`：识别区域（`x` `y` `w` `h`），省略为全屏
- `timeout`：最大阻塞时间。阻塞 / 点击 / 等待消失类默认 `30s`，`until_*` 默认 `120s`，`if_*` 默认只识别一次
- `interval`：循环检测间隔，默认 `500ms`
- `threshold`：图片匹配阈值（0~1），默认 `0.8`
- `on_timeout`：超时后的处理，`fail`（默认，任务失败）/ `skip`（跳过该步）/ `stop`（结束脚本）

```yaml
- click_text:
    text: 挑战
    roi:
      x: 1100
      y: 584
      w: 125
      h: 82
    timeout: 30s
- if_image:
    image: image_1.png
    roi:
      x: 434
      y: 353
      w: 378
      h: 324
    threshold: 0.8
    timeout: 1s
    then:
      - click:
          x: 466
          y: 384
          r: 84
    else:
      - stop: 未找到
```

旧的简写（`[x, y, r]`、`{ text: …, roi: [...] }`、`random_wait: 1s 3s`）仍可正常解析。

## 导入导出

导出文件为单个 `.mxuscript.yaml`：脚本正文之后跟一行 `--- # mxu-script-assets`，再跟 `文件名: base64` 形式的图片资源。导入时自动还原为脚本文件与 `.assets` 目录。
