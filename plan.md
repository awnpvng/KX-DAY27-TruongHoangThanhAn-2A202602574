# Plan — hoàn thành Day27 HITL Lab (KX-DAY27-TruongHoangThanhAn-2A202602574)

Môi trường: Windows, terminal PowerShell.

## 0. Kiểm tra trạng thái ban đầu

- [X] Tên repo đúng mẫu `KX-DAY27-HoVaTen-MSSV` → `KX-DAY27-TruongHoangThanhAn-2A202602574`.
- [X] `origin` đã trỏ đúng về `https://github.com/awnpvng/KX-DAY27-TruongHoangThanhAn-2A202602574.git`.
- [X] `.gitignore` đã có `.env` và `*.db` (loại `hitl_audit.db`).
- [X] Xác nhận repo đang **Public** trên GitHub (Settings → General → Danger Zone / Visibility).

## 1. [X] Cài đặt môi trường (uv)

```powershell
uv sync
```

Kiểm tra Python 3.11+ và `uv` đã có trong PATH (`uv --version`).

## 2. [X] Thiết lập `.env`

- [X] `OPENROUTER_API_KEY` đã điền.
- [X] `GITHUB_TOKEN` đã điền, scope `repo` xác nhận qua header `X-OAuth-Scopes: repo`.
- [X] `GITHUB_USER` đã điền (`awnpvng`).
- [X] `VinUni-AI20k/PR-Demo` nay đã **public** và truy cập được (trước đó từng 404 do repo private/thiếu quyền — đã tự khắc phục phía giảng viên).
- [X] `OPENROUTER_API_KEY` đầu tiên hết credit (lỗi 402) → đã đổi key mới, verify bằng 1 request `curl` thật tới OpenRouter → trả lời "OK!" thành công.

Xác minh token (PowerShell):

```powershell
$env:GITHUB_TOKEN = (Get-Content .env | Select-String '^GITHUB_TOKEN=').ToString().Split('=')[1]
curl.exe -H "Authorization: Bearer $env:GITHUB_TOKEN" https://api.github.com/repos/VinUni-AI20k/PR-Demo | Select-String full_name
```

## 3. [X] Exercise 1 — Confidence routing (`exercises/exercise_1_confidence.py`)

Đã code xong và **verify chạy thật**: PR #1 → `auto_approve` (confidence 75%), PR #2 → `escalate` (confidence 55%) — hai nhánh khác nhau, đúng checkpoint. Đã tăng cường system prompt (yêu cầu model đánh giá rủi ro bảo mật nghiêm ngặt hơn khi tính confidence) vì lần chạy đầu model quá tự tin (85% cho cả 2 PR, không phân biệt được).

Việc cần làm (đã hoàn thành):

- `node_analyze`: gọi `get_llm().with_structured_output(PRAnalysis)` với system/user message chứa title + diff, trả `{"analysis": analysis}`.
- `node_route`: so sánh `state["analysis"].confidence` với `AUTO_APPROVE_THRESHOLD` / `ESCALATE_THRESHOLD`, trả `{"decision": ...}`.
- `build_graph`: `add_node` cho 6 node, `add_edge(START→fetch_pr→analyze→route)`, `add_conditional_edges("route", lambda s: s["decision"], {...})`, `add_edge` từ 3 terminal node → `END`.

Chạy thử:

```powershell
uv run python exercises/exercise_1_confidence.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
uv run python exercises/exercise_1_confidence.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/2
```

Tiêu chí đạt: hai PR in ra hai `decision` khác nhau.

## 4. [X] Exercise 2 — HITL với `interrupt()` (`exercises/exercise_2_hitl.py`)

Đã code xong và **verify chạy thật**: approve trên PR #1 → đăng comment thật lên GitHub thành công, `final_action = committed`. Riêng cơ chế `interrupt()`/`Command(resume=...)` ở nhánh `human_approval` (confidence 58–72%) được verify thêm bằng 1 script mock (giả lập `get_llm`/`fetch_pr`/`post_review_comment`, ép confidence=0.65) để không phụ thuộc may rủi của LLM thật — graph dừng đúng payload, resume đúng, gọi `post_review_comment` đúng khi approve.

Việc cần làm (đã hoàn thành):

- `node_human_approval`: gọi `interrupt({...})` với payload (`kind`, `confidence`, `confidence_reasoning`, `summary`, `comments`, `diff_preview`), lấy `response = interrupt(...)`, trả `{"human_choice": response["choice"], "human_feedback": response.get("feedback")}`.
- `build_graph`: compile với `checkpointer=MemorySaver()`.
- `main()`: thêm vòng lặp resume
  ```python
  while "__interrupt__" in result:
      payload = result["__interrupt__"][0].value
      answer = prompt_human(payload)
      result = app.invoke(Command(resume=answer), cfg)
  ```

Chạy thử:

```powershell
uv run python exercises/exercise_2_hitl.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
```

Kỳ vọng: terminal dừng lại, hiện panel xanh, chờ approve/reject/edit.

## 5. [X] Exercise 3 — Escalation Q&A (`exercises/exercise_3_escalation.py`)

Đã code xong và **verify chạy thật** với PR #2: model sinh đúng 4 câu hỏi cụ thể bám sát diff (vd. "rationale cho MD5 trong `auth.py`?", "hard-coded user_id trong `sync.py`?", "SQL string interpolation trong `storage.py`?"), nhận câu trả lời, tổng hợp lại review (confidence 55% → 75%), đăng comment thật lên PR #2 với `final_action = committed_after_escalation`.

Việc cần làm (đã hoàn thành):

- Thêm instruction vào system prompt của `node_analyze`: nếu confidence < 60% thì điền `escalation_questions` (2–4 câu cụ thể, tham chiếu file/section trong diff).
- `node_escalate`: gọi `interrupt({"kind": "escalation", "pr_url":..., "confidence":..., "confidence_reasoning":..., "summary":..., "risk_factors":..., "questions": questions})`, trả `{"escalation_answers": answers}`.
- `node_synthesize`: build prompt chứa diff gốc + phân tích ban đầu + Q&A, gọi lại LLM structured output, trả `{"analysis": refined}`.
- `build_graph`: nối `escalate → synthesize → commit` (commit đã nối → END).

Chạy thử:

```powershell
uv run python exercises/exercise_3_escalation.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/2
```

Kỳ vọng: panel vàng hiện câu hỏi, nhập câu trả lời, agent tổng hợp lại review.

## 6. [X] Exercise 4 — SQLite audit trail (`exercises/exercise_4_audit.py`)

Đã code xong toàn bộ 10 TODO và **verify chạy thật** với PR #1 (nhánh `auto_approve`): `audit.replay --thread <id>` in đúng 4 dòng `fetch_pr → analyze → route → auto_approve` với `conf=`, `risk=` (màu đúng), `decision=`, `execution_time_ms` hợp lệ; `audit.replay --list` hiện đúng bảng các phiên gần đây kèm `worst_risk`/`events`.

Việc cần làm (đã hoàn thành, 10 TODO):

- `audit()`: `await write_audit_event(thread_id=state["thread_id"], pr_url=state["pr_url"], entry=entry)`.
- Thêm 1 `AuditEntry` cho mỗi node: `analyze`, `route`, `commit`, `auto_approve`, `synthesize`.
- Thêm 2 `AuditEntry` cho `human_approval` và `escalate` (trước và sau `interrupt()`), dùng `risk_level_for(confidence)`, `execution_time_ms=int((time.monotonic()-t0)*1000)`, `reviewer_id=os.environ.get("GITHUB_USER")` ở bước sau interrupt.
- Dùng node `node_fetch_pr` có sẵn làm mẫu tham chiếu field.

Chạy thử:

```powershell
uv run python exercises/exercise_4_audit.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
uv run python -m audit.replay --thread <thread_id-in-ra>
uv run python -m audit.replay --list
```

## 7. [X] Exercise 5 — Streamlit UI (`app.py`)

Đã hoàn thành:

- Thêm `exercises/__init__.py` (rỗng) để `app.py` import được `from exercises.exercise_4_audit import build_graph`.
- Thêm hàm `recent_threads()` vào `audit/replay.py` (tái dùng query của `list_threads`, trả `list[dict]`) — sidebar "Recent sessions" gọi hàm này, mỗi dòng là 1 nút bấm để load lại `thread_id` cũ.
- `run_graph()`: build graph với checkpointer `cp`, gọi `ainvoke` ban đầu hoặc `ainvoke(Command(resume=...))` khi resume.
- `render_approval_card`: 3 nút Approve/Reject/Edit, mỗi nút trả dict `{"choice": ..., "feedback": ...}`.
- `render_escalation_card`: 1 `st.text_input` cho mỗi câu hỏi trong `payload["questions"]`, submit trả dict `{question: answer}`.
- `streamlit` đã có sẵn trong `pyproject.toml` (không cần thêm).

Đã **verify**: `uv run streamlit run app.py --server.headless true` khởi động server thành công, trang chính trả về HTTP 200 (không lỗi import/runtime). Sidebar "Recent sessions" dùng chung query đã verify qua `audit.replay --list`. Bạn nên tự mở UI 1 lần để test tương tác (bấm nút Approve/Reject/Edit, điền form escalation):

```powershell
uv run streamlit run app.py
```

## 8. [X] Dọn dẹp trước khi nộp

```powershell
git status
```

- [X] Đã verify `git status` KHÔNG liệt kê `.env` hay `hitl_audit.db` (đã bị `.gitignore` loại, kể cả sau khi exercise 4 tạo ra `hitl_audit.db` lúc test).
- [X] Đã verify không còn `raise NotImplementedError` thật trong `exercises/*.py` và `app.py` (chỉ còn 1 dòng docstring mô tả trong `exercise_4_audit.py`, không phải code).
- Xoá file `hitl_audit.db` cục bộ nếu muốn test lại từ đầu: `Remove-Item hitl_audit.db -ErrorAction SilentlyContinue`.

## 9. [ ] Commit & push

```powershell
git add .
git commit -m "Day27 HITL lab solution"
git push origin main
```

## 10. [ ] Kiểm tra cuối (trước khi nộp link)

- [X] Đã chạy thật đủ 4 exercise với PR #1/#2 — xác nhận routing (`auto_approve`/`escalate`), HITL pause (verify bằng mock test), audit trail ghi đúng.
- [X] Đã verify `uv run streamlit run app.py` khởi động và serve trang thành công (HTTP 200).
- [X] Đã tự đăng ít nhất 3 comment review thật lên PR #1 và PR #2 của `VinUni-AI20k/PR-Demo` dưới username `awnpvng` trong lúc test (thoả điều kiện "Comment review ... hiện dưới username GitHub của bạn").
- [ ] Mở repo trong cửa sổ ẩn danh (chưa đăng nhập GitHub) → xác nhận Public, thấy đủ code (không còn khung TODO).
- [ ] Clone sạch sang thư mục khác, chạy lại toàn bộ lệnh từ bước 1–7 để đảm bảo reproducible.
- [ ] Nộp link repo: `https://github.com/awnpvng/KX-DAY27-TruongHoangThanhAn-2A202602574`.

## Lưu ý an toàn

- Nếu `git status` từng liệt kê `.env`, phải coi token đã lộ và thu hồi (revoke) ngay tại https://github.com/settings/tokens.
- Không hard-code API key/token trực tiếp trong code — luôn đọc qua `.env` + `dotenv`.
