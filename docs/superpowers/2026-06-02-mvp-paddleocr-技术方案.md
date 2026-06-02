# PaddleOCR API 直连技术方案

日期：2026-06-02
状态：已确认，可更新实施计划

---

## 1. 决策背景

MVP 原计划所有云端能力（OCR、图片增强、去手写、AI 解析）使用 mock 服务。现决定 OCR 模块直接对接真实 PaddleOCR API（PaddleOCR-VL-1.6），其余云端能力保持 mock。

**理由：** OCR 是错题本的核心数据入口——题号识别、题型判断、文字摘要都依赖 OCR 结果。真实 OCR 数据能暴露 mock 阶段发现不了的问题：公式识别准确率、中英文混排效果、手写体干扰下的识别质量等。

---

## 2. API 概况

| 项目 | 值 |
|------|-----|
| 接口地址 | `https://paddleocr.aistudio-app.com/api/v2/ocr/jobs` |
| 模型 | `PaddleOCR-VL-1.6` |
| 认证方式 | Bearer Token |
| 调用模式 | 异步：提交任务 → 轮询状态 → 下载结果 |
| 输入 | 本地文件（multipart）或公网 URL |
| 输出 | JSONL，每页含 Markdown 文本 + 图片 |
| 轮询间隔 | 建议 5 秒 |
| 可选能力 | 文档方向分类、文档展平、图表识别 |

---

## 3. 架构决策

### 3.1 Token 安全

Token 不能写入 Flutter 客户端代码。打包进 IPA 即视为泄露。

**方案：** MVP 阶段使用 `.env` 文件管理 Token。项目根目录创建 `.env`，通过 `flutter_dotenv` 在运行时加载。`.env` 文件加入 `.gitignore`。

```bash
# .env（不进入版本控制）
PADDLEOCR_TOKEN=32879366a89bcdd3be5e35f5abb23f4b4d387539
PADDLEOCR_BASE_URL=https://paddleocr.aistudio-app.com/api/v2/ocr/jobs
PADDLEOCR_MODEL=PaddleOCR-VL-1.6
```

```dart
// lib/src/core/api/paddleocr_config.dart
import 'package:flutter_dotenv/flutter_dotenv.dart';

class PaddleOcrConfig {
  static String get token => dotenv.env['PADDLEOCR_TOKEN'] ?? '';
  static String get baseUrl => dotenv.env['PADDLEOCR_BASE_URL'] ?? '';
  static String get model => dotenv.env['PADDLEOCR_MODEL'] ?? 'PaddleOCR-VL-1.6';
}
```

```yaml
# pubspec.yaml 新增依赖
dependencies:
  flutter_dotenv: ^5.1.0

# assets 声明
flutter:
  assets:
    - .env
```

长期方案：通过一个薄后端代理转发 OCR 请求，Token 只存在于服务端。MVP 不做代理，直接客户端调 API，接受 demo 产品初期的短期安全风险。

### 3.2 直连 vs 代理

| 方案 | 优点 | 缺点 |
|------|------|------|
| Flutter 直连 API | 无中间层，开发快 | Token 在客户端，网络问题无重试保障 |
| 薄后端代理 | Token 安全，可加队列和重试 | 需要额外部署，MVP 范围外 |

**决策：** MVP 阶段 Flutter 直连。后续版本迁移到代理。

### 3.3 结果映射

API 返回的 JSONL 结构中的关键字段 → Drift `OcrResult` 表：

| API 字段 | OcrResult 列 | 说明 |
|----------|-------------|------|
| `result.layoutParsingResults[].markdown.text` | `rawText` | 题目文字内容 |
| `result.layoutParsingResults[].markdown.images` | 不单独存，路径写入 `blocksJson` | 题图中的图片提取 |
| 整体 API 响应 | `blocksJson` | 完整 JSONL 原文存一份，方便后续查看和调试 |
| 从 markdown.text 中提取 | `detectedQuestionNumber` | 正则匹配"第X题"或数字编号 |
| 从题目文本推断 | `detectedQuestionType` | 可选：有选项 → choice，有填空线 → fill_blank |
| 固定值 | `engine` | `PaddleOCR-VL-1.6` |

---

## 4. Flutter 端实现设计

### 4.1 新增/修改文件

```
lib/src/core/api/
├── cloud_processing_client.dart        # [修改] 新增 ocrQuestion 真实实现接口
├── mock_cloud_processing_client.dart   # [修改] OCR 部分替换为真实调用
├── paddleocr_client.dart              # [新增] PaddleOCR API 客户端
└── paddleocr_config.dart              # [新增] Token 和 URL 配置

test/core/api/
└── paddleocr_client_test.dart         # [新增] PaddleOCR 客户端测试
```

### 4.2 PaddleOCR 客户端核心类

```dart
// paddleocr_client.dart — 核心方法签名

class PaddleOcrClient {
  final Dio _dio;
  final String _token;

  /// 提交 OCR 任务，返回 jobId
  /// [imagePath] 本地图片路径或公网 URL
  /// [mode] local 或 url
  Future<String> submitJob(String imagePath, {OcrMode mode = OcrMode.local});

  /// 轮询任务状态，完成后返回结果下载 URL
  /// 超时 120 秒，每 5 秒轮询一次
  Future<OcrJobResult> pollJob(String jobId);

  /// 下载并解析 JSONL 结果
  Future<OcrParseResult> fetchAndParseResult(String jsonlUrl);

  /// 便捷方法：submit → poll → fetch
  Future<OcrParseResult> processImage(String imagePath);
}

enum OcrMode { local, url }

class OcrJobResult {
  final String jsonlUrl;
  final int extractedPages;
  final DateTime startTime;
  final DateTime endTime;
}

class OcrParseResult {
  final String rawText;        // 合并后的 Markdown 文本
  final String blocksJson;     // 完整 JSONL 原始数据
  final double confidence;     // 由 API 返回或设为 1.0
  final String? detectedQuestionNumber;
  final String? detectedQuestionType;
}
```

### 4.3 轮询与超时策略

```
提交 job
  │
  ├─ 每 5s 轮询 GET /jobs/{jobId}
  │   ├─ pending → 继续等待
  │   ├─ running → 继续等待，打印进度
  │   ├─ done → 获取 jsonlUrl，进入下载
  │   └─ failed → 抛出异常，记录 errorMsg
  │
  └─ 超时 120s → 抛出超时异常，job 状态保留
```

- 前台轮询：直接轮询，UI 显示处理进度。
- App 退到后台：Flutter 端的轮询可能被系统挂起。MVP 不处理这个场景——用户切回 App 后重新查询 job 状态。
- 轮询失败的降级：网络错误重试 3 次，间隔递增（2s/4s/8s），全部失败后标记 OCR 状态为 `failed`，不阻塞其他流程。

### 4.4 与现有 mock 架构的衔接

修改 `mock_cloud_processing_client.dart`：
- keep：`enhanceQuestion`、`cleanQuestion`、`generateSolution` 保持 mock
- replace：`ocrQuestion` 委托给 `PaddleOcrClient.processImage()`

调用方（`QuestionRepository` / Riverpod notifier）不感知差异，接口不变。

### 4.5 图片上传格式

本地文件模式使用 `dio` 的 `FormData` + `MultipartFile`：

```dart
final formData = FormData.fromMap({
  'model': 'PaddleOCR-VL-1.6',
  'optionalPayload': jsonEncode({
    'useDocOrientationClassify': false,
    'useDocUnwarping': false,
    'useChartRecognition': false,
  }),
  'file': await MultipartFile.fromFile(imagePath),
});
```

---

## 5. 对实施计划的影响

### 5.1 受影响的任务

| 任务 | 变更 |
|------|------|
| Task 1（依赖） | pubspec.yaml 无额外依赖，`dio` 已包含 |
| Task 5（Mock 客户端） | OCR 部分从纯 mock 变为真实 API 调用，其余保持 mock。`paddleocr_client.dart` 作为真实实现，mock client 的 OCR 方法委托给它 |
| Task 8（异步管线） | 入库后的 OCR 步骤不再是瞬时 mock，需要真实的异步等待（5-30 秒）。UI 需要反映真实的 `processing`/`running` 状态 |
| Task 14（E2E 验证） | OCR 需要真实网络环境，增加网络不可用时的降级验证 |

其余任务不受影响。

### 5.2 新增依赖

无。`dio` 已在 Task 1 的依赖列表中。

### 5.3 新增测试

- `paddleocr_client_test.dart`：测试提交/轮询/解析流程。测试需要网络且消耗 API 配额，标记为 integration test，默认不跑。CI 中跳过。
- 单元测试：测试 JSONL 解析逻辑和题号提取正则，不依赖网络。

---

## 6. 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| Token 泄露 | 安全，API 被滥用 | MVP 接受短期风险；`.env` 不进 Git；后续迁移到后端代理 |
| API 调用频率限制 | OCR 不可用 | 确认配额；单用户操作频率天然低（几分钟一道题） |
| 网络不稳定导致 OCR 失败 | 用户体验差 | 失败后标记 `failed`，用户可手动输入题号和题型；重试 3 次 |
| 多题模式下每道题都调用 OCR | 耗时和配额消耗 | 多题模式下可以批量提交（API 一次可处理整张试卷），然后按题框拆分结果 |
| PaddleOCR-VL 对数学公式的识别效果 | 产品核心功能 | 需要在 spike 阶段用真实试卷验证，如效果差需换模型或补充 prompt |

---

## 7. 确认结果

1. **Token 配额和限制：** 待确认。当前 Token 的每日调用上限和 IP 白名单策略未知，需在实际调佣中观察。
2. **多题模式的 OCR 策略：** 一次提交整张试卷图，API 返回 layoutParsing 后按题框坐标拆分文本到各错题。单题模式下提交裁剪后的单题图。
3. **Token 注入方式：** `.env` 文件 + `flutter_dotenv`。demo 产品初期可接受的策略。
4. **后端代理：** MVP 直连，后续版本迁移。
