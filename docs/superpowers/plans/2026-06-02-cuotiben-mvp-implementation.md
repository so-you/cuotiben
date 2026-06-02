# Cuotiben MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first Flutter iOS MVP for a local-first wrong-question notebook focused on junior/senior high school math and physics, with photo import, question selection, local library, real PaddleOCR, mock image enhancement/cleaning/AI solution, eraser-only repair, and PDF export.

**Architecture:** Implement a Flutter app at the repository root with a feature-first structure. The app stores images as files and keeps only necessary metadata in Drift/SQLite. OCR uses the real PaddleOCR-VL API (direct connection with `.env`-managed token). Image enhancement, hand-removal, and AI solution generation are represented by typed API interfaces plus mock services so the mobile workflow can be developed and tested before the real backend is connected.

**Tech Stack:** Flutter, Dart, Riverpod, go_router, Drift/SQLite, path_provider, image_picker, camera, dio, pdf, printing, flutter_markdown, flutter_dotenv, flutter_test.

---

## Scope

This plan implements the Flutter client MVP. OCR uses the real PaddleOCR-VL API via direct connection (see `docs/superpowers/2026-06-02-mvp-paddleocr-技术方案.md`). Image enhancement, hand-removal cleaning, and Doubao AI solution generation remain mock services with typed client contracts. The real backend for those services is intentionally deferred.

MVP constraints from the confirmed spec:

- iOS-first MVP. Android is deferred until after the iOS workflow is validated.
- Target users: junior/senior high school math and physics students, plus parent-assisted operation.
- No account system and no cloud question library.
- Four local tables only: `MistakeQuestion`, `OcrResult`, `AiSolution`, `PrintSet`.
- Trial paper image is only a local file path, not a separate table.
- Eraser-only repair. No full image editor, no brush system, no vector/line overlay, no crop/rotate editing after question selection.
- PDF answer area is question-type based: choice/fill blank no extra answer area; short answer/essay get answer area.

## Risk Controls

The eraser canvas and large-image handling are the highest implementation risks. Do not leave them until the end.

- Validate eraser drawing, image downsampling, and save-composition memory behavior immediately after database/routing setup.
- Store printable images as files, never database blobs.
- Generate a small thumbnail at import/save time and use thumbnails in list views.
- Keep original and printable image paths separate from thumbnails.
- If freehand eraser is not smooth enough on target devices, downgrade repair to simple white rectangle masking before building more UI around it.
- PaddleOCR uses the real API. The OCR client submits images, polls job status, downloads JSONL results, and writes parsed text and metadata to the local database. Token is loaded from `.env` via `flutter_dotenv` and excluded from version control.
- Cloud mock services (enhance, clean, AI) must mimic the real API shape: job returns URLs, client downloads or copies URL output into local files, and only local paths are written to the database.
- E2E validation targets iOS only for MVP. Missing iOS simulator runtime, CocoaPods, signing, or iPhone device access are environment blockers. Missing Android SDK is not an MVP blocker.

## File Structure

Create or modify these paths:

- `pubspec.yaml`: dependencies and assets.
- `lib/main.dart`: app bootstrap.
- `lib/src/app/cuotiben_app.dart`: `ProviderScope`, router, theme.
- `lib/src/app/router.dart`: go_router routes.
- `lib/src/core/theme/app_theme.dart`: restrained app theme.
- `lib/src/core/models/enums.dart`: subject, grade level, question type, processing status.
- `lib/src/core/files/app_file_store.dart`: app document directory and image copy helpers.
- `lib/src/core/files/image_derivatives.dart`: thumbnail and print-image derivative policy.
- `lib/src/core/db/app_database.dart`: Drift database and four table definitions.
- `lib/src/core/db/app_database_provider.dart`: Riverpod database provider.
- `.env`: PaddleOCR token and URL (gitignored, loaded by flutter_dotenv).
- `lib/src/core/api/cloud_processing_client.dart`: API interface and job/result models.
- `lib/src/core/api/mock_cloud_processing_client.dart`: deterministic mock enhance/clean/AI service. OCR delegates to the real PaddleOCR client.
- `lib/src/core/api/paddleocr_client.dart`: real PaddleOCR-VL API client (submit, poll, fetch, parse).
- `lib/src/core/api/paddleocr_config.dart`: loads PaddleOCR token and URL from `.env`.
- `lib/src/features/capture/capture_home_screen.dart`: mode selection and image import entry.
- `lib/src/features/capture/question_selection_screen.dart`: manual/mocked question boxes and save-to-library action.
- `lib/src/features/library/question_library_screen.dart`: filterable local question list.
- `lib/src/features/repair/question_detail_screen.dart`: images, metadata, AI answer, navigation to eraser repair.
- `lib/src/features/repair/eraser_repair_screen.dart`: eraser-only repair canvas.
- `lib/src/features/print/print_builder_screen.dart`: select questions and export PDF.
- `lib/src/features/settings/settings_screen.dart`: grade/subject/paper defaults.
- `test/core/models/enums_test.dart`
- `test/core/db/app_database_test.dart`
- `test/core/files/image_derivatives_test.dart`
- `test/core/api/mock_cloud_processing_client_test.dart`
- `test/core/api/paddleocr_client_test.dart` (integration test, skipped in CI, requires network)
- `test/features/print/print_layout_policy_test.dart`
- `test/widget/app_navigation_test.dart`

---

### Task 1: Flutter Scaffold And Dependencies

**Files:**
- Create: `pubspec.yaml` and Flutter platform files via `flutter create`
- Modify: `.gitignore`

- [ ] **Step 1: Verify Flutter is available**

Run:

```bash
flutter --version
```

Expected: prints a Flutter SDK version. If it prints `command not found`, install Flutter SDK first, then rerun this step.

- [ ] **Step 2: Scaffold the app**

Run:

```bash
flutter create --project-name cuotiben --org com.soyou --platforms=ios .
```

Expected: Flutter creates `ios/`, `lib/`, `test/`, and `pubspec.yaml` without deleting `docs/`. Do not generate `android/` for the MVP.

- [ ] **Step 3: Add dependencies**

Run:

```bash
flutter pub add flutter_riverpod go_router drift sqlite3_flutter_libs path_provider path dio image_picker camera pdf printing flutter_markdown flutter_dotenv uuid
flutter pub add --dev build_runner drift_dev mocktail
```

Expected: dependencies are added to `pubspec.yaml`.

- [ ] **Step 3a: Configure .env for PaddleOCR**

Create `.env` at the project root:

```bash
PADDLEOCR_TOKEN=32879366a89bcdd3be5e35f5abb23f4b4d387539
PADDLEOCR_BASE_URL=https://paddleocr.aistudio-app.com/api/v2/ocr/jobs
PADDLEOCR_MODEL=PaddleOCR-VL-1.6
```

Add `.env` to `.gitignore` — the token must never enter version control.

Declare `.env` as an asset in `pubspec.yaml`:

```yaml
flutter:
  assets:
    - .env
```

- [ ] **Step 4: Run the scaffold test**

Run:

```bash
flutter test
```

Expected: the default widget test passes or fails only because the default counter app is about to be replaced.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "chore: scaffold flutter app"
```

---

### Task 2: Core Enums And Print Rules

**Files:**
- Create: `lib/src/core/models/enums.dart`
- Create: `test/core/models/enums_test.dart`

- [ ] **Step 1: Write the failing enum tests**

Create `test/core/models/enums_test.dart`:

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

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
flutter test test/core/models/enums_test.dart
```

Expected: fails because `enums.dart` does not exist.

- [ ] **Step 3: Implement enums**

Create `lib/src/core/models/enums.dart`:

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

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
flutter test test/core/models/enums_test.dart
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/src/core/models/enums.dart test/core/models/enums_test.dart
git commit -m "feat: add mvp domain enums"
```

---

### Task 3: Local Drift Database

**Files:**
- Create: `lib/src/core/db/app_database.dart`
- Create: `lib/src/core/db/app_database_provider.dart`
- Create: `test/core/db/app_database_test.dart`

- [ ] **Step 1: Write the failing database test**

Create `test/core/db/app_database_test.dart`:

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

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
flutter test test/core/db/app_database_test.dart
```

Expected: fails because database files do not exist.

- [ ] **Step 3: Implement Drift tables**

Create `lib/src/core/db/app_database.dart` with four tables: `mistake_questions`, `ocr_results`, `ai_solutions`, `print_sets`. Required fields must match the confirmed spec.

`MistakeQuestion` must include at minimum:

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

Keep the four-table MVP boundary. Do not add a paper table or edit-layer table.

Include this query method:

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

Use string columns for enum values in MVP to avoid generated converter churn during the first build.

- [ ] **Step 4: Generate Drift code**

Run:

```bash
dart run build_runner build --delete-conflicting-outputs
```

Expected: `lib/src/core/db/app_database.g.dart` is generated.

- [ ] **Step 5: Run database test**

Run:

```bash
flutter test test/core/db/app_database_test.dart
```

Expected: PASS.

- [ ] **Step 6: Add Riverpod database provider**

Create `lib/src/core/db/app_database_provider.dart` with a `Provider<AppDatabase>` that opens the app documents directory database file. Tests can keep using `AppDatabase.forTesting`.

- [ ] **Step 7: Commit**

```bash
git add lib/src/core/db test/core/db
git commit -m "feat: add local database schema"
```

---

### Task 4: App Shell, Theme, And Routing

**Files:**
- Modify: `lib/main.dart`
- Create: `lib/src/app/cuotiben_app.dart`
- Create: `lib/src/app/router.dart`
- Create: `lib/src/core/theme/app_theme.dart`
- Create: initial screen files under `lib/src/features/**` with route titles and MVP entry actions
- Create: `test/widget/app_navigation_test.dart`

- [ ] **Step 1: Write routing smoke test**

Create `test/widget/app_navigation_test.dart`:

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

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
flutter test test/widget/app_navigation_test.dart
```

Expected: fails because app shell does not exist.

- [ ] **Step 3: Implement app shell**

Implement:

- `CuotibenApp` with `MaterialApp.router`.
- Routes: `/`, `/capture/select`, `/library`, `/questions/:id`, `/questions/:id/repair`, `/print`, `/settings`.
- Placeholder screens with stable titles and primary buttons.
- A quiet mobile app theme with readable typography and no marketing landing page.
- Riverpod provider skeleton:
  - `appDatabaseProvider` owns the Drift database lifecycle.
  - `questionRepositoryProvider` depends on `appDatabaseProvider`.
  - `cloudProcessingClientProvider` depends on the mock client in MVP.
  - Capture, library, detail, repair, and print screens use feature-local notifier providers; they do not instantiate repositories or clients directly.

- [ ] **Step 4: Run routing test**

Run:

```bash
flutter test test/widget/app_navigation_test.dart
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib test/widget
git commit -m "feat: add app shell and routes"
```

---

### Task 5: Hybrid Cloud Client (Real PaddleOCR + Mock Enhance/Clean/AI)

**Files:**
- Create: `lib/src/core/api/cloud_processing_client.dart`
- Create: `lib/src/core/api/mock_cloud_processing_client.dart`
- Create: `lib/src/core/api/paddleocr_client.dart`
- Create: `lib/src/core/api/paddleocr_config.dart`
- Create: `.env` (if not created in Task 1 Step 3a)
- Create: `test/core/api/mock_cloud_processing_client_test.dart`
- Create: `test/core/api/paddleocr_client_test.dart`

- [ ] **Step 1: Write tests**

Create mock client tests asserting:

- `detectPaperQuestions` returns at least one box, each with an optional `ocrText` and `suggestedQuestionNumber`.
- `enhanceQuestion` returns a fake remote `enhancedImageUrl`.
- `cleanQuestion` returns a fake remote `cleanedImageUrl` and confidence.
- `downloadResultImage` copies a mock URL output into an app-local file and returns a local path.
- `generateSolution` returns Markdown with fixed headings.

Create PaddleOCR client integration test (skipped in CI, requires network):

- `submitJob` returns a valid jobId for a small test image.
- `pollJob` completes within 120 seconds.
- `fetchAndParseResult` extracts rawText and blocksJson from a JSONL response.
- JSONL parsing handles missing fields gracefully (no crash on unexpected structure).

- [ ] **Step 2: Implement API contracts and PaddleOCR config**

Define typed Dart classes for:

```dart
QuestionBox
ImageProcessingResult
OcrProcessingResult
AiSolutionResult
ProcessingJobStatus
RemoteImageResult
```

Create `paddleocr_config.dart`:

```dart
import 'package:flutter_dotenv/flutter_dotenv.dart';

class PaddleOcrConfig {
  static String get token => dotenv.env['PADDLEOCR_TOKEN'] ?? '';
  static String get baseUrl => dotenv.env['PADDLEOCR_BASE_URL'] ?? '';
  static String get model => dotenv.env['PADDLEOCR_MODEL'] ?? 'PaddleOCR-VL-1.6';
}
```

- [ ] **Step 3: Implement PaddleOCR client**

Create `paddleocr_client.dart` with:

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

Polling: every 5 seconds. Retry on network error 3× with exponential backoff (2s/4s/8s). Timeout at 120s. Multi-question mode submits the full paper image once; OCR text is split by question box coordinates downstream.

- [ ] **Step 4: Implement mock client for non-OCR services**

The mock client keeps `enhanceQuestion`, `cleanQuestion`, and `generateSolution` as deterministic mocks. The `ocrQuestion` method delegates to `PaddleOcrClient`.

The mock client must be deterministic and not call the network for its mocked methods. It should still mimic the real integration shape: cloud steps produce URLs, then a download/copy step writes local files.

- [ ] **Step 5: Run tests**

Run:

```bash
flutter test test/core/api/mock_cloud_processing_client_test.dart
```

Skip PaddleOCR integration test in CI (requires network and API quota):

```bash
flutter test --tags=integration test/core/api/paddleocr_client_test.dart || true
```

Expected: mock tests PASS.

- [ ] **Step 6: Commit**

```bash
git add lib/src/core/api test/core/api .env
git commit -m "feat: add hybrid cloud client (real PaddleOCR + mock enhance/clean/AI)"
```

---

### Task 6: Eraser Technical Spike

**Files:**
- Create: `lib/src/features/repair/eraser_canvas.dart`
- Create: `lib/src/core/files/image_derivatives.dart`
- Create: `test/core/files/image_derivatives_test.dart`
- Create or modify: `test/features/repair/eraser_canvas_test.dart`

- [ ] **Step 1: Define the image derivative policy**

Write tests for:

- list thumbnail max width is `400`.
- repair canvas decode width is capped to a configured working size, initially `1600`.
- PDF image max width is capped separately from the thumbnail.
- derivative filenames are stable and do not overwrite original images.

- [ ] **Step 2: Implement image derivative helpers**

Implement a small policy layer for image sizes and filenames. The implementation can use Flutter/image APIs later, but the policy must exist before import and repair screens start depending on raw image paths.

- [ ] **Step 3: Build eraser canvas POC**

Create a reusable eraser canvas widget that supports:

- loading a downsampled working image.
- tracking continuous finger strokes with `CustomPainter` + pointer/gesture events.
- drawing white strokes over a separate in-memory overlay.
- saving by composing the working image and white overlay to bytes.
- abandoning the edit without writing a new file.

Record implementation notes in this task section if device testing shows frame drops or memory spikes. If freehand strokes are not viable on target devices, replace this task's output with white rectangle masking before continuing.

- [ ] **Step 4: Validate spike**

Run:

```bash
flutter test test/core/files/image_derivatives_test.dart test/features/repair/eraser_canvas_test.dart
```

If a real device is available, manually test one 5-15MB photographed paper image before implementing the full import flow.

- [ ] **Step 5: Commit**

```bash
git add lib/src/core/files lib/src/features/repair test
git commit -m "feat: validate eraser repair spike"
```

---

### Task 7: File Store, Permissions, And Image Import Flow

**Files:**
- Create: `lib/src/core/files/app_file_store.dart`
- Modify: `ios/Runner/Info.plist`
- Modify: `lib/src/features/capture/capture_home_screen.dart`
- Modify: `lib/src/features/capture/question_selection_screen.dart`

- [ ] **Step 0: Configure and verify platform permissions**

Configure iOS camera and gallery permissions before wiring image import:

- iOS `Info.plist` includes `NSCameraUsageDescription` and `NSPhotoLibraryUsageDescription`.
- Run an iOS simulator or iPhone smoke check for the permission prompt.

- [ ] **Step 1: Write file-store unit test**

Create tests that:

- copy a fixture image into an app-local `papers/` directory and return a stable local path.
- copy a selected question image into `questions/`.
- generate and store a max-width 400px thumbnail for library lists.
- keep original image, printable image, and thumbnail paths distinct.

- [ ] **Step 2: Implement `AppFileStore`**

Implement methods:

```dart
Future<String> copyPaperImage(File source);
Future<String> copyQuestionImage(File source);
Future<String> createThumbnail(File source);
Future<String> writeRepairedImageBytes(Uint8List bytes, String fileName);
```

Question list screens must use thumbnails, not full-resolution question images.

- [ ] **Step 3: Wire image import**

Use `image_picker` from the capture home screen. After import, navigate to question selection with the local paper image path.

- [ ] **Step 4: Add manual question selection MVP**

For MVP planning, implement a simple adjustable rectangle over the imported image. Multi-question auto boxes can call the mock API later; manual box selection must work even without cloud results.

- [ ] **Step 5: Validate**

Run:

```bash
flutter test
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add ios/Runner/Info.plist lib/src/core/files lib/src/features/capture test
git commit -m "feat: add image import and question selection"
```

---

### Task 8: Save Selected Questions To Library

**Files:**
- Modify: `lib/src/features/capture/question_selection_screen.dart`
- Create: `lib/src/features/library/question_repository.dart`
- Modify: `lib/src/features/library/question_library_screen.dart`
- Test: `test/features/library/question_repository_test.dart`

- [ ] **Step 1: Write repository test**

Test that saving a selected question creates a `MistakeQuestion` row with:

- `sourcePaperImagePath`
- `originalQuestionImagePath`
- `thumbnailImagePath`
- `subject` as `math` or `physics`
- `gradeLevel`
- `questionNumber`
- `questionType`
- `cropRectJson`
- `processingStatus` as `pending`

- [ ] **Step 2: Implement repository**

Create `QuestionRepository` with:

```dart
Future<int> createQuestionFromSelection(CreateQuestionInput input);
Future<List<MistakeQuestion>> watchQuestions();
Future<void> updateQuestionType(int id, QuestionType type);
Future<void> updateFinalRepairedImagePath(int id, String path);
```

- [ ] **Step 3: Wire save action and trigger mock processing pipeline**

After selecting a question, save it and navigate to library. After save, immediately trigger the mock cloud processing pipeline through a Riverpod notifier:

1. `enhanceQuestion` → writes `enhancedImagePath` on the `MistakeQuestion` row.
2. `cleanQuestion` → writes `cleanedImagePath`.
3. `ocrQuestion` (real PaddleOCR API call) → creates an `OcrResult` row and optionally updates `questionNumber` and `questionType`. This step involves a real network call: submit image → poll job (5-30s typical) → download JSONL result. Multi-question mode submits the full paper image once; OCR results are split by question box coordinates after parsing.

Each step runs independently and updates the database when its result is ready. The UI watches `processingStatus` and reflects the current stage. If OCR fails (network error, job failed, timeout), the row stays in its previous state and the user can still view, repair, and print. Mock enhance/clean steps resolve instantly; the real OCR step takes real time.

- [ ] **Step 4: Run tests**

Run:

```bash
flutter test
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/src/features/capture lib/src/features/library test/features/library
git commit -m "feat: save selected questions locally"
```

---

### Task 9: Library List And Filters

**Files:**
- Modify: `lib/src/features/library/question_library_screen.dart`
- Modify: `lib/src/features/library/question_repository.dart`

- [ ] **Step 1: Add repository filter test**

Test filtering by:

- subject: math/physics
- gradeLevel: juniorHigh/seniorHigh
- processingStatus

- [ ] **Step 2: Implement filter query**

Add a query method that accepts nullable filter fields and orders newest first.

- [ ] **Step 3: Implement library UI**

Screen requirements:

- dense list, not a marketing page.
- thumbnail from `thumbnailImagePath`, subject, grade, question number, processing status.
- segmented filters for math/physics and junior/senior high.
- tapping a row opens question detail.

- [ ] **Step 4: Run tests**

Run:

```bash
flutter test
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/src/features/library test/features/library
git commit -m "feat: add question library filters"
```

---

### Task 10: Question Detail, OCR, And AI Solution

**Files:**
- Modify: `lib/src/features/repair/question_detail_screen.dart`
- Modify: `lib/src/features/library/question_repository.dart`
- Modify: `lib/src/core/api/mock_cloud_processing_client.dart`

- [ ] **Step 1: Add tests for solution Markdown headings and warnings**

Assert generated Markdown includes:

- `## 答案`
- `## 解题思路`
- `## 详细步骤`
- `## 易错点`
- `## 同类题方法`
- `## 不确定提示`

Also assert that when the mock returns non-empty `warnings`, the `不确定提示` section contains the warning text, and when `warnings` is empty the section reads `暂无补充`.

- [ ] **Step 2: Implement detail screen**

Show:

- best available image: `finalRepairedImagePath`, then `cleanedImagePath`, then `enhancedImagePath`, then `originalQuestionImagePath`.
- OCR summary if available.
- editable question type control.
- editable question number.
- `答案解析` button.
- `橡皮擦修复` button.

- [ ] **Step 3: Implement mock AI generation**

When tapping `答案解析`, call mock client, save an `AiSolution`, and render `explanationMarkdown` with `flutter_markdown`.

- [ ] **Step 4: Run tests**

Run:

```bash
flutter test
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/src/features/repair lib/src/features/library lib/src/core/api test
git commit -m "feat: add question detail and ai solution"
```

---

### Task 11: Eraser-Only Repair

**Files:**
- Modify: `lib/src/features/repair/eraser_repair_screen.dart`
- Modify: `lib/src/features/repair/eraser_canvas.dart`
- Modify: `lib/src/features/library/question_repository.dart`
- Modify: `lib/src/core/files/app_file_store.dart`

- [ ] **Step 1: Write repair persistence test**

Test that saving repair updates only `finalRepairedImagePath` on `MistakeQuestion` and does not create any edit-layer row.

- [ ] **Step 2: Implement eraser canvas**

Implement a simple eraser-only flow:

- load best available image.
- load a repair working-size image, not the raw full-resolution camera file.
- user draws white eraser strokes over the image using the spike-tested canvas.
- save composed image bytes to `finalRepairedImagePath`.
- provide “放弃本次修复” that returns without changing the database.
- keep the original question image, cleaned image, and thumbnail untouched.

No restore brush, no shapes, no text, no line overlay, no crop, no rotate.

- [ ] **Step 3: Run tests**

Run:

```bash
flutter test
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add lib/src/features/repair lib/src/features/library lib/src/core/files test
git commit -m "feat: add eraser-only repair"
```

---

### Task 12: PDF Export

**Files:**
- Create: `lib/src/features/print/print_layout_policy.dart`
- Modify: `lib/src/features/print/print_builder_screen.dart`
- Create: `test/features/print/print_layout_policy_test.dart`

- [ ] **Step 1: Write print policy test**

Create `test/features/print/print_layout_policy_test.dart`:

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
    // Build 3 fixture questions with different aspect ratios and question types,
    // then assert the returned PDF bytes are non-empty and reported page count is >= 1.
  });
}
```

- [ ] **Step 2: Implement print policy**

Create an `AnswerAreaMode` enum and `PrintLayoutPolicy.answerAreaHeight`.

- [ ] **Step 3: Implement PDF generation**

Use `pdf` to create A4 portrait pages:

- header: title, subject, date.
- each question image scaled proportionally.
- answer area only if `questionType.needsAnswerArea`.
- downsample or use print-sized derivatives before embedding very large images.
- footer: page number.

The layout must handle:

- a long/tall question image spanning pages if needed.
- several short choice/fill-blank questions on one page without extra answer gaps.
- extreme wide or narrow aspect ratios without distortion.

Additional print rules:

- User can reorder questions by dragging or up/down buttons before export. The final order is written to `PrintSet.questionIds`.
- AI solution content is never embedded in the PDF, regardless of whether an `AiSolution` exists for the question. The PDF contains only question images and answer areas.
- The paper size, title, and answer area mode are taken from user settings (Task 13) or the print builder UI.

Use `printing` for preview/share.

Expose a small testable PDF builder result that includes the generated bytes and page count so tests do not depend on visual inspection.

- [ ] **Step 4: Run tests**

Run:

```bash
flutter test test/features/print/print_layout_policy_test.dart
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/src/features/print test/features/print
git commit -m "feat: add question-type based pdf export"
```

---

### Task 13: Settings And MVP Polish

**Files:**
- Modify: `lib/src/features/settings/settings_screen.dart`
- Modify: `lib/src/core/db/app_database.dart`
- Modify: `lib/src/core/theme/app_theme.dart`

- [ ] **Step 1: Add settings screen**

Support:

- grade level: 初中 / 高中.
- grade.
- default subject: 数学 / 物理.
- default paper size: A4.

- [ ] **Step 2: Add empty/loading/error states**

Implement visible states for:

- empty library.
- mock cloud processing failure.
- AI generation failure.
- missing image path.

- [ ] **Step 3: Run tests**

Run:

```bash
flutter test
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add lib
git commit -m "feat: add settings and mvp states"
```

---

### Task 14: End-To-End Validation

**Files:**
- Modify: only files that fail `flutter analyze`, `flutter test`, or iOS smoke validation.

- [ ] **Step 1: Run static analysis**

Run:

```bash
flutter analyze
```

Expected: no errors.

- [ ] **Step 2: Run all tests**

Run:

```bash
flutter test
```

Expected: all tests pass.

- [ ] **Step 3: Run iOS smoke test**

Run on an iOS simulator or iPhone device:

Run:

```bash
flutter run -d ios
```

Expected: app launches and supports import, save to library (with real PaddleOCR processing), eraser repair, AI mock solution, and PDF preview. Verify OCR result appears in question detail within 30 seconds after save.

- [ ] **Step 4: Record iOS platform blockers**

If iOS cannot be run because simulator runtimes, CocoaPods, signing, or device access are missing, record the exact `flutter doctor` blocker in the task notes. Android SDK status is intentionally ignored for MVP validation.

- [ ] **Step 5: Commit validation fixes**

```bash
git add .
git commit -m "chore: validate mvp workflow"
```

---

## Self-Review

Spec coverage:

- Target persona is covered by filters, settings, and AI prompt constraints.
- Four-table local model is covered by Task 3.
- No image editor beyond eraser is covered by Tasks 6 and 11.
- Question-type PDF answer area is covered by Task 12.
- Real PaddleOCR integration and mock enhance/clean/AI are covered by Task 5.
- Non-blocking local flow is covered by Tasks 7-10.

Placeholder scan:

- OCR uses the real PaddleOCR-VL API. No mock OCR placeholder.
- Image enhancement, hand-removal, and Doubao AI solution use mock services. Their real backend integration is intentionally outside this plan and should get a separate plan.

Type consistency:

- Question type values are centralized in `QuestionType`.
- The printable image path precedence is consistently `finalRepairedImagePath`, `cleanedImagePath`, `enhancedImagePath`, `originalQuestionImagePath`.
