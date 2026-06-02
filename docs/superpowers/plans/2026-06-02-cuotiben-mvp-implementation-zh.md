# 错题本 MVP 实施计划

> **面向 AI 执行者：** 必须使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实施。步骤使用 checkbox（`- [ ]`）语法追踪。

**目标：** 构建首个 Flutter iOS MVP，一个本地优先的错题本，聚焦初高中数学和物理，支持拍照导入、错题选择、本地题库、真实 PaddleOCR、mock 图片增强/清理/AI 解析、橡皮擦修复和 PDF 导出。

**架构：** 在仓库根目录实现 Flutter 应用，采用 feature-first 目录结构。图片以文件形式存储，仅在 Drift/SQLite 中保存必要的元数据。OCR 使用真实 PaddleOCR-VL API（直连，Token 通过 `.env` 管理）。图片增强、去手写清理和 AI 解析以类型化 API 接口加 mock 服务的方式呈现，确保移动端完整流程可在接入真实后端之前完成开发和测试。

**技术栈：** Flutter、Dart、Riverpod、go_router、Drift/SQLite、path_provider、image_picker、camera、dio、pdf、printing、flutter_markdown、flutter_dotenv、flutter_test。

---

## 范围

本计划实现 Flutter 客户端 MVP。OCR 使用真实 PaddleOCR-VL API 直连（详见 `docs/superpowers/2026-06-02-mvp-paddleocr-技术方案.md`）。图片增强、去手写清理和豆包 AI 解析仍为 mock 服务，以类型化客户端契约表示。这些服务的真实后端有意延后。

来自已确认 spec 的 MVP 约束：

- iOS 优先 MVP。Android 延后至 iOS 流程验证通过之后。
- 目标用户：初高中理科生，以及家长代操作场景。
- 无账号系统，无云端错题库。
- 仅 4 张本地表：`MistakeQuestion`、`OcrResult`、`AiSolution`、`PrintSet`。
- 试卷原图仅作为本地文件路径，不单独建表。
- 仅橡皮擦修复。不实现完整图片编辑器、笔刷系统、矢量/线图覆盖层，错题选择后不提供裁剪/旋转编辑。
- PDF 作答区基于题型：选择题和填空题不额外留空，解答题留作答区。

## 风险控制

橡皮擦画布和大图处理是最高实现风险，不可留到后期。

- 在数据库/路由搭建完成后立即验证橡皮擦绘制、图片降采样和合成保存的内存表现。
- 可打印图片以文件形式存储，禁止写入数据库 blob。
- 导入/保存时异步生成小缩略图，列表视图只用缩略图。
- 原始图片路径和可打印图片路径与缩略图路径分离。
- 如果目标设备上自由橡皮擦不够流畅，在构建更多 UI 之前降级为白色矩形遮盖修复。
- PaddleOCR 使用真实 API。OCR 客户端提交图片、轮询 job 状态、下载 JSONL 结果，将解析后的文本和元数据写入本地数据库。Token 从 `.env` 通过 `flutter_dotenv` 加载，不进入版本控制。
- 云端 mock 服务（增强、清理、AI）必须模拟真实 API 形态：任务返回 URL，客户端下载或将 URL 输出复制到本地文件，仅将本地路径写入数据库。
- 端到端验证仅针对 iOS。缺失 iOS 模拟器运行时、CocoaPods、签名或 iPhone 设备访问属于环境阻塞项。缺失 Android SDK 不是 MVP 阻塞项。

## 文件结构

创建或修改以下路径：

- `pubspec.yaml`：依赖和资源。
- `lib/main.dart`：应用启动入口。
- `lib/src/app/cuotiben_app.dart`：`ProviderScope`、路由、主题。
- `lib/src/app/router.dart`：go_router 路由定义。
- `lib/src/core/theme/app_theme.dart`：克制的 app 主题。
- `lib/src/core/models/enums.dart`：学科、年级、题型、处理状态。
- `lib/src/core/files/app_file_store.dart`：app 文档目录和图片复制辅助。
- `lib/src/core/files/image_derivatives.dart`：缩略图和打印图片衍生策略。
- `lib/src/core/db/app_database.dart`：Drift 数据库和四表定义。
- `lib/src/core/db/app_database_provider.dart`：Riverpod 数据库 provider。
- `.env`：PaddleOCR Token 和 URL（gitignored，由 flutter_dotenv 加载）。
- `lib/src/core/api/cloud_processing_client.dart`：API 接口和 job/result 模型。
- `lib/src/core/api/mock_cloud_processing_client.dart`：确定性 mock 增强/清理/AI 服务。OCR 委托给真实 PaddleOCR 客户端。
- `lib/src/core/api/paddleocr_client.dart`：真实 PaddleOCR-VL API 客户端（提交、轮询、下载、解析）。
- `lib/src/core/api/paddleocr_config.dart`：从 `.env` 加载 PaddleOCR Token 和 URL。
- `lib/src/features/capture/capture_home_screen.dart`：模式选择和图片导入入口。
- `lib/src/features/capture/question_selection_screen.dart`：手动/mock 题框选择和保存到题库。
- `lib/src/features/library/question_library_screen.dart`：可筛选的本地错题列表。
- `lib/src/features/repair/question_detail_screen.dart`：图片、元数据、AI 答案、跳转橡皮擦修复。
- `lib/src/features/repair/eraser_repair_screen.dart`：仅橡皮擦修复画布。
- `lib/src/features/print/print_builder_screen.dart`：选择题目并导出 PDF。
- `lib/src/features/settings/settings_screen.dart`：年级/学科/纸张默认设置。
- `test/core/models/enums_test.dart`
- `test/core/db/app_database_test.dart`
- `test/core/files/image_derivatives_test.dart`
- `test/core/api/mock_cloud_processing_client_test.dart`
- `test/core/api/paddleocr_client_test.dart`（集成测试，CI 中跳过，需要网络）
- `test/features/print/print_layout_policy_test.dart`
- `test/widget/app_navigation_test.dart`

---

### 任务 1：Flutter 脚手架和依赖

**涉及文件：**
- 创建：`pubspec.yaml` 和 Flutter 平台文件（通过 `flutter create`）
- 修改：`.gitignore`

- [ ] **步骤 1：验证 Flutter 可用**

运行：

```bash
flutter --version
```

预期：打印 Flutter SDK 版本号。如打印 `command not found`，先安装 Flutter SDK，然后重新执行此步骤。

- [ ] **步骤 2：创建项目脚手架**

运行：

```bash
flutter create --project-name cuotiben --org com.soyou --platforms=ios .
```

预期：Flutter 创建 `ios/`、`lib/`、`test/` 和 `pubspec.yaml`，不删除 `docs/`。MVP 不生成 `android/`。

- [ ] **步骤 3：添加依赖**

运行：

```bash
flutter pub add flutter_riverpod go_router drift sqlite3_flutter_libs path_provider path dio image_picker camera pdf printing flutter_markdown flutter_dotenv uuid
flutter pub add --dev build_runner drift_dev mocktail
```

预期：依赖被添加到 `pubspec.yaml`。

- [ ] **步骤 3a：配置 .env（PaddleOCR）**

在项目根目录创建 `.env`：

```bash
PADDLEOCR_TOKEN=32879366a89bcdd3be5e35f5abb23f4b4d387539
PADDLEOCR_BASE_URL=https://paddleocr.aistudio-app.com/api/v2/ocr/jobs
PADDLEOCR_MODEL=PaddleOCR-VL-1.6
```

将 `.env` 加入 `.gitignore`——Token 绝不能进入版本控制。

在 `pubspec.yaml` 中声明 `.env` 为资源文件：

```yaml
flutter:
  assets:
    - .env
```

- [ ] **步骤 4：运行脚手架测试**

运行：

```bash
flutter test
```

预期：默认 widget 测试通过，或仅因即将替换默认计数器应用而失败。

- [ ] **步骤 5：提交**

```bash
git add .
git commit -m "chore: scaffold flutter app"
```

---

### 任务 2：核心枚举和打印规则

**涉及文件：**
- 创建：`lib/src/core/models/enums.dart`
- 创建：`test/core/models/enums_test.dart`

- [ ] **步骤 1：编写会失败的枚举测试**

创建 `test/core/models/enums_test.dart`：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:cuotiben/src/core/models/enums.dart';

void main() {
  group('QuestionType', () {
    test('only solution-style questions need an answer area', () {
      expect(QuestionType.choice.needsAnswerArea, isFalse);
      expect(QuestionType.fillBlank.needsAnswerArea, isFalse);
      expect(QuestionType.shortAnswer.needsAnswerArea, isTrue);
      expect(QuestionType.essay.needsAnswerArea, isTrue);
      expect(QuestionType.unknown.needsAnswerArea, isTrue);
    });

    test('parses unknown values defensively', () {
      expect(QuestionTypeX.fromStorage('choice'), QuestionType.choice);
      expect(QuestionTypeX.fromStorage('bad-value'), QuestionType.unknown);
    });
  });
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：

```bash
flutter test test/core/models/enums_test.dart
```

预期：失败，因为 `enums.dart` 不存在。

- [ ] **步骤 3：实现枚举**

创建 `lib/src/core/models/enums.dart`：

```dart
enum Subject { math, physics }

enum GradeLevel { juniorHigh, seniorHigh }

enum ProcessingStatus {
  pending,
  processing,
  cleaned,
  needsRepair,
  solutionReady,
  failed,
}

enum QuestionType {
  choice,
  fillBlank,
  shortAnswer,
  essay,
  unknown,
}

extension QuestionTypeX on QuestionType {
  bool get needsAnswerArea {
    return switch (this) {
      QuestionType.choice => false,
      QuestionType.fillBlank => false,
      QuestionType.shortAnswer => true,
      QuestionType.essay => true,
      QuestionType.unknown => true,
    };
  }

  String get storageValue {
    return switch (this) {
      QuestionType.choice => 'choice',
      QuestionType.fillBlank => 'fill_blank',
      QuestionType.shortAnswer => 'short_answer',
      QuestionType.essay => 'essay',
      QuestionType.unknown => 'unknown',
    };
  }

  static QuestionType fromStorage(String value) {
    return switch (value) {
      'choice' => QuestionType.choice,
      'fill_blank' => QuestionType.fillBlank,
      'short_answer' => QuestionType.shortAnswer,
      'essay' => QuestionType.essay,
      _ => QuestionType.unknown,
    };
  }
}
```

- [ ] **步骤 4：运行测试验证通过**

运行：

```bash
flutter test test/core/models/enums_test.dart
```

预期：通过。

- [ ] **步骤 5：提交**

```bash
git add lib/src/core/models/enums.dart test/core/models/enums_test.dart
git commit -m "feat: add mvp domain enums"
```

---

### 任务 3：本地 Drift 数据库

**涉及文件：**
- 创建：`lib/src/core/db/app_database.dart`
- 创建：`lib/src/core/db/app_database_provider.dart`
- 创建：`test/core/db/app_database_test.dart`

- [ ] **步骤 1：编写会失败的数据库测试**

创建 `test/core/db/app_database_test.dart`：

```dart
import 'package:drift/native.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:cuotiben/src/core/db/app_database.dart';

void main() {
  test('inserts and reads a mistake question', () async {
    final db = AppDatabase.forTesting(NativeDatabase.memory());
    addTearDown(db.close);

    final id = await db.into(db.mistakeQuestions).insert(
      MistakeQuestionsCompanion.insert(
        sourceType: 'gallery',
        sourcePaperImagePath: '/tmp/paper.jpg',
        originalQuestionImagePath: '/tmp/question.jpg',
        thumbnailImagePath: '/tmp/question-thumb.jpg',
        subject: 'math',
        gradeLevel: 'senior_high',
        grade: '高二',
        questionNumber: '21',
        questionType: 'short_answer',
        cropRectJson: '{"x":0.1,"y":0.2,"width":0.7,"height":0.3}',
        processingStatus: 'pending',
      ),
    );

    final row = await db.mistakeQuestionsById(id).getSingle();
    expect(row.subject, 'math');
    expect(row.questionNumber, '21');
    expect(row.questionType, 'short_answer');
    expect(row.cropRectJson, contains('width'));
  });
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：

```bash
flutter test test/core/db/app_database_test.dart
```

预期：失败，因为数据库文件不存在。

- [ ] **步骤 3：实现 Drift 表**

创建 `lib/src/core/db/app_database.dart`，包含四张表：`mistake_questions`、`ocr_results`、`ai_solutions`、`print_sets`。必要字段必须与已确认的 spec 匹配。

`MistakeQuestion` 至少需包含：

- `sourceType`
- `sourcePaperImagePath`
- `originalQuestionImagePath`
- `enhancedImagePath`
- `cleanedImagePath`
- `finalRepairedImagePath`
- `thumbnailImagePath`
- `subject`
- `gradeLevel`
- `grade`
- `questionNumber`
- `questionType`
- `cropRectJson`
- `tagsJson`
- `note`
- `processingStatus`
- `createdAt`
- `updatedAt`

保持四表 MVP 边界。不要添加试卷表或编辑图层表。

包含以下查询方法：

```dart
@DriftDatabase(tables: [MistakeQuestions, OcrResults, AiSolutions, PrintSets])
class AppDatabase extends _$AppDatabase {
  AppDatabase(super.e);

  AppDatabase.forTesting(QueryExecutor executor) : super(executor);

  @override
  int get schemaVersion => 1;

  SimpleSelectStatement<MistakeQuestions, MistakeQuestion> mistakeQuestionsById(int id) {
    return select(mistakeQuestions)..where((row) => row.id.equals(id));
  }
}
```

MVP 中枚举值使用字符串列存储，以避免首次构建时生成类型转换器的额外变动。

- [ ] **步骤 4：生成 Drift 代码**

运行：

```bash
dart run build_runner build --delete-conflicting-outputs
```

预期：`lib/src/core/db/app_database.g.dart` 被生成。

- [ ] **步骤 5：运行数据库测试**

运行：

```bash
flutter test test/core/db/app_database_test.dart
```

预期：通过。

- [ ] **步骤 6：添加 Riverpod 数据库 provider**

创建 `lib/src/core/db/app_database_provider.dart`，包含一个 `Provider<AppDatabase>`，打开 app 文档目录中的数据库文件。测试可以继续使用 `AppDatabase.forTesting`。

- [ ] **步骤 7：提交**

```bash
git add lib/src/core/db test/core/db
git commit -m "feat: add local database schema"
```

---

### 任务 4：App 框架、主题和路由

**涉及文件：**
- 修改：`lib/main.dart`
- 创建：`lib/src/app/cuotiben_app.dart`
- 创建：`lib/src/app/router.dart`
- 创建：`lib/src/core/theme/app_theme.dart`
- 创建：`lib/src/features/**` 下的初始页面文件，带路由标题和 MVP 入口操作
- 创建：`test/widget/app_navigation_test.dart`

- [ ] **步骤 1：编写路由冒烟测试**

创建 `test/widget/app_navigation_test.dart`：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:cuotiben/src/app/cuotiben_app.dart';

void main() {
  testWidgets('shows capture entry screen first', (tester) async {
    await tester.pumpWidget(const CuotibenApp());
    expect(find.text('错题本'), findsOneWidget);
    expect(find.text('拍照收题'), findsOneWidget);
  });
}
```

- [ ] **步骤 2：运行测试验证失败**

运行：

```bash
flutter test test/widget/app_navigation_test.dart
```

预期：失败，因为 app 框架不存在。

- [ ] **步骤 3：实现 app 框架**

实现：

- 使用 `MaterialApp.router` 的 `CuotibenApp`。
- 路由：`/`、`/capture/select`、`/library`、`/questions/:id`、`/questions/:id/repair`、`/print`、`/settings`。
- 各占位页面带有稳定的标题和主要操作按钮。
- 一个克制的移动端 app 主题，使用清晰可读的排版风格，不含营销落地页。
- Riverpod provider 骨架：
  - `appDatabaseProvider` 管理 Drift 数据库生命周期。
  - `questionRepositoryProvider` 依赖 `appDatabaseProvider`。
  - `cloudProcessingClientProvider` MVP 阶段依赖 mock client。
  - 拍照、题库、详情、修复和打印页面使用功能模块内部的 notifier provider，不直接实例化 repository 或 client。

- [ ] **步骤 4：运行路由测试**

运行：

```bash
flutter test test/widget/app_navigation_test.dart
```

预期：通过。

- [ ] **步骤 5：提交**

```bash
git add lib test/widget
git commit -m "feat: add app shell and routes"
```

---

### 任务 5：混合云端客户端（真实 PaddleOCR + Mock 增强/清理/AI）

**涉及文件：**
- 创建：`lib/src/core/api/cloud_processing_client.dart`
- 创建：`lib/src/core/api/mock_cloud_processing_client.dart`
- 创建：`lib/src/core/api/paddleocr_client.dart`
- 创建：`lib/src/core/api/paddleocr_config.dart`
- 创建：`.env`（如未在任务 1 步骤 3a 创建）
- 创建：`test/core/api/mock_cloud_processing_client_test.dart`
- 创建：`test/core/api/paddleocr_client_test.dart`

- [ ] **步骤 1：编写测试**

创建 mock 客户端测试，断言：

- `detectPaperQuestions` 返回至少一个题框，每个题框附带可选的 `ocrText` 和 `suggestedQuestionNumber`。
- `enhanceQuestion` 返回一个假的远程 `enhancedImageUrl`。
- `cleanQuestion` 返回一个假的远程 `cleanedImageUrl` 和置信度。
- `downloadResultImage` 将 mock URL 输出复制到 app 本地文件，返回本地路径。
- `generateSolution` 返回带有固定标题的 Markdown。

创建 PaddleOCR 客户端集成测试（CI 中跳过，需要网络）：

- `submitJob` 对小型测试图片返回有效 jobId。
- `pollJob` 在 120 秒内完成。
- `fetchAndParseResult` 从 JSONL 响应中提取 rawText 和 blocksJson。
- JSONL 解析时缺失字段不崩溃（对非预期结构优雅降级）。

- [ ] **步骤 2：实现 API 契约和 PaddleOCR 配置**

为以下类型定义 Dart 类型化类：

```dart
QuestionBox
ImageProcessingResult
OcrProcessingResult
AiSolutionResult
ProcessingJobStatus
RemoteImageResult
```

创建 `paddleocr_config.dart`：

```dart
import 'package:flutter_dotenv/flutter_dotenv.dart';

class PaddleOcrConfig {
  static String get token => dotenv.env['PADDLEOCR_TOKEN'] ?? '';
  static String get baseUrl => dotenv.env['PADDLEOCR_BASE_URL'] ?? '';
  static String get model => dotenv.env['PADDLEOCR_MODEL'] ?? 'PaddleOCR-VL-1.6';
}
```

- [ ] **步骤 3：实现 PaddleOCR 客户端**

创建 `paddleocr_client.dart`，包含：

```dart
class PaddleOcrClient {
  final Dio _dio;
  final String _token;

  Future<String> submitJob(String imagePath, {OcrMode mode = OcrMode.local});
  Future<OcrJobResult> pollJob(String jobId, {Duration timeout = const Duration(seconds: 120)});
  Future<OcrParseResult> fetchAndParseResult(String jsonlUrl);
  Future<OcrParseResult> processImage(String imagePath); // submit → poll → fetch
}
```

轮询：每 5 秒一次。网络错误重试 3 次，间隔递增（2s/4s/8s）。超时 120 秒。多题模式一次提交整张试卷图，OCR 文本在后续按题框坐标拆分。

- [ ] **步骤 4：为非 OCR 服务实现 mock 客户端**

Mock 客户端保留 `enhanceQuestion`、`cleanQuestion`、`generateSolution` 为确定性 mock。`ocrQuestion` 方法委托给 `PaddleOcrClient`。

Mock 客户端必须对 mock 方法保持确定性，不调用网络。仍需模拟真实集成形态：云端步骤产生 URL，下载/复制步骤写入本地文件。

- [ ] **步骤 5：运行测试**

运行：

```bash
flutter test test/core/api/mock_cloud_processing_client_test.dart
```

PaddleOCR 集成测试跳过 CI（需要网络和 API 配额）：

```bash
flutter test --tags=integration test/core/api/paddleocr_client_test.dart || true
```

预期：mock 测试通过。

- [ ] **步骤 6：提交**

```bash
git add lib/src/core/api test/core/api .env
git commit -m "feat: add hybrid cloud client (real PaddleOCR + mock enhance/clean/AI)"
```

---

### 任务 6：橡皮擦技术验证

**涉及文件：**
- 创建：`lib/src/features/repair/eraser_canvas.dart`
- 创建：`lib/src/core/files/image_derivatives.dart`
- 创建：`test/core/files/image_derivatives_test.dart`
- 创建或修改：`test/features/repair/eraser_canvas_test.dart`

- [ ] **步骤 1：定义图片衍生策略**

编写测试：

- 列表缩略图最大宽度为 `400`。
- 修复画布解码宽度上限为配置的工作尺寸，初始值设为 `1600`。
- PDF 图片最大宽度与缩略图分开控制。
- 衍生文件名稳定，不会覆盖原始图片。

- [ ] **步骤 2：实现图片衍生辅助**

实现一个小型的图片尺寸和文件名策略层。具体实现可以稍后使用 Flutter/image API，但策略必须在导入和修复页面开始依赖原始图片路径之前存在。

- [ ] **步骤 3：构建橡皮擦画布 POC**

创建一个可复用的橡皮擦画布 widget，支持：

- 加载降采样后的工作图片。
- 通过 `CustomPainter` + 指针/手势事件追踪连续手指划动。
- 在独立的内存覆盖层上绘制白色笔触。
- 将工作图片和白色覆盖层合成并输出为字节，完成保存。
- 放弃编辑且不写入新文件。

如果真机测试显示掉帧或内存峰值，在本任务段落记录。如果目标设备上自由笔触不可行，在继续之前将本任务输出替换为白色矩形遮盖。

- [ ] **步骤 4：验证 spike**

运行：

```bash
flutter test test/core/files/image_derivatives_test.dart test/features/repair/eraser_canvas_test.dart
```

如果真机可用，在实现完整导入流程之前，手动测试一张 5-15MB 的拍摄试卷图片。

- [ ] **步骤 5：提交**

```bash
git add lib/src/core/files lib/src/features/repair test
git commit -m "feat: validate eraser repair spike"
```

---

### 任务 7：文件存储、权限和图片导入流程

**涉及文件：**
- 创建：`lib/src/core/files/app_file_store.dart`
- 修改：`ios/Runner/Info.plist`
- 修改：`lib/src/features/capture/capture_home_screen.dart`
- 修改：`lib/src/features/capture/question_selection_screen.dart`

- [ ] **步骤 0：配置并验证平台权限**

在接入图片导入之前配置 iOS 相机和图库权限：

- iOS `Info.plist` 包含 `NSCameraUsageDescription` 和 `NSPhotoLibraryUsageDescription`。
- 在 iOS 模拟器或 iPhone 上进行权限弹窗冒烟验证。

- [ ] **步骤 1：编写文件存储单元测试**

创建测试：

- 将固定图片复制到 app 本地 `papers/` 目录，返回稳定的本地路径。
- 将选中的题目图片复制到 `questions/` 目录。
- 生成并保存最大宽度 400px 的列表缩略图。
- 保持原始图片、可打印图片和缩略图路径分离。

- [ ] **步骤 2：实现 `AppFileStore`**

实现以下方法：

```dart
Future<String> copyPaperImage(File source);
Future<String> copyQuestionImage(File source);
Future<String> createThumbnail(File source);
Future<String> writeRepairedImageBytes(Uint8List bytes, String fileName);
```

题目列表页面必须使用缩略图，而非全分辨率题图。

- [ ] **步骤 3：接入图片导入**

在拍照首页使用 `image_picker`。导入后，携带本地试卷图片路径导航到错题选择页。

- [ ] **步骤 4：添加手动错题选择 MVP**

MVP 阶段，在导入图片上实现简单的可调整矩形框。多题自动框选可稍后调用 mock API；手动框选必须在无云端结果时也能工作。

- [ ] **步骤 5：验证**

运行：

```bash
flutter test
```

预期：全部测试通过。

- [ ] **步骤 6：提交**

```bash
git add ios/Runner/Info.plist lib/src/core/files lib/src/features/capture test
git commit -m "feat: add image import and question selection"
```

---

### 任务 8：错题入库

**涉及文件：**
- 修改：`lib/src/features/capture/question_selection_screen.dart`
- 创建：`lib/src/features/library/question_repository.dart`
- 修改：`lib/src/features/library/question_library_screen.dart`
- 测试：`test/features/library/question_repository_test.dart`

- [ ] **步骤 1：编写 repository 测试**

测试保存选中错题时创建的 `MistakeQuestion` 行包含：

- `sourcePaperImagePath`
- `originalQuestionImagePath`
- `thumbnailImagePath`
- `subject` 为 `math` 或 `physics`
- `gradeLevel`
- `questionNumber`
- `questionType`
- `cropRectJson`
- `processingStatus` 为 `pending`

- [ ] **步骤 2：实现 repository**

创建 `QuestionRepository`，包含：

```dart
Future<int> createQuestionFromSelection(CreateQuestionInput input);
Future<List<MistakeQuestion>> watchQuestions();
Future<void> updateQuestionType(int id, QuestionType type);
Future<void> updateFinalRepairedImagePath(int id, String path);
```

- [ ] **步骤 3：接入保存操作并触发 mock 处理管线**

错题选择后，保存并导航到题库。保存后，通过 Riverpod notifier 立即触发 mock 云端处理管线：

1. `enhanceQuestion` → 写入 `MistakeQuestion` 的 `enhancedImagePath`。
2. `cleanQuestion` → 写入 `cleanedImagePath`。
3. `ocrQuestion`（真实 PaddleOCR API 调用）→ 创建 `OcrResult` 行，可选更新 `questionNumber` 和 `questionType`。此步骤包含真实网络调用：提交图片 → 轮询 job（通常 5-30 秒）→ 下载 JSONL 结果。多题模式一次提交整张试卷图，OCR 结果在解析后按题框坐标拆分。

每一步独立运行，结果就绪后更新数据库。UI 监听 `processingStatus` 反映当前阶段。如果 OCR 失败（网络错误、job 失败、超时），该行保持之前的状态，用户仍可查看、修复和打印。Mock 增强/清理步骤即时完成，真实 OCR 步骤需真实等待时间。

- [ ] **步骤 4：运行测试**

运行：

```bash
flutter test
```

预期：通过。

- [ ] **步骤 5：提交**

```bash
git add lib/src/features/capture lib/src/features/library test/features/library
git commit -m "feat: save selected questions locally"
```

---

### 任务 9：题库列表和筛选

**涉及文件：**
- 修改：`lib/src/features/library/question_library_screen.dart`
- 修改：`lib/src/features/library/question_repository.dart`

- [ ] **步骤 1：添加 repository 筛选测试**

测试按以下条件筛选：

- subject：math/physics
- gradeLevel：juniorHigh/seniorHigh
- processingStatus

- [ ] **步骤 2：实现筛选查询**

添加查询方法，接受可空筛选字段，按最新优先排序。

- [ ] **步骤 3：实现题库列表 UI**

页面要求：

- 紧凑列表，非营销页面。
- 显示 `thumbnailImagePath` 缩略图、学科、年级、题号、处理状态。
- 数学/物理和初中/高中的分段筛选器。
- 点击行打开错题详情。

MVP 不做标签筛选 UI。`tagsJson` 字段仅作数据存储，筛选功能延后到后续版本。

- [ ] **步骤 4：运行测试**

运行：

```bash
flutter test
```

预期：通过。

- [ ] **步骤 5：提交**

```bash
git add lib/src/features/library test/features/library
git commit -m "feat: add question library filters"
```

---

### 任务 10：错题详情、OCR 和 AI 解析

**涉及文件：**
- 修改：`lib/src/features/repair/question_detail_screen.dart`
- 修改：`lib/src/features/library/question_repository.dart`
- 修改：`lib/src/core/api/mock_cloud_processing_client.dart`

- [ ] **步骤 1：为解析 Markdown 标题和 warnings 编写测试**

断言生成的 Markdown 包含：

- `## 答案`
- `## 解题思路`
- `## 详细步骤`
- `## 易错点`
- `## 同类题方法`
- `## 不确定提示`

同时断言：当 mock 返回非空 `warnings` 时，`不确定提示` 段落包含警告文字；当 `warnings` 为空时，该段落显示 `暂无补充`。

- [ ] **步骤 2：实现详情页**

显示：

- 最佳可用图片：`finalRepairedImagePath`，然后 `cleanedImagePath`，然后 `enhancedImagePath`，然后 `originalQuestionImagePath`。
- OCR 摘要（如有）。
- 可编辑的题型控件。
- 可编辑的题号。
- `答案解析` 按钮。
- `橡皮擦修复` 按钮。

- [ ] **步骤 3：实现 mock AI 生成**

点击 `答案解析` 时，调用 mock client，保存 `AiSolution`，使用 `flutter_markdown` 渲染 `explanationMarkdown`。

- [ ] **步骤 4：运行测试**

运行：

```bash
flutter test
```

预期：通过。

- [ ] **步骤 5：提交**

```bash
git add lib/src/features/repair lib/src/features/library lib/src/core/api test
git commit -m "feat: add question detail and ai solution"
```

---

### 任务 11：橡皮擦修复

**涉及文件：**
- 修改：`lib/src/features/repair/eraser_repair_screen.dart`
- 修改：`lib/src/features/repair/eraser_canvas.dart`
- 修改：`lib/src/features/library/question_repository.dart`
- 修改：`lib/src/core/files/app_file_store.dart`

- [ ] **步骤 1：编写修复持久化测试**

测试保存修复后仅更新 `MistakeQuestion` 的 `finalRepairedImagePath`，且不创建任何编辑图层行。

- [ ] **步骤 2：实现橡皮擦画布**

实现简单橡皮擦流程：

- 加载最佳可用图片。
- 加载修复工作尺寸图片，而非原始全分辨率相机文件。
- 用户在图片上绘制白色橡皮擦笔触，使用已验证的 spike 画布。
- 将合成后的图片字节保存到 `finalRepairedImagePath`。
- 提供"放弃本次修复"按钮，返回时不修改数据库。
- 保持原始题图、清理图和缩略图不变。

不实现恢复笔刷、形状、文字、线图覆盖、裁剪、旋转。

- [ ] **步骤 3：运行测试**

运行：

```bash
flutter test
```

预期：通过。

- [ ] **步骤 4：提交**

```bash
git add lib/src/features/repair lib/src/features/library lib/src/core/files test
git commit -m "feat: add eraser-only repair"
```

---

### 任务 12：PDF 导出

**涉及文件：**
- 创建：`lib/src/features/print/print_layout_policy.dart`
- 修改：`lib/src/features/print/print_builder_screen.dart`
- 创建：`test/features/print/print_layout_policy_test.dart`

- [ ] **步骤 1：编写打印策略测试**

创建 `test/features/print/print_layout_policy_test.dart`：

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:cuotiben/src/core/models/enums.dart';
import 'package:cuotiben/src/features/print/print_layout_policy.dart';

void main() {
  test('answer area is only added for solution-style questions', () {
    expect(PrintLayoutPolicy.answerAreaHeight(QuestionType.choice, AnswerAreaMode.medium), 0);
    expect(PrintLayoutPolicy.answerAreaHeight(QuestionType.fillBlank, AnswerAreaMode.medium), 0);
    expect(PrintLayoutPolicy.answerAreaHeight(QuestionType.shortAnswer, AnswerAreaMode.medium), greaterThan(0));
    expect(PrintLayoutPolicy.answerAreaHeight(QuestionType.essay, AnswerAreaMode.medium), greaterThan(0));
  });

  test('mixed question pdf layout can produce non-empty pages', () async {
    // 构造 3 道不同宽高比和题型的固定题目，
    // 然后断言返回的 PDF 字节非空且报告页数 >= 1。
  });
}
```

- [ ] **步骤 2：实现打印策略**

创建 `AnswerAreaMode` 枚举和 `PrintLayoutPolicy.answerAreaHeight`。

- [ ] **步骤 3：实现 PDF 生成**

使用 `pdf` 创建 A4 纵向页面：

- 页眉：标题、学科、日期。
- 每道题图按比例缩放。
- 仅在 `questionType.needsAnswerArea` 为真时添加作答区。
- 嵌入非常大的图片前先降采样或使用打印尺寸衍生图。
- 页脚：页码。

布局必须处理：

- 超长/超高的题图跨页。
- 多道短选择题/填空题在同一页上，无额外作答间隙。
- 极端宽高比图片不变形。

额外打印规则：

- 用户可通过拖拽或上/下箭头调整题目顺序。最终顺序写入 `PrintSet.questionIds`。
- AI 解析内容绝不嵌入 PDF，无论该题是否有 `AiSolution`。PDF 仅包含题图和作答区。
- 纸张尺寸、标题和作答区模式取自用户设置（任务 13）或打印组卷页 UI。

使用 `printing` 进行预览/分享。

暴露一个可测试的 PDF 构建结果，包含生成的字节和页数，使测试不依赖视觉比对。

- [ ] **步骤 4：运行测试**

运行：

```bash
flutter test test/features/print/print_layout_policy_test.dart
```

预期：通过。

- [ ] **步骤 5：提交**

```bash
git add lib/src/features/print test/features/print
git commit -m "feat: add question-type based pdf export"
```

---

### 任务 13：设置和 MVP 打磨

**涉及文件：**
- 修改：`lib/src/features/settings/settings_screen.dart`
- 修改：`lib/src/core/db/app_database.dart`
- 修改：`lib/src/core/theme/app_theme.dart`

- [ ] **步骤 1：添加设置页**

支持：

- 学段：初中 / 高中。
- 年级。
- 默认学科：数学 / 物理。
- 默认纸张尺寸：A4。

- [ ] **步骤 2：添加空态/加载/错误状态**

为以下场景实现可见状态：

- 题库为空。
- mock 云端处理失败。
- AI 解析生成失败。
- 图片路径缺失。

- [ ] **步骤 3：运行测试**

运行：

```bash
flutter test
```

预期：通过。

- [ ] **步骤 4：提交**

```bash
git add lib
git commit -m "feat: add settings and mvp states"
```

---

### 任务 14：端到端验证

**涉及文件：**
- 仅修改 `flutter analyze`、`flutter test` 或 iOS 冒烟验证中失败的文件。

- [ ] **步骤 1：运行静态分析**

运行：

```bash
flutter analyze
```

预期：无错误。

- [ ] **步骤 2：运行全部测试**

运行：

```bash
flutter test
```

预期：全部测试通过。

- [ ] **步骤 3：运行 iOS 冒烟测试**

在 iOS 模拟器或 iPhone 真机上运行：

```bash
flutter run -d ios
```

预期：app 启动并支持导入、保存到题库（含真实 PaddleOCR 处理）、橡皮擦修复、AI mock 解析和 PDF 预览。验证保存后 30 秒内错题详情中出现 OCR 结果。

- [ ] **步骤 4：记录 iOS 平台阻塞项**

如果因为模拟器运行时、CocoaPods、签名或设备访问缺失而无法运行 iOS，在任务备注中记录 `flutter doctor` 的具体阻塞项。Android SDK 状态在 MVP 验证中有意忽略。

- [ ] **步骤 5：提交验证修复**

```bash
git add .
git commit -m "chore: validate mvp workflow"
```

---

## 自检

Spec 覆盖：

- 目标用户画像：由筛选器、设置和 AI prompt 约束覆盖。
- 四表本地数据模型：任务 3 覆盖。
- 橡皮擦之外的图片编辑不实现：任务 6 和 11 覆盖。
- 基于题型生成 PDF 作答区：任务 12 覆盖。
- 真实 PaddleOCR 集成和 mock 增强/清理/AI：任务 5 覆盖。
- 不阻塞本地流程：任务 7-10 覆盖。

占位扫描：

- OCR 使用真实 PaddleOCR-VL API。无 mock OCR 占位。
- 图片增强、去手写和豆包 AI 解析使用 mock 服务。其真实后端集成有意置于本计划范围之外，应另立计划。

类型一致性：

- 题型值集中在 `QuestionType` 中。
- 可打印图片路径优先级始终为：`finalRepairedImagePath` → `cleanedImagePath` → `enhancedImagePath` → `originalQuestionImagePath`。
