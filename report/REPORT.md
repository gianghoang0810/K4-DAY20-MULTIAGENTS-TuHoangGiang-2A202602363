# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Tu Hoang Giang | 2A202602363 | 100% (cá nhân) |

- Mô hình (tên deployment hoặc `LAB_MODEL`), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `openai/gpt-4o-mini` (OpenRouter), `LAB_TEMPERATURE=0`, `recursion_limit=60`
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: `deepagents==0.7.21`, Windows 10, chạy trong Docker container (`python:3.12-slim` + git)
- Số lần chạy tác vụ đã dùng / ngân sách: 10 lần chạy tác vụ học (3 baseline + 3 subagents + 3 skills-auto + 1 lần smoke test ban đầu) / ngân sách 21 lần chạy
- Commit của tag `freeze`: (chưa freeze - dừng tại cổng G2)

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
  - `data-learn`: 1 cuộc gọi (tác tử chính gọi `implementer` khi lệnh Python với thư viện pandas bị lỗi `ModuleNotFoundError`). Lời gọi này đã giúp thực hiện tính toán qua shell script và vượt qua check `duplicate_rows_removed`.
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
| `data-validation` | Tổng quát | Đúng (hướng dẫn kiểm tra định dạng dữ liệu đầu vào/ra, xử lý missing/invalid, trùng lặp, tiền tệ ở dạng cents) | 11 dòng, trigger: "Use when you need to validate data inputs and outputs to ensure correctness and compliance with specifications." Đo ở S9: `skills_read=0` (0/8 pass, 368,593 tokens, GraphRecursionError). Tác tử không chủ động đọc skill, không áp dụng checklist về cents và deduplication. |
| `general-error-handling` | Tổng quát | Đúng (hướng dẫn format log UTC, lọc cấp độ ERROR/CRITICAL, kiểm tra schema output, đếm lặp) | 11 dòng, trigger: "Use when you need to ensure that error handling and logging conventions are followed in your code." Đo ở S9: `skills_read=0` (1/9 pass, 22,047 tokens). Tác tử không đọc skill, chỉ tự đạt check cấu trúc cơ bản, bỏ lỡ toàn bộ 3 check house rules do không nạp schema. |
| `testing-and-documentation` | Tổng quát | Đúng (hướng dẫn viết unit test, docstring, cập nhật CHANGELOG.md, thêm regression test) | 11 dòng, trigger: "Use when you need to ensure that your code is well-tested and documented according to project standards." Đo ở S9: `skills_read=0` (1/10 pass, 18,466 tokens). Tác tử không đọc skill, không tạo file `test_regressions.py` hay cập nhật `CHANGELOG.md`. |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

```text
(dán bảng ở đây)
```

## 8. Phân tích

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Nhóm đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

1. **Độ biến thiên và tính ngẫu nhiên lớn giữa các lần chạy (Run-to-run stochasticity / noise)**: Mặc dù cấu hình mô hình đặt `temperature=0`, hệ thống tác tử vẫn thể hiện tính bất định rất cao. Minh chứng rõ rệt nhất là ở tác vụ `baseline data-learn`:
   - Lần chạy đầu (được lưu tại `results_failed/baseline-data-learn-1`): Tác tử rơi vào vòng lặp bash/python cố chấp, tiêu tốn tới **398,087 token**, chạm trần `GraphRecursionError` (limit 60) và đạt điểm **0/8**.
   - Lần chạy lại (tại `results/baseline/data-learn`): Tác tử sau khi gặp lỗi thiếu thư viện `pandas` đã dừng thử và ghi file JSON mẫu sau 5 tool calls, chỉ tiêu tốn **25,710 token** (chênh lệch hơn 15 lần) và kết thúc hợp lệ với điểm **1/8**.
   Hiện tượng này cũng lặp lại ở `skills-auto data-learn` (tiêu tốn **368,593 token** và chạm đệ quy), cho thấy độ nhiễu lớn này là một hạn chế cốt lõi cần lưu ý khi diễn giải các chênh lệch điểm số giữa các điều kiện.
2. **Kích thước mẫu thử nghiệm hạn chế**: Thí nghiệm chỉ tiến hành trên 3 tác vụ học và 3 tác vụ đánh giá với 1 lần chạy cho mỗi cấu hình (do ngân sách tài nguyên và token). Do cỡ mẫu nhỏ, một số check đạt có thể do phán đoán ngẫu nhiên (ví dụ đoán "North" trúng `top_region`) thay vì phản ánh năng lực hệ thống nhất quán.
3. **Rào cản tự động kích hoạt tri thức thủ tục (Skill retrieval barrier)**: Dù các skill tự sinh được curator cấu trúc chuẩn và lưu đúng thư mục quy định, tác tử zero-shot không có cơ chế bắt buộc phải tìm kiếm hay nạp skill vào bộ nhớ trước khi giải quyết bài toán (`skills_read = 0` ở toàn bộ 3 tác vụ của S9). Điều này khiến tri thức tự tiến hóa chưa được chuyển giao hiệu quả vào hành động thực tế của tác tử nếu không có chỉ dẫn prompting cưỡng bức.

## 10. Kết luận

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
