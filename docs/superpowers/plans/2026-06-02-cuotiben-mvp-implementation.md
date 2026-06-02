# Cuotiben MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first Flutter MVP for a local-first wrong-question notebook focused on junior/senior high school math and physics, with photo import, question selection, local library, eraser-only repair, AI/OCR mock results, and PDF export.

**Architecture:** Implement a Flutter app at the repository root with a feature-first structure. The app stores images as files and keeps only necessary metadata in Drift/SQLite. Cloud OCR, image cleanup, and AI solution generation are represented by typed API interfaces plus mock services so the mobile workflow can be developed and tested before the real backend is connected.

**Tech Stack:** Flutter, Dart, Riverpod, go_router, Drift/SQLite, path_provider, image_picker, camera, dio, pdf, printing, flutter_markdown, flutter_test.

---

## Scope

This plan implements the Flutter client MVP only. It does not build the real PaddleOCR, image cleanup, or Doubao AI backend. Those integrations are represented by mock services and typed client contracts.

MVP constraints from the confirmed spec:

- Target users: junior/senior high school math and physics students, plus parent-assisted operation.
- No account system and no cloud question library.
- Four local tables only: `MistakeQuestion`, `OcrResult`, `AiSolution`, `PrintSet`.
- Trial paper image is only a local file path, not a separate table.
- Eraser-only repair. No full image editor, no brush system, no vector/line overlay, no crop/rotate editing after question selection.
- PDF answer area is question-type based: choice/fill blank no extra answer area; short answer/essay get answer area.

## File Structure

Create or modify these paths:

- `pubspec.yaml`: dependencies and assets.
- `lib/main.dart`: app bootstrap.
- `lib/src/app/cuotiben_app.dart`: `ProviderScope`, router, theme.
- `lib/src/app/router.dart`: go_router routes.
- `lib/src/core/theme/app_theme.dart`: restrained app theme.
- `lib/src/core/models/enums.dart`: subject, grade level, question type, processing status.
- `lib/src/core/files/app_file_store.dart`: app document directory and image copy helpers.
- `lib/src/core/db/app_database.dart`: Drift database and four table definitions.
- `lib/src/core/db/app_database_provider.dart`: Riverpod database provider.
- `lib/src/core/api/cloud_processing_client.dart`: API interface and job/result models.
- `lib/src/core/api/mock_cloud_processing_client.dart`: deterministic mock OCR/cleanup/AI service.
- `lib/src/features/capture/capture_home_screen.dart`: mode selection and image import entry.
- `lib/src/features/capture/question_selection_screen.dart`: manual/mocked question boxes and save-to-library action.
- `lib/src/features/library/question_library_screen.dart`: filterable local question list.
- `lib/src/features/repair/question_detail_screen.dart`: images, metadata, AI answer, navigation to eraser repair.
- `lib/src/features/repair/eraser_repair_screen.dart`: eraser-only repair canvas.
- `lib/src/features/print/print_builder_screen.dart`: select questions and export PDF.
- `lib/src/features/settings/settings_screen.dart`: grade/subject/paper defaults.
- `test/core/models/enums_test.dart`
- `test/core/db/app_database_test.dart`
- `test/core/api/mock_cloud_processing_client_test.dart`
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
flutter create --project-name cuotiben --org com.soyou --platforms=android,ios .
```

Expected: Flutter creates `android/`, `ios/`, `lib/`, `test/`, and `pubspec.yaml` without deleting `docs/`.

- [ ] **Step 3: Add dependencies**

Run:

```bash
flutter pub add flutter_riverpod go_router drift sqlite3_flutter_libs path_provider path dio image_picker camera pdf printing flutter_markdown uuid
flutter pub add --dev build_runner drift_dev mocktail
```

Expected: dependencies are added to `pubspec.yaml`.

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
        subject: 'math',
        gradeLevel: 'senior_high',
        grade: '高二',
        questionType: 'short_answer',
        processingStatus: 'pending',
      ),
    );

    final row = await db.mistakeQuestionsById(id).getSingle();
    expect(row.subject, 'math');
    expect(row.questionType, 'short_answer');
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

Create `lib/src/core/db/app_database.dart` with four tables: `mistake_questions`, `ocr_results`, `ai_solutions`, `print_sets`. Required fields must match the confirmed spec. Include this query method:

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

### Task 5: File Store And Image Import Flow

**Files:**
- Create: `lib/src/core/files/app_file_store.dart`
- Modify: `lib/src/features/capture/capture_home_screen.dart`
- Modify: `lib/src/features/capture/question_selection_screen.dart`

- [ ] **Step 1: Write file-store unit test**

Create a test that copies a fixture image into an app-local `papers/` directory and returns a stable local path. Use a temporary directory and a small generated byte file.

- [ ] **Step 2: Implement `AppFileStore`**

Implement methods:

```dart
Future<String> copyPaperImage(File source);
Future<String> copyQuestionImage(File source);
Future<String> writeRepairedImageBytes(Uint8List bytes, String fileName);
```

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
git add lib/src/core/files lib/src/features/capture test
git commit -m "feat: add image import and question selection"
```

---

### Task 6: Mock Cloud Processing Client

**Files:**
- Create: `lib/src/core/api/cloud_processing_client.dart`
- Create: `lib/src/core/api/mock_cloud_processing_client.dart`
- Create: `test/core/api/mock_cloud_processing_client_test.dart`

- [ ] **Step 1: Write mock API tests**

Create tests asserting:

- `detectPaperQuestions` returns at least one box.
- `enhanceQuestion` returns an enhanced image path.
- `cleanQuestion` returns a cleaned image path and confidence.
- `generateSolution` returns Markdown with fixed headings.

- [ ] **Step 2: Implement API contracts**

Define typed Dart classes for:

```dart
QuestionBox
ImageProcessingResult
OcrProcessingResult
AiSolutionResult
ProcessingJobStatus
```

The mock client must be deterministic and not call network.

- [ ] **Step 3: Run tests**

Run:

```bash
flutter test test/core/api/mock_cloud_processing_client_test.dart
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add lib/src/core/api test/core/api
git commit -m "feat: add mock cloud processing client"
```

---

### Task 7: Save Selected Questions To Library

**Files:**
- Modify: `lib/src/features/capture/question_selection_screen.dart`
- Create: `lib/src/features/library/question_repository.dart`
- Modify: `lib/src/features/library/question_library_screen.dart`
- Test: `test/features/library/question_repository_test.dart`

- [ ] **Step 1: Write repository test**

Test that saving a selected question creates a `MistakeQuestion` row with:

- `sourcePaperImagePath`
- `originalQuestionImagePath`
- `subject` as `math` or `physics`
- `gradeLevel`
- `questionType`
- `processingStatus` as `pending`

- [ ] **Step 2: Implement repository**

Create `QuestionRepository` with:

```dart
Future<int> createQuestionFromSelection(CreateQuestionInput input);
Future<List<MistakeQuestion>> watchQuestions();
Future<void> updateQuestionType(int id, QuestionType type);
Future<void> updateFinalRepairedImagePath(int id, String path);
```

- [ ] **Step 3: Wire save action**

After selecting a question, save it and navigate to library.

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

### Task 8: Library List And Filters

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
- thumbnail, subject, grade, question number, processing status.
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

### Task 9: Question Detail, OCR, And AI Solution

**Files:**
- Modify: `lib/src/features/repair/question_detail_screen.dart`
- Modify: `lib/src/features/library/question_repository.dart`
- Modify: `lib/src/core/api/mock_cloud_processing_client.dart`

- [ ] **Step 1: Add tests for solution Markdown headings**

Assert generated Markdown includes:

- `## 答案`
- `## 解题思路`
- `## 详细步骤`
- `## 易错点`
- `## 同类题方法`
- `## 不确定提示`

- [ ] **Step 2: Implement detail screen**

Show:

- best available image: `finalRepairedImagePath`, then `cleanedImagePath`, then `enhancedImagePath`, then `originalQuestionImagePath`.
- OCR summary if available.
- editable question type control.
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

### Task 10: Eraser-Only Repair

**Files:**
- Modify: `lib/src/features/repair/eraser_repair_screen.dart`
- Modify: `lib/src/features/library/question_repository.dart`
- Modify: `lib/src/core/files/app_file_store.dart`

- [ ] **Step 1: Write repair persistence test**

Test that saving repair updates only `finalRepairedImagePath` on `MistakeQuestion` and does not create any edit-layer row.

- [ ] **Step 2: Implement eraser canvas**

Implement a simple eraser-only flow:

- load best available image.
- user draws white eraser strokes over the image.
- save composed image bytes to `finalRepairedImagePath`.
- provide “放弃本次修复” that returns without changing the database.

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

### Task 11: PDF Export

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
}
```

- [ ] **Step 2: Implement print policy**

Create an `AnswerAreaMode` enum and `PrintLayoutPolicy.answerAreaHeight`.

- [ ] **Step 3: Implement PDF generation**

Use `pdf` to create A4 portrait pages:

- header: title, subject, date.
- each question image scaled proportionally.
- answer area only if `questionType.needsAnswerArea`.
- footer: page number.

Use `printing` for preview/share.

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

### Task 12: Settings And MVP Polish

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

### Task 13: End-To-End Validation

**Files:**
- Modify: only files that fail `flutter analyze`, `flutter test`, or Android/iOS smoke validation.

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

- [ ] **Step 3: Run Android smoke test**

Run:

```bash
flutter run -d android
```

Expected: app launches and supports import, save to library, eraser repair, AI mock solution, and PDF preview.

- [ ] **Step 4: Run iOS smoke test**

Run:

```bash
flutter run -d ios
```

Expected: same workflow works on iOS simulator or device.

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
- No image editor beyond eraser is covered by Task 10.
- Question-type PDF answer area is covered by Task 11.
- Mock cloud OCR/AI and non-blocking local flow are covered by Tasks 6-9.

Placeholder scan:

- No task depends on a real PaddleOCR/Doubao backend.
- Real backend integration is intentionally outside this plan and should get a separate plan.

Type consistency:

- Question type values are centralized in `QuestionType`.
- The printable image path precedence is consistently `finalRepairedImagePath`, `cleanedImagePath`, `enhancedImagePath`, `originalQuestionImagePath`.
