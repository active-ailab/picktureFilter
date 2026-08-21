# picktureFilter

## 项目简介

当前仓库实现的是一个单页运动记录解析工具，而不是仓库名称及 Bitable 登记所描述的图片过滤器。页面在浏览器本地读取二进制 `detail.data`，按 TLV 结构识别运动数据 TAG，将记录分类为 JSON 后打包下载，并提供 TAG 统计和心率曲线查看。

为避免误导，本文档按当前 `main` 分支的真实 `index.html` 行为编写；图片过滤用途与现有代码不一致的问题见“注意事项”。

## 核心功能

- 在浏览器本地读取 `detail.data` 二进制文件。
- 识别单字节和双字节 TAG，并按 TAG 名称聚合记录。
- 对时间戳、事件、心率、血氧、海拔、步频、跑步功率、HYROX 分段等部分已知 TAG 进行结构化解析。
- 未实现专用解析器或长度不满足要求的数据以 `raw_hex` 保留。
- 统计总记录数、TAG 类型数，以及心率、秒级数据、HYROX 分段等记录数量。
- 将每种 TAG 写入独立 JSON，并生成包含来源、总数、TAG 汇总和文件映射的 `summary.json`。
- 使用页面内置的无压缩 ZIP 生成逻辑，将全部 JSON 一次下载到本地。
- 使用 Chart.js 绘制 `RT_HEARTRATE` 心率曲线，并显示最高、最低、平均心率、运动时长和采样数。

## 适用场景

- 开发或测试人员快速检查设备导出的 `detail.data` 内容。
- 按 TAG 拆分大体量运动记录，便于检索、对比与问题定位。
- 验证心率采样数量和趋势，以及部分 HYROX、海拔、功率等字段。
- 将未知或暂未适配的数据保留为十六进制，供后续协议分析。

当前实现不适用于 common、Sports 等仓库的图片浏览或过滤。

## 目录结构

```text
picktureFilter/
├── index.html                         # 页面、TLV 解析、ZIP 导出与图表逻辑
└── .github/
    └── workflows/
        └── feishu-notify.yml          # push 或手动触发的飞书通知工作流
```

## 使用方法

### 启动页面

可直接使用现代浏览器打开 `index.html`。如需通过本地 HTTP 服务访问，在仓库目录运行：

```bash
python3 -m http.server 8000
```

随后访问 `http://localhost:8000/`。

### 解析数据

1. 在“detail.data（必选）”区域选择待解析文件。
2. 页面中的 `sport_record_save_detail.h` 选择项为可选项，但当前代码明确标注“暂不使用”，不会参与解析。
3. 点击“解析并下载 ZIP”。
4. 页面完成 TLV 扫描后，会自动下载 `<时间戳>_<原文件名>.zip`。
5. 在页面统计区查看各 TAG 数量；点击 `RT_HEARTRATE` 卡片可查看心率曲线。

### 输出结构

ZIP 内包含以时间戳和源文件名命名的目录：

```text
<timestamp>_<source>/
├── summary.json
├── TIMESTAMP.json
├── RT_HEARTRATE.json
├── HYROX_LAP.json
└── ...                               # 仅生成输入中实际出现的 TAG 文件
```

每条记录包含源文件偏移、数值 TAG、TAG 名称、数据长度及解析后的 `data`。

## 技术说明

- 应用主体为原生 HTML/CSS/JavaScript，无构建步骤和包管理配置。
- TLV 数值按小端序读取。
- 页面通过 CDN 加载 `Chart.js 4.4.1`；TLV 解析和 ZIP 生成逻辑位于本地 `index.html`。
- ZIP 使用 stored（不压缩）方式生成，因此输出文件可能接近各 JSON 文件大小总和。
- 心率时间轴通过检测 `relative_time_ms` 回绕并按 32768 ms 周期补偿。

## 注意事项

- **用途不一致：** Bitable 登记用途是过滤 common、Sports 等仓库中的大量图片，但当前代码没有扫描图片、预览图片或按条件过滤图片的实现；仓库实际页面标题和逻辑均为运动记录解析。
- `sport_record_save_detail.h` 当前只显示所选文件信息，不参与协议解析。
- 仅部分 TAG 有专用字段解析；其他 TAG 会输出 `raw_hex`，不能视为完整协议解码。
- TLV 扫描遇到越界记录时会停止；代码当前不会生成详细的损坏位置诊断。
- 心率图依赖外部 CDN。离线或 CDN 不可访问时，数据解析与 ZIP 导出仍由本地代码完成，但图表库可能无法加载。
- 所有业务文件由浏览器本地读取；代码没有上传接口，但使用敏感数据前仍应确认浏览器和运行环境可信。
- 仓库没有自动化测试、构建脚本或输入样例，修改协议解析后需使用经过确认的脱敏样本人工验证。
- GitHub Actions 通知工作流依赖 `FEISHU_APP_ID`、`FEISHU_APP_SECRET` 及外部复用工作流，与解析功能无关。
