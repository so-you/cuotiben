# 错题本 Flutter App MVP 产品设计草稿

状态：草稿，随需求确认持续更新
日期：2026-06-02

## 1. 产品定位

这是一款面向中小学生、部分大学生和家长的手机错题本管理软件，使用 Flutter 开发 Android 和 iOS 双端应用。

产品核心不是普通 OCR 错题本，而是解决纸质试卷错题整理时无法可靠保留示例图、几何图、物理示意图、公式和原题版式的问题。MVP 的目标是让用户拍照收集错题后，能得到接近原题的可打印练习稿，去掉学生已经写过的手写答案、批改痕迹和草稿内容，方便重新打印做题。

## 2. 已确认的 MVP 决策

- 首版采用“图片主数据 + OCR 辅助”的架构。
- 题目以清理后的题图作为打印和复习的主数据，避免几何图、物理图、公式和版式在 OCR 转写中丢失。
- OCR 使用类似 PaddleOCR 的能力，主要用于题号、文字摘要、搜索、分类、AI 解析 prompt 和辅助排版。
- OCR、自动题框、自动去手写、AI 解析由云端提供。
- 云端处理不能阻塞用户继续整理错题。上传慢、OCR 失败或自动清理失败时，用户仍然可以手动框题、入库、编辑和打印。
- MVP 不做账号和云端错题库，错题数据保存到手机本地。后续版本再考虑账号、云同步和多设备能力。
- MVP 支持中小学全科，数学和物理图形题作为核心验收样例。
- MVP 支持 Android 和 iOS。
- 打印默认输出“最终编辑题图 + 空白作答区”，不默认打印答案解析。
- 错题详情提供“答案解析”按钮，由 AI 自动生成解析。
- MVP 不做复杂几何图/物理图完整矢量重绘，采用原图修补 + 简单线图覆盖层。

## 3. 主流程

1. 用户拍照试卷原图，或从相册导入试卷图片。
2. 用户选择单题模式或多题模式。
3. 单题模式下，用户调整一个题目框；多题模式下，系统尽量自动识别多道题框，用户勾选错题并可调整题框。
4. 用户选择错题后，错题先进入本地题库，保存原始题图。
5. 系统在题库内异步处理错题：图片显示效果增强、PaddleOCR 识别、自动去除手写笔记和答案。
6. 用户进入单题编辑页，对每个题目进行人工修正，包括橡皮擦、恢复、裁剪、旋转、黑白增强、撤销/重做、简单线图覆盖。
7. 用户在错题详情页点击答案解析按钮，查看 AI 生成的答案和解题步骤。
8. 用户从错题库选择多道题，进入打印组卷页，设置空白作答区并导出 PDF。

## 4. 页面结构

### 4.1 首页 / 拍照入口

- 进入相机拍试卷。
- 选择单题模式或多题模式。
- 支持从相册导入试卷图片。

### 4.2 错题选择页

- 单题模式：用户调整一个题目框。
- 多题模式：系统尽量自动框出多道题，用户勾选错题并可拖拽调整题框。
- 即使云端识别慢，也允许用户手动框选并入库。

### 4.3 错题库

- 按学科、年级、时间、标签筛选。
- 每道错题展示缩略图、题号/OCR 摘要、处理状态。
- 处理状态包括：待增强、处理中、已清理、需人工修正、解析已生成。

### 4.4 错题详情 / 编辑页

- 展示原始题图、系统增强清理图、用户最终编辑图。
- 支持橡皮擦、恢复笔刷、裁剪、旋转、黑白增强、撤销/重做。
- 支持简单线图覆盖层：直线、箭头、圆、矩形、自由线、文本标注。
- 提供答案解析按钮，调用 AI 生成或查看解析。

### 4.5 打印组卷页

- 用户从错题库选择多道题。
- 设置每题空白作答区大小。
- 导出 PDF：最终编辑题图 + 空白作答区。
- 默认不打印答案解析。

### 4.6 设置页

- 配置学段/年级、默认学科、打印纸张尺寸。
- 后续版本再加账号、云端同步、会员或家长协作。

## 5. 图像版本原则

题库中每道题至少保留三份图：

- 原始题图：从试卷原图裁剪出来的题目原始图片，用作回退和重新处理依据。
- 系统增强/清理图：云端或本地处理后的题图，包括黑白增强、去阴影、对比度增强、自动去手写。
- 用户最终编辑图：用户通过橡皮擦、恢复、裁剪和线图覆盖修正后的最终打印版本。

打印默认使用用户最终编辑图。如果用户未编辑，则使用系统增强/清理图；如果系统处理失败，则允许使用原始题图经过本地增强后的版本。

## 6. 图形修补策略

MVP 不承诺复杂数学几何图和物理图的完整矢量复原。原因是拍照畸变、细线、虚线、角标、箭头、受力方向、手写覆盖和题干标注会混在一起，自动矢量复原容易生成“干净但错误”的图。

MVP 采用以下策略：

- 保留原图中的题目图形作为主内容。
- 用户使用橡皮擦、恢复笔刷、增强和去阴影修补原图。
- 提供简单线图覆盖层，用直线、箭头、圆、矩形、自由线、文本标注补回局部线条。
- 覆盖层不破坏底图，用户可以单独删除或调整。

## 7. 本地数据模型

MVP 本地数据采用“图片文件 + SQLite 元数据”的方式。图片文件保存在 App 本地文件目录，SQLite 保存错题、处理状态、OCR、编辑层和打印记录等结构化数据。

### 7.1 PaperImage

表示一次拍照或从相册导入的试卷原图。

建议字段：

- id：本地唯一 ID。
- sourceType：camera 或 gallery。
- originalPath：试卷原图本地路径。
- width / height：原图尺寸。
- capturedAt：拍摄或导入时间。
- mode：single 或 multiple，记录本次使用单题模式还是多题模式。
- status：active 或 deleted。

### 7.2 MistakeQuestion

表示一道已进入错题库的错题。

建议字段：

- id：本地唯一 ID。
- paperImageId：关联的试卷原图 ID。
- subject：学科。
- grade：年级或学段。
- questionNumber：题号，可由 OCR 辅助识别，也允许用户手动修改。
- cropRect：题目在试卷原图中的裁剪区域。
- tags：标签列表。
- note：用户备注。
- processingStatus：待增强、处理中、已清理、需人工修正、解析已生成等状态。
- createdAt / updatedAt：创建和更新时间。

### 7.3 QuestionImageVariant

表示一道错题的不同图片版本。

建议字段：

- id：本地唯一 ID。
- questionId：关联错题 ID。
- variantType：original、enhanced、cleaned、final。
- imagePath：图片本地路径。
- width / height：图片尺寸。
- source：local、cloud 或 user_edit。
- jobId：如果由云端任务生成，记录对应任务 ID。
- createdAt：创建时间。

### 7.4 EditLayer

表示用户在题图上的编辑层，避免直接破坏底图。

建议字段：

- id：本地唯一 ID。
- questionId：关联错题 ID。
- layerType：erase、restore、line、arrow、rectangle、circle、freehand、text。
- payload：编辑层参数 JSON，例如笔刷路径、线条坐标、颜色、粗细、文本内容。
- orderIndex：图层顺序。
- visible：是否显示。
- createdAt / updatedAt：创建和更新时间。

### 7.5 OcrResult

表示 PaddleOCR 类能力返回的 OCR 结果。

建议字段：

- id：本地唯一 ID。
- questionId：关联错题 ID。
- rawText：OCR 合并文本。
- blocksJson：文本框、置信度、坐标等结构化结果 JSON。
- confidence：整体置信度。
- detectedQuestionNumber：识别出的题号。
- engine：OCR 引擎名称和版本。
- createdAt：生成时间。

### 7.6 AiSolution

表示 AI 生成的答案解析。

建议字段：

- id：本地唯一 ID。
- questionId：关联错题 ID。
- promptVersion：使用的 prompt 版本。
- modelProvider：例如 doubao。
- modelName：模型名称。
- answerText：最终答案。
- explanationMarkdown：解析内容，支持 Markdown。
- status：generating、ready、failed。
- createdAt / updatedAt：创建和更新时间。

### 7.7 PrintSet

表示一次组卷打印记录。

建议字段：

- id：本地唯一 ID。
- title：打印标题。
- questionIds：题目 ID 列表，保持用户选择顺序。
- paperSize：A4 等纸张设置。
- answerAreaMode：每题空白作答区大小设置。
- pdfPath：生成 PDF 的本地路径。
- createdAt：创建时间。

## 8. 云端接口设计

云端处理采用异步任务模式。App 上传图片后拿到 jobId，可以继续本地操作；任务完成后 App 下载处理结果并保存到本地。

### 8.1 试卷题区检测

`POST /process/paper-detect`

用途：上传试卷原图，返回多题识别的题框建议。

请求：

- image：试卷图片文件。
- mode：single 或 multiple。

返回：

- jobId：异步任务 ID。
- status：queued。

任务结果：

- boxes：题目框列表，每个框包含 x、y、width、height、confidence、suggestedQuestionNumber。

### 8.2 单题图片增强

`POST /process/question-enhance`

用途：上传单题图，返回黑白增强、去阴影、对比度增强、清晰化后的图片。

请求：

- image：单题图片文件。
- options：增强选项，例如 grayscale、contrast、shadowRemoval、deskew。

返回：

- jobId：异步任务 ID。
- status：queued。

任务结果：

- enhancedImageUrl：增强图下载地址。
- metrics：清晰度、对比度等辅助指标。

### 8.3 自动去除手写和批改痕迹

`POST /process/question-clean`

用途：上传单题图，自动去除学生手写答案、批改痕迹和草稿内容，生成可打印的清理版题图。

请求：

- image：单题图片文件。
- preservePrintedText：默认 true，要求尽量保留印刷题干、公式和示例图。
- removeTypes：handwriting、redMark、scratch 等。

返回：

- jobId：异步任务 ID。
- status：queued。

任务结果：

- cleanedImageUrl：清理图下载地址。
- maskImageUrl：识别出的清理遮罩下载地址，可用于用户编辑页展示和修正。
- confidence：清理结果置信度。

### 8.4 OCR 识别

`POST /ocr/question`

用途：使用 PaddleOCR 类能力识别题目文字、题号和文本框。

请求：

- image：单题图片文件，优先使用增强图。
- subject：可选学科。
- grade：可选年级。

返回：

- jobId：异步任务 ID。
- status：queued。

任务结果：

- rawText：合并文本。
- blocks：文本框、坐标、置信度列表。
- detectedQuestionNumber：题号。
- confidence：整体置信度。

### 8.5 AI 答案解析

`POST /ai/solution`

用途：根据题图和 OCR 文本生成答案解析。

请求：

- questionImage：题目图片，优先使用增强图或最终编辑图。
- ocrText：OCR 文本。
- subject：学科。
- grade：年级或学段。
- outputStyle：面向学生和家长的解析风格。

返回：

- jobId：异步任务 ID。
- status：queued。

任务结果：

- answerText：答案。
- explanationMarkdown：分步解析。
- warnings：模型不确定或题图信息不足时的提示。

### 8.6 查询异步任务

`GET /jobs/{jobId}`

用途：查询云端任务状态和结果。

返回：

- jobId：任务 ID。
- type：任务类型。
- status：queued、running、succeeded、failed。
- progress：进度。
- result：任务完成后的结果对象。
- error：失败原因。

## 9. 云端处理边界

- 云端图片只用于临时处理，处理结果下载到 App 本地后，后续版本需要提供自动删除策略和隐私设置。
- 云端 OCR、自动清理和 AI 解析失败时，不阻塞用户手动编辑、入库和打印。
- OCR 结果只作为辅助信息，不作为打印主数据。
- 自动去手写结果必须允许用户通过橡皮擦、恢复和图层工具修正。

## 10. 待确认设计

- AI 答案解析接口和 prompt 输出格式。
- PDF 打印版式。
- Flutter 技术架构和依赖选择。
