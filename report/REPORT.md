# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Tu Hoang Giang | 2A202602363 | 100% (cá nhân) |

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `openai/gpt-4o-mini` (OpenRouter), `LAB_TEMPERATURE=0`, `recursion_limit=60`
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: `deepagents==0.7.21`, Windows 10, chạy trong Docker container (`python:3.12-slim` + git)
- Số lần chạy tác vụ đã dùng: 21 lần chạy chính thức (18 lần trong `results/{baseline,subagents,skills-auto}` + 3 lần `skills-auto-dev`), 1 lần lỗi lưu ở `results_failed/baseline-data-learn-1`, curator chạy 2 lần, 1 lần kiểm tra kết nối (giảng viên không quy định ngân sách số lần chạy)
- Commit của tag `freeze`: `8bb62a4bf22cb4dd9157ac7c953c6b03e1f7704c` (`8bb62a4`)

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): Dự đoán điều kiện `subagents` sẽ có điểm số tổng thể chỉ tương đương hoặc cải thiện rất nhỏ so với `baseline` trên các tác vụ kỹ thuật, nhưng sẽ tiêu tốn token nhiều hơn đáng kể (+40% đến +50%) và có độ trễ cao hơn. Căn cứ: Số liệu từ 3 tác vụ học cho thấy `subagents` chỉ tăng thêm 1 check kỹ thuật đạt (7/18 so với 6/18 ở baseline), trong khi số token trung bình tăng từ 45,102 lên 66,023 (+46.4%). Ở 2/3 tác vụ (`code-learn` và `logs-learn`), tác tử chính có xu hướng tự xử lý trực tiếp (`subagent_calls = 0`). Như tài liệu `02_subagents.md` và nghiên cứu của Anthropic đã chỉ ra, việc ủy quyền qua subagent phi trạng thái (stateless) tạo chi phí overhead rất lớn và dễ mất mát ngữ cảnh nếu prompt ủy quyền không chuyển tải trọn vẹn đặc tả.
- H2 (skills-auto so với baseline): Dự đoán điều kiện `skills-auto` khó cải thiện điểm số so với `baseline` trên cả tập học lẫn tập đánh giá, thậm chí điểm số có thể dao động ngang bằng hoặc thấp hơn nếu tác tử không chủ động đọc skill hoặc gặp phân tâm lặp. Căn cứ: Kết quả thực nghiệm ở Phần 3.4 (S9) cho thấy `skills_read = 0` ở toàn bộ 3 tác vụ học; tác tử không chủ động gọi `read_file` vào thư mục skill mà trực tiếp thao tác vào mã nguồn. Đồng thời, nghiên cứu **SkillsBench** ghi nhận rằng skill do LLM tự sinh (khác với skill do con người viết tay cẩn thận) trung bình không đem lại lợi ích ròng, do các chỉ dẫn thường mang tính checklist chung chung và không thay thế được khả năng suy luận kỹ thuật chuyên sâu khi gặp lỗi môi trường.
- H3 (tác vụ học so với tác vụ đánh giá): Dự đoán điểm số trên các tác vụ đánh giá (evaluation tasks) của tất cả các điều kiện sẽ thấp hơn hoặc bằng tác vụ học, đặc biệt ở nhóm check quy ước tổ chức (`rule_`). Các skill sinh ra từ curator sẽ thể hiện hiện tượng quá khớp (overfitting) với ngữ cảnh tác vụ học và có khả năng chuyển giao (transferability) rất thấp sang tác vụ đánh giá. Căn cứ: Nghiên cứu **SkillEvolBench** chỉ ra khoảng cách chuyển giao lớn giữa tập học và tập kiểm thử mới. Hơn nữa, hệ thống chấm điểm của tác vụ đánh giá chứa các "house rules" nội bộ hoàn toàn mới chưa từng xuất hiện trong phản hồi hay vết của tác vụ học. Vì curator chỉ tổng hợp từ feedback của tác vụ học, bộ skill tự sinh không thể dự đoán trước các quy tắc ẩn mới này, do đó điểm số các check quy ước trên tập đánh giá dự kiến sẽ không được cải thiện đáng kể.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep` (công cụ tệp), `execute` (shell) và `task` (subagent). Công cụ cho phép chạy lệnh shell là `execute`.
2. Mô tả công cụ `task` giới thiệu `general-purpose` là: *"General-purpose agent for researching complex questions, searching for files and content, and executing multi-step tasks."* Subagent này hoạt động độc lập và phi trạng thái theo mặc định (*stateless by default*), chỉ nhìn thấy nội dung chỉ dẫn/prompt được tác tử chính gửi sang, không nhìn thấy toàn bộ ngữ cảnh hay hội thoại trước đó của tác tử chính.
3. Trích dẫn câu hướng dẫn hành vi:
   - Từ mô tả công cụ `task`: *"Each invocation is stateless by default: the agent sees only the prompt you give it and returns a single final report. Put full detail in the prompt and state exactly what it should return — unless an agent type below says it inherits your conversation instead."*
   - Từ mô tả công cụ `execute`: *"You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail."*

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Theo số liệu đo lường trực tiếp từ `results/baseline/*-learn/run.json` (tổng cộng 27 check qua 3 tác vụ học):
- **Số check đạt**: 6/27 (toàn bộ là check kỹ thuật: 1 ở `data-learn`, 5 ở `code-learn`).
- **Số check thất bại**: 21/27, trong đó:
  - **Check kỹ thuật trượt**: **12 check** (chiếm 12/21 ≈ **57.14%** — **chiếm đa số**).
  - **Check quy ước (`rule_`) trượt**: **9 check** (chiếm 9/21 ≈ **42.86%**, tức nhóm E ≈ **43%**, không phải đa số).

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| `data-learn` | `north_q1_revenue`, `north_q1_orders`, `missing_amount_orders`, `duplicate_rows_removed` | D / F | `detail: wrong value (got 0)`. Vết trace: Tác tử chạy `python3 -c "import pandas..."` và gặp `ModuleNotFoundError: No module named 'pandas'`. Do container không có thư viện pandas (đúng theo đặc tả `pyproject.toml`, không cài thêm), tác tử không chuyển sang dùng thư viện chuẩn `csv`/`datetime` (Nhóm D) mà ghi ra file `workspace/answer.json` toàn số `0` mẫu (`{"north_q1_revenue": 0, ...}`) rồi báo cáo nộp placeholder (Nhóm F). |
| `data-learn` | `rule_money_in_cents` | E | `detail: RULE: money values in answer.json are integer cents (1606.67 USD is written 160667).` |
| `data-learn` | `rule_meta_block` | E | `detail: RULE: answer.json has an object meta = {"source": <input file name>, "rows_in": ..., "rows_used": ...}` |
| `data-learn` | `rule_clean_csv` | E | `detail: RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents...` |
| `code-learn` | `csv_quoting_follows_docstring` | A | `detail: to_csv_row returned 'Desk, large "oak",10.00,2'` (docstring yêu cầu quote/escape trường có dấu phẩy hoặc ngoặc kép nhưng tác tử không đọc kỹ đặc tả). |
| `code-learn` | `parse_price_all_formats` | D | `detail: wrong for: ['(12.00)']` (tác tử chỉ xử lý dấu phẩy, bỏ sót định dạng ngoặc kế toán biểu thị số âm/giá trị đặc biệt). |
| `code-learn` | `rule_type_hints` | E | `detail: RULE: every public function (name not starting with '_') in the package has type annotations on all parameters and on the return value.` |
| `code-learn` | `rule_regression_tests` | E | `detail: RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3); the file must pass.` |
| `code-learn` | `rule_changelog` | E | `detail: RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): <short description>'` |
| `logs-learn` | `valid_structure`, `entry_count`, `timestamps_utc`, `exception_fields`, `repeat_counts`, `counts_by_service` | B / F | `detail: FileNotFoundError: [Errno 2] No such file or directory: '.../workspace/errors.json'`. Vết trace: Tác tử chỉ thực hiện đúng **2 tool calls** (`read_file workspace/app.log` và `read_file workspace/README.md`) rồi kết thúc sớm mà không hề tạo file kết quả (Nhóm B: không kiểm chứng sau khi chạy; Nhóm F: kết thúc tác vụ khi chưa hoàn thành). |
| `logs-learn` | `rule_schema_header` | E | `detail: RULE: the top-level object has "schema_version": 2 and "generated_by": "log-triage".` |
| `logs-learn` | `rule_service_names` | E | `detail: RULE: service names in the output are lower-case with '-' replaced by '_' (payment-service -> payment_service).` |
| `logs-learn` | `rule_sorted_errors` | E | `detail: RULE: errors is sorted by service, then by timestamp_utc, ascending.` |

Nhận xét:
- **Nhóm lỗi chiếm đa số**: Các **lỗi kỹ thuật (nhóm A, B, D, F)** chiếm **đa số** với 12/21 check trượt (57.14%). Nhóm E (vi phạm quy ước tổ chức `rule_`) chỉ chiếm 9/21 check trượt (~42.86%, tức ~43%), không phải đa số.
- **Nguyên nhân chung**:
  - Với nhóm lỗi kỹ thuật (A, B, D, F): Tác tử thiếu tính kiên trì khi gặp lỗi môi trường (như không có pandas trong `data-learn` dẫn tới nản chí nộp placeholder), thiếu vòng lặp tự kiểm tra/xác nhận tệp đầu ra (như trong `logs-learn` chỉ đọc 2 file rồi dừng mà chưa ghi file `errors.json`), và bỏ sót các trường hợp định dạng biên trong docstring (`code-learn`). Lưu ý rằng môi trường Docker container chỉ có môi trường Python tiêu chuẩn và không có pandas (hoàn toàn đúng theo khai báo dependencies trong `pyproject.toml`, tuyệt đối không cài thêm thư viện ngoài).
  - Với nhóm E (quy ước tổ chức): Chiếm 42.86% lỗi. Toàn bộ 9/9 check `rule_` đều trượt vì các quy ước này là yêu cầu kiểm thử nội bộ không được nêu trong mô tả nhiệm vụ ban đầu. Tác tử zero-shot không thể tự đoán biết nếu không có tài liệu/skill hướng dẫn quy ước.
- **Khả năng phòng ngừa của Skill**: Skill hoàn toàn có thể phòng ngừa nhóm E nếu curator cung cấp checklist rõ ràng về các quy ước ẩn (định dạng cents, file clean.csv, meta block, schema_version, CHANGELOG, regression test). Đồng thời, skill dạng checklist quy trình cũng có thể hỗ trợ phòng ngừa nhóm B/D (nhắc nhở kiểm tra các trường hợp biên và xác nhận tệp đầu ra tồn tại trước khi kết thúc).

## 5. Điều kiện `subagents` (Phần 2.3)

- **Các subagent đã định nghĩa**:
  - `explorer`: Khám phá cấu trúc dự án, tìm kiếm tệp tin, phân tích codebase và xác định vị trí lỗi hoặc các file dữ liệu mà không chỉnh sửa mã nguồn.
  - `implementer`: Chỉnh sửa mã nguồn hoặc thực hiện xử lý dữ liệu theo yêu cầu kỹ thuật cụ thể.
  - `reviewer`: Rà soát đối chiếu các thay đổi với yêu cầu bài toán, kiểm tra test suite và tính toàn vẹn của mã trước khi kết thúc tác vụ.
- **`subagent_calls` ở từng tác vụ và nhận xét**:
  - `data-learn`: 1 cuộc gọi (tác tử chính gọi `implementer` khi lệnh Python với thư viện pandas bị lỗi `ModuleNotFoundError`). Lời gọi này đã giúp thực hiện tính toán qua shell script và vượt qua check `duplicate_rows_removed`, nhưng lại mất check `top_region` mà baseline đạt được, nên điểm số chung của `data-learn` không thay đổi (vẫn là 1/8).
  - `code-learn`: 0 cuộc gọi. Tác tử chính tự đọc file, chỉnh sửa qua các công cụ file và tự chạy lệnh pytest trong workspace.
  - `logs-learn`: 0 cuộc gọi. Tác tử chính tự phân tích log và tự xuất file JSON trực tiếp.
  - *Nhận xét*: Tác tử chính xu hướng tự giải quyết nếu các công cụ cơ bản hoạt động trơn tru; chỉ ủy quyền khi gặp trở ngại kỹ thuật cụ thể trong luồng thực thi chính.
- **Thông tin thiếu hoặc thừa khi giao việc**:
  - Ở `data-learn`, prompt giao việc cho `implementer` đã tóm tắt đầy đủ các trường cần tính toán trong `answer.json`, nhưng thiếu hướng dẫn chi tiết về chuẩn định dạng ngày tháng ISO-8601 UTC và không chuyển tiếp được quy ước house rules (do chưa biết). Sau khi subagent hoàn thành, tác tử chính không gọi `reviewer` để kiểm chứng độc lập mà sử dụng luôn kết quả để nộp.
- **Ảnh hưởng đến token và thời gian**:
  - **Số token trung bình**: Tăng từ **45,102** tokens (baseline) lên **66,023** tokens (subagents), tức tăng khoảng **46.4%**.
  - **Thời gian thực thi**: `code-learn` tăng từ 47.6s lên 182.7s; `data-learn` tăng từ 78.8s lên 170.0s.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- **Số lần chạy curator**: 2 lần chạy (`python -m lab.curator`).
  - *Lần 1*: Sinh 3 skill (`data-validation`, `general-error-handling`, `testing-and-documentation`). Qua đối chiếu, bộ skill phủ trực tiếp 3/9 check `rule_*` (`rule_money_in_cents`, `rule_regression_tests`, `rule_changelog`) (< 6/9).
  - *Lần 2*: Đã sao lưu bộ 1 sang `results_failed/skills-run1/` và chạy lại curator lần 2 (sinh `data-validation`, `error-handling`, `testing-and-documentation`). Tuy nhiên bộ lần 2 bị thiếu dòng hướng dẫn `regression tests` và `docstrings` cụ thể, không phủ tốt hơn bộ 1.
  - *Quyết định*: Xóa bộ lần 2 và khôi phục bộ lần 1 từ `results_failed/skills-run1/` theo đúng quy tắc quyết định. Tuyệt đối không can thiệp sửa tay nội dung skill.
- **Số skill bị xóa**: 3 skill của lần chạy 2 (thay thế bằng bộ lần 1 khôi phục). Cả 3 skill đang giữ đều vượt qua `validate_skill` với 0 lỗi.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `data-validation` | Tổng quát nhưng `description` chung chung: mô tả "validate data inputs and outputs..." không nêu rõ tác vụ cụ thể là xử lý file dữ liệu (CSV/JSON). | Không có hướng dẫn sai, nhưng thiếu: chỉ phủ 1/3 check quy tắc của `data-learn` (`rule_money_in_cents`); thiếu hướng dẫn tạo `clean.csv` và block `meta`. Thân skill chỉ 6 gạch đầu dòng chung chung (chỉ ghi "e.g., cents" thiếu công thức cụ thể). | 11 dòng, trigger: "Use when you need to validate data inputs and outputs to ensure correctness and compliance with specifications." Đo ở S9: `skills_read=0` (0/8 pass, 368,593 tokens, GraphRecursionError). Tác tử không mở đọc skill, không áp dụng checklist về cents và deduplication. |
| `general-error-handling` | Tổng quát nhưng `description` bị lệch nội dung: mô tả về xử lý lỗi trong code ("in your code"), trong khi nội dung thân skill là quy trình đọc và phân tích file log ra file JSON. | Không có hướng dẫn sai, nhưng thiếu: không phủ được check quy tắc nào trong 3 check `rule_` của `logs-learn` (thiếu `schema_version`/`generated_by`, chuẩn hóa tên service, và thứ tự sắp xếp theo service/timestamp). Thân skill chỉ 6 gạch đầu dòng chung chung. | 11 dòng, trigger: "Use when you need to ensure that error handling and logging conventions are followed in your code." Đo ở S9: `skills_read=0` (1/9 pass, 22,047 tokens). Tác tử không đọc skill, bỏ lỡ toàn bộ 3 check house rules do không nạp schema. |
| `testing-and-documentation` | Tổng quát nhưng `description` chung chung: mô tả "well-tested and documented according to project standards", không nêu rõ ngữ cảnh sửa package Python có test suite bị lỗi. | Không có hướng dẫn sai, nhưng thiếu: chỉ phủ 2/3 check quy tắc của `code-learn` (`rule_regression_tests`, `rule_changelog`); thiếu hướng dẫn `rule_type_hints` cho mọi public function. Thân skill chỉ 6 gạch đầu dòng chung chung. | 11 dòng, trigger: "Use when you need to ensure that your code is well-tested and documented according to project standards." Đo ở S9: `skills_read=0` (1/10 pass, 18,466 tokens). Tác tử không mở đọc skill, không tạo file `test_regressions.py` hay cập nhật `CHANGELOG.md`. |

Nhận xét về chất lượng skill và nguyên nhân không được đọc (`skills_read = 0`):
- **Phê bình `description`**: Cả 3 skill đều có `description` quá chung chung, trong đó `general-error-handling` bị lệch hẳn nội dung (nói về code conventions thay vì phân tích log), làm giảm khả năng tác tử nhận diện sự liên quan của skill (`05_skill_quality.md` mục 5).
- **Hành vi tác tử**: Tác tử zero-shot hoàn toàn bỏ qua chỉ dẫn hệ thống `SKILLS_NOTE` ("As your FIRST action, read the SKILL.md..."): trong toàn bộ các trace của `skills-auto`, tool call đầu tiên luôn tác động thẳng vào `workspace/`, không có bất kỳ tool call nào vào `skills/`.
- **Giới hạn độ phủ**: Kể cả nếu tác tử có đọc skill, độ phủ chỉ đạt 3/9 check quy ước của tập học cũng giới hạn mức cải thiện tối đa đối với nhóm check quy ước.

## 7. Kết quả so sánh (Phần 4.3, 4.4)

Bảng so sánh tổng hợp (`python -m lab.compare`):

| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 5/10 | 5/10 | 5/10 |
| data-learn | 1/8 | 1/8 | 1/8 |
| logs-learn | 0/9 | 1/9 | 0/9 |
| code-eval | 2/11 | 2/11 | 3/11 |
| data-eval | 0/9 | 0/9 | 3/9 |
| logs-eval | 1/10 | 1/10 | 0/10 |
| **Mean score - learning tasks** | 0.21 | 0.25 | 0.21 |
| **Mean score - evaluation tasks** | 0.09 | 0.09 | 0.20 |
| **Mean tokens per run** | 128,284 | 138,227 | 273,538 |
| **Runs that read a skill** | 0/6 | 0/6 | 0/6 |

Bảng phân rã chi tiết kỹ thuật và quy ước (`scripts/check_breakdown.py`):

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval      3/18         0/12         211,466      0/3     
baseline      learn     6/18         0/9           45,102      0/3     
subagents     eval      3/18         0/12         210,430      0/3     
subagents     learn     7/18         0/9           66,023      0/3     
skills-auto   eval      6/18         0/12         509,605      0/3     
skills-auto   learn     6/18         0/9           37,472      0/3     
```

`skills_read = 0/6` ở mọi run `skills-auto`, trong khi `skills_sha256` khớp hash đóng băng: skill được nạp vào sandbox nhưng tác tử không mở. Do đó chênh lệch 0.20 so với 0.09 trên tác vụ đánh giá không quy được cho skill.

Danh sách các lần chạy có `error` hoặc `skills_modified=true` và cách xử lý:
- **Các lần chạy có `error`**: Có 6/12 run đánh giá bị `GraphRecursionError` (gồm 6 lần chạy):
  1. `baseline code-eval`: `GraphRecursionError` (limit 60, score 2/11, tokens 236,292, tool calls 96).
  2. `baseline data-eval`: `GraphRecursionError` (limit 60, score 0/9, tokens 380,383, tool calls 30).
  3. `subagents code-eval`: `GraphRecursionError` (limit 60, score 2/11, tokens 213,586, tool calls 37).
  4. `subagents data-eval`: `GraphRecursionError` (limit 60, score 0/9, tokens 399,025, tool calls 30).
  5. `skills-auto code-eval`: `GraphRecursionError` (limit 60, score 3/11, tokens 222,854, tool calls 36).
  6. `skills-auto logs-eval`: `GraphRecursionError` (limit 60, score 0/10, tokens 1,292,867, tool calls 29).
  - *Cách xử lý*: Theo quy định thực nghiệm và hướng dẫn bài lab, lỗi `GraphRecursionError` phản ánh đúng giới hạn bước suy luận của tác tử và đã được bảo toàn đầy đủ vết trong `trace.md` (nhờ cơ chế stream). Toàn bộ các lần chạy này là dữ liệu thực nghiệm hợp lệ, được giữ nguyên vẹn trong bộ kết quả và không chạy lại.
  - *Phân tích data-eval*: Ở tác vụ `data-eval`, `baseline` và `subagents` bị rơi vào vòng lặp kẹt lỗi cú pháp `SyntaxError` lặp lại (30 tool calls, ~380k–399k tokens, cạn bước đệ quy) do viết lệnh `python -c` trên một dòng nối các câu lệnh ghép bằng dấu `;` nên đạt `0/9`; ngược lại, `skills-auto` không bị lặp mà viết một script Python nhiều dòng hoàn chỉnh bằng thư viện chuẩn (3 tool calls, 13,094 tokens), tính toán thành công và đạt `3/9` (`top_category`, `missing_total_orders`, `duplicate_events_removed`). Chênh lệch này đến từ cách tác tử định dạng câu lệnh thực thi, hoàn toàn không đến từ skill (`skills_read = 0`).
- **Các lần chạy có `skills_modified=true`**: Rỗng (toàn bộ 21 lần chạy chính thức đều ghi nhận `skills_modified=false`; các tác tử tuyệt đối không tự ý sửa đổi nội dung các tệp skill trong quá trình thực thi).

## 8. Phân tích

1. **So sánh cải thiện giữa tác vụ học và tác vụ đánh giá**:
   - Trên tác vụ **học**: Điều kiện `subagents` tăng điểm trung bình từ `0.21` (baseline) lên `0.25`, sự gia tăng này đến duy nhất từ tác vụ `logs-learn` (vượt qua check kỹ thuật `valid_structure`: 1/9 so với 0/9). Ở tác vụ `data-learn`, subagents chỉ đổi check đạt (`duplicate_rows_removed` thay vì `top_region` của baseline) nhưng tổng điểm không đổi (cùng đạt 1/8). Điều kiện `skills-auto` đạt điểm `0.21`, bằng đúng baseline.
   - Trên tác vụ **đánh giá**: Điều kiện `skills-auto` ghi nhận điểm trung bình `0.20`, cao hơn so với `0.09` của `baseline` và `subagents`. Tuy nhiên, `skills_read = 0/6` ở mọi run của `skills-auto`, trong khi `skills_sha256` khớp hash đóng băng: skill được nạp vào sandbox nhưng tác tử không mở. Do đó chênh lệch 0.20 so với 0.09 trên tác vụ đánh giá không quy được cho skill; sự chênh lệch này đến từ việc ở `data-eval`, tác tử `skills-auto` viết một script Python nhiều dòng hợp lệ nên chạy được và đạt `3/9`, trong khi `baseline` và `subagents` bị kẹt vòng lặp `SyntaxError` do gộp lệnh `python -c` trên một dòng (đều đạt `0/9`).
   - Điều kiện `subagents` cải thiện tác vụ học (`0.25` vs `0.21`) nhưng hoàn toàn không cải thiện tác vụ đánh giá (`0.09` vs `0.09`). Đây là dấu hiệu của việc **kém khái quát hóa (lack of transferability)**: các subagent chỉ tình cờ hỗ trợ được trong ngữ cảnh cụ thể của tác vụ học nhưng không chuyển giao được năng lực giải quyết vấn đề sang các tác vụ đánh giá mới khi prompt ủy quyền thiếu thông tin hoặc tác tử chính không gọi subagent.

2. **Phân rã điểm kỹ thuật vs quy ước (`rule_`) và tác động của Skill**:
   - Theo kết quả từ `check_breakdown.py`, điểm quy ước `house rules` đạt **0/9** ở tác vụ học và **0/12** ở tác vụ đánh giá trên **cả 3 điều kiện**.
   - Skill do curator sinh hoàn toàn **không giúp được nhóm check quy ước**: nguyên nhân trực tiếp là tác tử không bao giờ đọc skill (`read a skill = 0/3` ở tất cả các điều kiện).
   - Các check quy ước **mới** của tác vụ đánh giá (gồm `rule_version_bump` trong `code-eval`, `rule_sorted_keys_format` trong `data-eval`, và `rule_source_line` trong `logs-eval`) hoàn toàn không được skill giúp đỡ vì hai lý do cốt lõi: (1) Tác tử không nạp skill vào ngữ cảnh thực thi; (2) Các quy ước mới này chưa từng xuất hiện trong phản hồi hay vết của tác vụ học, nên curator không thể suy luận ra để đưa vào nội dung skill.

3. **Giải thích qua vết (`trace.md`) và `skills_read`**:
   - **Check đạt nhưng không do skill**: Ở `skills-auto data-eval`, tác tử đạt 3/9 (so với 0/9 ở baseline) gồm các check `top_category`, `missing_total_orders`, `duplicate_events_removed`. Tuy nhiên, trường `skills_read = 0` và vết trace cho thấy tác tử không dùng skill mà thực thi một script Python nhiều dòng bằng thư viện chuẩn (`json`, `datetime`, `collections.defaultdict`), khử trùng lặp theo `id`, coi `total == -1` là thiếu, và chuẩn hóa `category` (`strip().lower()`). Ba check đạt là kết quả tính toán thật từ script này. Check `march_revenue_utc` và `march_orders_utc` bị trượt có khả năng do script lọc tháng 3 trực tiếp theo `order_time.month` từ chuỗi thời gian có offset mà chưa quy đổi sang UTC (`order_time.astimezone(timezone.utc)`).
   - **Check skill không giúp**: Check `rule_changelog` và `rule_regression_tests` trong `code-learn` và `code-eval`. Mặc dù skill `testing-and-documentation` có dòng hướng dẫn rõ ràng (*"- Maintain a CHANGELOG.md...; - Include regression tests..."*), tác tử có `skills_read = 0`, hoàn toàn không mở đọc tệp skill này mà chỉ tập trung sửa mã nguồn rồi thoát, khiến 100% check quy ước này thất bại.
   - **Về nguyên nhân `skills_read = 0`**: (a) Tác tử bỏ qua chỉ dẫn `SKILLS_NOTE`, luôn hành động đầu tiên vào `workspace/` thay vì `skills/`; (b) Các `description` của skill quá chung chung hoặc lệch nội dung (như `general-error-handling` nói về code thay vì log), làm giảm khả năng kích hoạt skill; (c) Kể cả nếu được đọc, bộ skill chỉ phủ 3/9 check `rule_` của tập học, tạo giới hạn trần rất thấp đối với các check quy ước.

4. **Chi phí và hiệu quả token**:
   - Ghi rõ lần chạy `skills-auto logs-eval` với **1,292,867 token** là một **ngoại lệ** cực đoan (outlier) do tác tử sinh lặp lại chuỗi văn bản log inline khổng lồ qua 29 tool calls.
   - Thống kê chi tiết lấy trực tiếp từ `run.json` qua 6 lần chạy của từng điều kiện:
     - `baseline`: Token dao động từ 17,724 đến 380,383; **trung vị token là 54,798**; trung bình là **128,284 token**.
     - `subagents`: Token dao động từ 18,680 đến 399,025; **trung vị token là 88,844** (+62.1% so với baseline); trung bình là **138,227 token** (+7.7%).
     - `skills-auto`:
       - **Trung vị token**: **40,770 token**.
       - **Trung bình cả 6 run** (tính cả ngoại lệ): **273,539 token**.
       - **Token trung bình của `skills-auto` khi bỏ run ngoại lệ `logs-eval`** (5 run còn lại): **69,673 token**.
   - Bảng phân tích hiệu quả điểm số trên mỗi 100,000 token (Điểm TB ÷ Token TB × 100,000):

| Điều kiện | Điểm TB (6 run) | Token TB | Điểm / 100k token |
| --- | --- | --- | --- |
| baseline | 0.151 | 128,284 | 0.118 |
| subagents | 0.170 | 138,227 | 0.123 |
| skills-auto | 0.205 | 273,539 | 0.075 |
| skills-auto (bỏ `logs-eval`) | 0.246 | 69,673 | 0.353 |

   - **Nhận xét hiệu quả**: Theo trung bình 6 run, `subagents` nhỉnh hơn `baseline` rất ít (0.123 so với 0.118 điểm/100k token) và `skills-auto` thấp nhất (0.075) do một run ngoại lệ tiêu tốn 1.29M token. Nếu bỏ run ngoại lệ `logs-eval`, `skills-auto` đạt hiệu suất cao nhất (0.353 điểm/100k token), tuy nhiên chênh lệch điểm số đã được chứng minh là do nhiễu thực nghiệm (Mục 8 ý 6) chứ không phản ánh cải thiện năng lực thực chất. Do đó, **không thể kết luận điều kiện nào thực sự có hiệu quả chi phí cao hơn**. Kiến trúc đa tác tử `subagents` không đáng chi phí trong thí nghiệm này vì không hề cải thiện điểm số ở tác vụ đánh giá.

5. **Đánh giá rò rỉ dữ liệu và quá khớp**:
   - **Phòng chống rò rỉ**: Quy trình curator tuân thủ nghiêm ngặt nguyên tắc cách ly thông tin: hàm `curate_skills` chỉ đọc kết quả từ thư mục tác vụ học (`role == "learn"`), không chạm tới tác vụ đánh giá. Hàm kiểm định `validate_skill` quét toàn bộ nội dung qua danh sách `eval_markers()` để đảm bảo không xuất hiện bất kỳ chuỗi định danh nào của tập đánh giá. Cả 3 skill đều vượt qua `validate_skill` với 0 lỗi.
   - **Quá khớp (Overfitting)**: Các skill được sinh ra ở dạng checklist quy trình chung (kiểm tra kiểu dữ liệu, định dạng cents, UTC, CHANGELOG), không bị overfit vào tên file cụ thể. Tuy nhiên, sự xuất hiện của các quy tắc mới ở tập đánh giá (`rule_version_bump`, `rule_source_line`) cho thấy tri thức rút ra từ tập học không thể bao quát các yêu cầu chưa biết trước.

6. **Đánh giá độ nhiễu thực nghiệm**:
   - So sánh điểm tác vụ học của cùng bộ skill trước đóng băng (`skills-auto-dev` ở S9) và sau đóng băng (`skills-auto` ở S11):
     - `skills-auto-dev` (S9): `code-learn` 1/10 (0.10), `data-learn` 0/8 (0.00, chạm recursion limit), `logs-learn` 1/9 (0.11) $\to$ Điểm trung bình = **0.070** (≈ **0.07**).
     - `skills-auto` (S11): `code-learn` 5/10 (0.50), `data-learn` 1/8 (0.125), `logs-learn` 0/9 (0.00) $\to$ Điểm trung bình = **0.208** (≈ **0.21**).
   - **Độ lệch do nhiễu**: Cùng một bộ skill, cùng môi trường và cùng `temperature=0`, nhiễu học giữa `skills-auto-dev` (**0.07**) so với `skills-auto` (**0.21**) lệch tới **0.14**, lớn hơn chênh lệch **0.11** giữa các điều kiện trên tác vụ đánh giá (0.20 so với 0.09).
   - **Kết luận**: **Không phân biệt được các điều kiện** (không có cơ sở khẳng định điều kiện nào thực sự vượt trội hơn do độ nhiễu ngẫu nhiên quá lớn).
   - **Cặp bằng chứng baseline**: Minh chứng rõ nét ở cặp chạy `baseline data-learn`: lần 1 chạy khói gặp đệ quy tiêu tốn **398k token và đạt 0/8 điểm** (`results_failed/baseline-data-learn-1`), trong khi lần 2 chạy chính thức kết thúc gọn gàng chỉ tốn **25.7k token và đạt 1/8 điểm** (`results/baseline/data-learn`). Sự biến thiên ngẫu nhiên nội tại trong cùng một cấu hình lớn hơn chênh lệch giữa các cấu hình khác nhau.

7. **Đối chiếu các giả thuyết H1, H2, H3 với kết quả thực nghiệm**:
   - **H1 (subagents so với baseline)**: **ĐÚNG MỘT PHẦN**. Điểm số đánh giá bằng baseline (`0.09` so với `0.09`), điểm học tăng nhẹ (`0.25` so với `0.21`, duy nhất từ `logs-learn`). Về chi phí token: trên tác vụ học tăng +46.4% (66,023 so với 45,102 token) đúng như dự đoán (+40% đến +50%), nhưng trên tác vụ đánh giá mức tiêu tốn token gần như không đổi (210,430 so với 211,466 token); tính trên cả 6 run, trung bình token chỉ tăng +7.7% dù trung vị tăng +62.1% $\to$ dự đoán "+40–50% token" chỉ đúng trên tác vụ học.
   - **H2 (skills-auto so với baseline)**: **ĐÚNG VỀ CƠ CHẾ, ĐIỂM SỐ BỊ NHIỄU**. Dự đoán tác tử không tự đọc skill được kiểm chứng chính xác (`skills_read = 0/6` ở mọi run, `skills_sha256` khớp hash đóng băng). Trên tác vụ học, điểm số bằng baseline (`0.21` so với `0.21`). Trên tác vụ đánh giá, điểm số ghi nhận `0.20` so với `0.09` nhưng chênh lệch này hoàn toàn không do skill (vì `skills_read = 0/6`) mà do tác tử ở `skills-auto data-eval` viết script Python nhiều dòng hợp lệ (đạt `3/9`), tránh được vòng lặp `SyntaxError` của lệnh `python -c` một dòng ở baseline và subagents.
   - **H3 (tác vụ học so với tác vụ đánh giá)**: **ĐÚNG**. Điểm số tác vụ đánh giá thấp hơn học ở baseline (`0.09` so với `0.21`) và subagents (`0.09` so với `0.25`), xấp xỉ bằng ở skills-auto (`0.20` so với `0.21`). Toàn bộ 100% check quy ước `rule_` trên tập đánh giá đều trượt (`0/12` trên cả 3 điều kiện); các quy tắc mới ở tập đánh giá (`rule_version_bump`, `rule_sorted_keys_format`, `rule_source_line`) hoàn toàn không thể chuyển giao từ tri thức tập học.

## 9. Hạn chế và tính hợp lệ

1. **Thử nghiệm chạy đơn lẻ và cỡ mẫu nhỏ (Sample size)**: Mỗi điều kiện chỉ được chạy 1 lần duy nhất trên 3 tác vụ học và 3 tác vụ đánh giá do giới hạn ngân sách tính toán. Cỡ mẫu nhỏ không cho phép thực hiện kiểm định ý nghĩa thống kê (p-value), khiến các kết luận so sánh chỉ mang tính quan sát định tính và dễ bị chi phối bởi các yếu tố may rủi.
2. **Độ biến thiên và nhiễu ngẫu nhiên rất lớn giữa các lần chạy (Run-to-run stochasticity)**: Mặc dù đặt `temperature=0`, mô hình vẫn thể hiện sự phân nhánh hành vi rõ rệt. Nhiễu học giữa `skills-auto-dev` (0.07) so với `skills-auto` (0.21) lệch tới 0.14, lớn hơn chênh lệch 0.11 giữa các điều kiện trên tác vụ đánh giá; kết hợp với cặp bằng chứng `baseline data-learn` (398k/0/8 so với 25.7k/1/8) dẫn tới kết luận không phân biệt được các điều kiện.
3. **Hiện tượng lặp công cụ và hạn chế của mô hình (`openai/gpt-4o-mini`)**: Cần tách biệt rõ các nguyên nhân: (a) Không có `pandas` trong container (đúng `pyproject.toml`) làm tác tử bế tắc ở `data-learn` của **cả 4 run** (baseline, subagents, skills-auto-dev, skills-auto đều gọi `import pandas`); nặng nhất ở `skills-auto-dev`, nơi tác tử lặp 54 lần lệnh có `import pandas` đến khi chạm `recursion_limit`; (b) Lần chạy tiêu tốn 1,292,867 token ở `skills-auto logs-eval` là vòng lặp thực thi 27 lệnh `execute` (+2 `read_file`) sinh lặp chuỗi log inline khổng lồ, hoàn toàn không liên quan đến `pandas`; (c) Lỗi cạn bước ở `data-eval` của baseline và subagents là do vòng lặp `SyntaxError` của lệnh `python -c` một dòng. Những hạn chế về định dạng lệnh và lặp mù quáng này khiến năng lực suy luận của mô hình bị che khuất bởi các lỗi công cụ lặp lại.
4. **Rào cản tự động kích hoạt tri thức thủ tục (`skills_read = 0/6`)**: Tác tử zero-shot không có cơ chế chủ động tìm kiếm và nạp các tệp skill vào ngữ cảnh làm việc trước khi hành động dù skill đã hiện diện trong sandbox (`skills_sha256` khớp hash đóng băng). Do `skills_read = 0/6` ở mọi run `skills-auto`, chênh lệch 0.20 so với 0.09 trên tác vụ đánh giá không thể quy cho skill.

## 10. Kết luận

1. Kiến trúc đa tác tử (`subagents`) không đem lại ưu thế vượt trội về hiệu năng so với đường cơ sở đơn tác tử (`0.25` so với `0.21` trên tập học, và cùng đạt `0.09` trên tập đánh giá), trong khi làm gia tăng chi phí token (+62.1% theo trung vị, +7.7% theo trung bình).
2. Ở mọi run của `skills-auto`, `skills_read = 0/6` dù `skills_sha256` khớp mã băm đóng băng (skill được nạp nhưng không được đọc), do đó chênh lệch 0.20 so với 0.09 trên tác vụ đánh giá không quy được cho skill mà do tác tử ở `skills-auto data-eval` định dạng script Python nhiều dòng hợp lệ nên tính toán được 3/9 check cơ bản, trong khi hai điều kiện kia kẹt vòng lặp `SyntaxError` của lệnh `python -c` một dòng.
3. Toàn bộ 100% check quy ước tổ chức (`rule_`) trên tác vụ đánh giá đều thất bại (`0/12`), chứng minh tri thức quy ước tự sinh từ tập học không có khả năng chuyển giao sang các yêu cầu mới chưa từng gặp.
4. Mức độ nhiễu thực nghiệm giữa `skills-auto-dev` (0.07) và `skills-auto` (0.21) trên tập học chênh lệch tới 0.14, lớn hơn chênh lệch 0.11 giữa các điều kiện trên tác vụ đánh giá, cùng với cặp bằng chứng `baseline data-learn` (398k/0/8 so với 25.7k/1/8), dẫn đến kết luận không phân biệt được sự vượt trội giữa các điều kiện.
5. Để hệ thống tác tử tự tiến hóa thực sự hiệu quả, cần bổ sung cơ chế bắt buộc nạp skill vào ngữ cảnh làm việc và cơ chế phát hiện ngắt sớm vòng lặp khi môi trường thiếu hụt thư viện.

## Phụ lục

- **Các lệnh đã chạy (theo thứ tự thực hiện)**:
  1. `docker build -t lab-deepagents .` và dựng image phái sinh `lab-deepagents-git` (bổ sung git và cấu hình safe.directory).
  2. `python scripts/tour.py` và kiểm tra chữ ký API `deepagents==0.7.21`.
  3. Kiểm tra kết nối LLM (V6) qua OpenRouter với mô hình `openai/gpt-4o-mini`.
  4. Cài đặt các mô-đun: `src/lab/subagents.py` (S1), `src/lab/agent.py` (S2), `src/lab/runner.py` (S3), `src/lab/curator.py` (S4).
  5. Chạy kiểm thử pytest: `docker run --rm -v "${PWD}:/lab" -w /lab lab-deepagents-git pytest` (đạt 29 passed).
  6. Khởi tạo báo cáo: `report/REPORT.md` (S5).
  7. Thay đổi thân hàm `run_task` trong `src/lab/runner.py` sang `agent.stream(..., stream_mode="values")` bọc trong `try...except` nhằm lưu giữ đầy đủ vết `trace.md` khi tác tử gặp ngoại lệ hoặc chạm trần đệ quy `GraphRecursionError`.
  8. Chạy S6: 6 lần chạy học tuần tự (`baseline data-learn`, `code-learn`, `logs-learn` và `subagents learn`). Lần chạy khói đầu tiên của data-learn gặp lỗi đệ quy 398k token được lưu trữ tại `results_failed/baseline-data-learn-1`.
  9. Chạy phân tích lỗi S7: `python scripts/check_breakdown.py`, cập nhật Mục 4 và Mục 5 của báo cáo.
  10. Chạy S8: Curator lần 1 sinh 3 skill (`data-validation`, `general-error-handling`, `testing-and-documentation`). Sau khi đối chiếu độ phủ quy tắc, đã sao lưu bộ 1 sang `results_failed/skills-run1/` và chạy curator lần 2. Do bộ lần 2 phủ ít rule hơn, đã xóa bộ 2 và khôi phục nguyên trạng bộ 1.
  11. Chạy S9: `python -m lab.runner --condition skills-auto --tasks learn`, ghi nhận kết quả và chuyển thư mục sang `results/skills-auto-dev`.
  12. Chạy S10: Soạn giả thuyết H1–H3 ở Mục 2 và bổ sung bằng chứng nhiễu ở Mục 9. Tạo commit và tag `freeze`.
  13. Chạy S11:
      - `python -m lab.runner --condition baseline --tasks eval`
      - `python -m lab.runner --condition subagents --tasks eval`
      - `python -m lab.runner --condition skills-auto --tasks all` (ghi nhận run 1.29M token vượt ngưỡng dừng 500k đã đặt, nhưng không dừng được vì chạy cả lô bằng `--tasks all`).
      - `python scripts/verify_freeze.py` (kết quả: `checked 6 runs of skill conditions: OK`).
  14. Chạy S12: `python -m lab.compare > report/table.md` và `python scripts/check_breakdown.py`, cập nhật Mục 7.
  15. Hoàn thiện báo cáo S13 (Mục 1, 7, 8, 9, 10 và Phụ lục).
  16. Thực hiện kiểm tra cuối S14.
- **Thay đổi runner sang stream và lý do**:
  - Trong `src/lab/runner.py`, phương thức `agent.invoke(...)` ban đầu sẽ làm mất toàn bộ danh sách `messages` khi gặp ngoại lệ `GraphRecursionError`, dẫn tới `trace.md` bị rỗng.
  - Đã chuyển đổi sang `agent.stream(..., stream_mode="values")` với khối `try...except Exception as e: record["error"] = ...`. Nhờ đó, ngay cả khi tác tử chạm giới hạn bước đệ quy, toàn bộ các tin nhắn và tool call đã thực hiện đều được lưu lại đầy đủ trong `trace.md` để phục vụ phân loại lỗi.
- **Nội dung thư mục `results_failed/`**:
  - `results_failed/baseline-data-learn-1/`: Lưu vết lần chạy khói đầu tiên của `baseline data-learn` bị lặp vô tận (tiêu tốn 398,087 token, chạm limit 60, điểm 0/8).
  - `results_failed/skills-run1/`: Lưu bản sao lưu 3 skill nguyên bản do curator sinh ra ở lần 1 trước khi chạy thử nghiệm curator lần 2, sau đó được dùng để khôi phục nguyên trạng về `skills/auto/`.
- **Ghi chú về mức trần token ở S11**:
  - Lần chạy `skills-auto logs-eval` tiêu tốn 1,292,867 token (run 1.29M token), vượt ngưỡng dừng 500,000 token đã đặt trong quy tắc, nhưng không dừng được vì chạy cả lô bằng `--tasks all` (lệnh thực thi tuần tự tự động cả 6 tác vụ liên tiếp trong cùng một tiến trình container). Toàn bộ dữ liệu vết và số liệu token được giữ nguyên đầy đủ để phân tích minh bạch.

