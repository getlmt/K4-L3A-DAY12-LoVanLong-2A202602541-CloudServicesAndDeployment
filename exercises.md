# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

| Họ và tên | Mã học viên |
|-----------|-------------|
| Lò Văn Long | 2A202602541 |

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: lúc deploy lên Railway mình tạo service `agent` nhưng giả sử quên
> chạy `railway variables --set AGENT_API_KEY=...`.
>
> - **Có mặc định `"changeme"`:** app vẫn khởi động, `/health` trả 200, Railway
>   báo "Deployment successful" nên mình tưởng mọi thứ ổn. Nhưng URL là public,
>   ai đoán được (hoặc đọc được trong repo công khai) chữ `changeme` là gọi
>   `/ask` thoải mái, tiêu ngân sách LLM của mình. Mình chỉ phát hiện khi nhìn
>   hóa đơn hoặc log lạ — tức là sau khi thiệt hại đã xảy ra.
> - **Không có mặc định (fail fast):** pydantic ném `ValidationError` ngay lúc
>   `Settings()` được tạo, container chết, deploy báo lỗi và log chỉ rõ thiếu
>   `agent_api_key`. Mình biết ngay trong vài giây, sửa bằng cách set biến rồi
>   deploy lại, chưa có request nào lọt vào.
>
> Nói gọn: lỗi cấu hình nên lộ ra lúc deploy (khi mình đang nhìn), không nên
> lộ ra lúc có người lạ đang dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật khi chạy `docker compose up` rồi gọi `/ask` (lần thứ hai
> của user `smoke`):
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:13:34.685669+00:00", "user_id": "smoke", "tokens_in": 44, "tokens_out": 46, "cost_usd": 3.42e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và tìm kiếm theo trường:** ví dụ lọc `event = "ask_completed" AND
>    user_id = "smoke"` để xem một user đã hỏi những lúc nào. Với chuỗi tự do
>    thì không có `user_id` để lọc, chỉ grep mò được.
> 2. **Cộng dồn / vẽ biểu đồ / cảnh báo:** tính tổng `cost_usd` theo ngày hoặc
>    theo user, đếm số request mỗi phút, đặt cảnh báo khi `level = "error"` tăng
>    đột biến. Mình thấy rõ điều này trên Railway: log JSON của app được
>    Railway tự tách thành các trường `event=... service=... version=...`.
>
> Thêm nữa, `timestamp` theo chuẩn ISO-8601 UTC nên ghép log của nhiều container
> lại vẫn sắp xếp đúng thứ tự thời gian.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật từ `docker images`: `agent:single 1.73GB`, `agent:multi 310MB` —
> nhỏ hơn khoảng 5.6 lần. (Bản 1 stage build từ Dockerfile gốc ở commit đầu
> tiên của repo.)
>
> Xem `docker history` thì phần thư viện Python gần như bằng nhau ở hai bản
> (layer `pip install` ~95MB ở bản 1 stage, layer `COPY /opt/venv` ~96MB ở bản
> multi-stage). Vậy **gần như toàn bộ ~1.4GB chênh lệch đến từ base image**:
>
> - `python:3.11` bản đầy đủ chứa sẵn bộ công cụ build: có `gcc 14.2` (mình
>   chạy `gcc --version` trong image 1 stage thì có, trong image multi-stage thì
>   không có), `/usr/lib/gcc` ~26MB, header C `/usr/include` ~57MB, tài liệu
>   `/usr/share/doc` ~58MB, cộng thêm git, các thư viện `-dev`, công cụ hệ thống…
> - `python:3.11-slim` bỏ hết những thứ đó, chỉ giữ đủ để *chạy* Python.
>
> Multi-stage giúp giữ image runtime gọn: nếu thư viện nào cần compiler để cài,
> việc cài diễn ra ở stage `builder`, runtime chỉ copy kết quả (`/opt/venv`)
> sang. Image nhỏ hơn thì pull/deploy nhanh hơn và ít phần mềm hơn cũng là ít
> lỗ hổng bảo mật hơn (không có gcc sẵn để kẻ tấn công biên dịch mã độc).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thử trên một bản sao của repo: build một lần, sửa 1 ký tự trong
> `app/main.py` (`SERVICE_VERSION` từ `"1.0.0"` thành `"1.0.1"`), rồi build lại
> với `--progress=plain`.
>
> **Với Dockerfile của mình** — build lại mất **2 giây**:
>
> | Layer | Kết quả |
> |---|---|
> | `builder`: `FROM python:3.11-slim`, `RUN python -m venv`, `COPY requirements.txt`, `RUN pip install` | CACHED |
> | `runtime`: `RUN groupadd/useradd`, `WORKDIR /app`, `COPY --from=builder /opt/venv` | CACHED |
> | `COPY app ./app` | **chạy lại** (nội dung `app/` đổi) |
> | `COPY utils ./utils` | **chạy lại** (layer phía trên đổi thì mọi layer sau nó phải build lại, dù `utils/` không đổi) |
>
> Layer `pip install` chỉ phụ thuộc `requirements.txt`, mà file này không đổi
> nên Docker dùng lại cache — không phải tải lại thư viện.
>
> **Nếu đặt `COPY . .` trước `RUN pip install`** (mình viết một Dockerfile thử
> kiểu này, sửa 1 ký tự rồi build lại) — mất **34 giây**:
>
> | Layer | Kết quả |
> |---|---|
> | `FROM`, `WORKDIR /app` | CACHED |
> | `COPY . .` | **chạy lại** (có file đổi) |
> | `RUN pip install -r requirements.txt` | **chạy lại**, tải và cài lại toàn bộ thư viện (`Successfully installed PyYAML… fastapi… pydantic…`) |
>
> Docker quyết định dùng cache theo nội dung của các file được `COPY` vào
> layer đó. `COPY . .` chứa cả code, nên sửa 1 ký tự code là cache của layer
> này mất, kéo theo `pip install` phía sau cũng phải chạy lại. 34 giây là khi
> mạng tốt; lúc mình build, mạng tới pypi chậm và bị timeout nên có lần mất
> hơn 2 phút. Trên CI, mỗi lần push đều phải chịu thời gian này.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. Code Python có lỗ hổng, ví dụ một thư viện bị lỗi cho phép thực thi lệnh
>    từ xa (RCE), hoặc mình vô tình dùng `eval`/`subprocess` với dữ liệu người
>    dùng gửi lên.
> 2. Kẻ tấn công chạy được lệnh shell bên trong container, **với quyền của
>    process app** — ở đây là root (UID 0).
> 3. Là root nên trong container họ làm được mọi thứ: đọc biến môi trường
>    (`AGENT_API_KEY`, `REDIS_URL` có mật khẩu), sửa code app, cài công cụ.
> 4. Root trong container cũng là UID 0 trên host (nếu không bật user
>    namespace). Chỉ cần thêm một cấu hình sai — mount volume của host, mount
>    `/var/run/docker.sock`, chạy `--privileged`, hoặc một lỗ hổng kernel/runtime
>    để "thoát container" — là họ có quyền root trên máy host, ảnh hưởng mọi
>    container khác trên máy đó.
>
> Lệnh `USER app` cắt chuỗi ở **bước 2**: process app chạy bằng user thường
> (mình kiểm tra bằng `docker compose exec agent whoami` → `app`). Kẻ tấn công
> chiếm được app cũng chỉ có quyền của user `app`: không cài được gói hệ thống,
> không sửa được file của root, và nếu có thoát ra host thì cũng chỉ là một UID
> không có đặc quyền. Họ phải tìm thêm một lỗ hổng leo thang quyền nữa — khó
> hơn nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**, gấp đôi hạn mức.
>
> Cách đạt được: với cửa sổ cố định theo phút đồng hồ, bộ đếm reset về 0 đúng
> giây `:00`.
>
> - Lúc `10:00:59` gửi 10 request → hết quota của phút `10:00`.
> - Lúc `10:01:00` bộ đếm reset, gửi tiếp 10 request → hết quota của phút
>   `10:01`.
>
> 20 request nằm gọn trong khoảng ~1–2 giây mà vẫn "đúng luật".
>
> Với sliding window, lúc `10:01:00` app đếm các request trong 60 giây **gần
> nhất** (từ `10:00:00` tới `10:01:00`), vẫn thấy 10 request lúc `10:00:59`
> → request thứ 11 bị 429. Mình quan sát đúng hành vi này trên Railway: gọi
> 15 lần liên tiếp được `200` × 10 rồi `429` × 5, và phải đợi hết 60 giây mới
> gọi lại được `200`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau ở **đơn vị đo và khung thời gian**:
>
> | | Rate limit | Cost guard |
> |---|---|---|
> | Đếm cái gì | Số request | Số tiền (USD) |
> | Khung thời gian | 60 giây gần nhất (trượt) | Cả tháng (key `cost:<user>:<YYYY-MM>`) |
> | Mã lỗi | 429 Too Many Requests | 402 Payment Required |
> | Bảo vệ khỏi | Spam, bot, quá tải server | Hóa đơn LLM vượt ngân sách |
>
> - **Rate limit cho qua nhưng cost guard chặn:** một user gửi đều đặn 5
>   request/phút (dưới hạn mức 10), nhưng mỗi request là một câu hỏi rất dài
>   kèm lịch sử 20 message, tốn nhiều token. Suốt nhiều ngày như vậy, tổng chi
>   phí tháng chạm `MONTHLY_BUDGET_USD = 10.0` → request tiếp theo bị 402 dù tốc
>   độ gọi hoàn toàn bình thường.
> - **Cost guard cho qua nhưng rate limit chặn:** đầu tháng, user mới hầu như
>   chưa tiêu gì, nhưng một script lỗi gửi 15 request ngắn "test" trong vài
>   giây. Chi phí mỗi request chỉ ~0.00002 USD, ngân sách còn gần nguyên,
>   nhưng từ request thứ 11 bị 429 (đúng như mình thấy khi test trên Railway).
>
> Vì vậy cần cả hai: rate limit chặn "gọi quá nhanh", cost guard chặn "tiêu quá
> nhiều" — tích lũy chậm thì rate limit không bắt được.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Giả sử endpoint gộp được dùng làm **liveness probe** (orchestrator dựa vào nó
> để quyết định restart), kiểm tra mỗi ~10 giây, restart sau vài lần fail liên
> tiếp:
>
> 1. **t = 0s:** Redis mất kết nối. Cả 3 container vẫn sống bình thường, chỉ
>    là không gọi được Redis.
> 2. **Lần probe tiếp theo:** cả 3 container cùng trả 503 vì cùng phụ thuộc
>    một Redis — lỗi xảy ra **đồng loạt**, không phải một container hỏng.
> 3. **Sau vài lần fail (~20–30s):** orchestrator kết luận cả 3 "đã chết" và
>    **restart cả 3 cùng lúc**. Load balancer không còn instance nào → mọi
>    request (kể cả `/health` và những request không cần Redis) nhận 502/503.
>    Request đang xử lý dở bị cắt.
> 4. **t ≈ 30s:** Redis sống lại, nhưng các container đang trong quá trình
>    khởi động lại nên chưa phục vụ được.
> 5. **Sau đó:** các container khởi động xong, probe xanh trở lại. Thời gian
>    sập thực tế **dài hơn 30 giây** của Redis, và restart không sửa được gì
>    vì lỗi nằm ở Redis chứ không nằm ở container. Nếu Redis chập chờn lâu
>    hơn, cụm rơi vào vòng lặp restart liên tục.
>
> Khi tách ra như bài làm: `/health` không chạm Redis → vẫn 200, không container
> nào bị restart. `/ready` trả 503 `{"status": "not ready", "redis": false}` →
> load balancer tạm ngừng đẩy traffic vào. Redis sống lại → `/ready` về 200 →
> traffic quay lại ngay, không mất thời gian khởi động lại, process và kết nối
> vẫn giữ nguyên.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy 3 container `agent` phía sau Nginx (`nginx/nginx.conf`, round-robin)
> rồi gọi `/ask` 6 lần với cùng một `X-User-Id`. Kết quả thật, container xử lý
> lấy từ log `ask_completed` của từng container:
>
> | Request | Container xử lý | `history_length` |
> |---|---|---|
> | 1 | agent-3 | 0 |
> | 2 | agent-1 | 2 |
> | 3 | agent-2 | 4 |
> | 4 | agent-3 | 6 |
> | 5 | agent-1 | 8 |
> | 6 | agent-2 | 10 |
>
> Mỗi request rơi vào một container khác nhau mà `history_length` vẫn tăng đều
> 0 → 2 → 4 → … → 10, vì cả 3 container cùng đọc/ghi một Redis.
>
> **Nếu lưu trong dict Python:** mỗi container có RAM và dict riêng, chỉ thấy
> những lượt mà chính nó xử lý. Với thứ tự round-robin trên, con số sẽ là
> **0, 0, 0, 2, 2, 2**: 3 request đầu rơi vào 3 container "trắng", mỗi
> container mới chỉ nhớ đúng 1 lượt hỏi-đáp (2 message) của riêng nó. Agent
> "mất trí nhớ" ngẫu nhiên tùy request rơi vào đâu, và khi container restart
> thì lịch sử trong dict mất sạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp thật:** sau khi deploy lên Railway, `/health` và `/ready` đều 200,
> nhưng khi chạy lệnh kiểm tra `/ask` có API key trong `DEPLOYMENT.md` với câu
> hỏi `"Deploy là gì?"` thì nhận:
>
> ```
> HTTP/1.1 400 Bad Request
> {"detail":"There was an error parsing the body"}
> ```
>
> **Tìm nguyên nhân:**
> - Cùng lúc đó, vòng lặp rate limit gọi `/ask` với body `{"question":"test"}`
>   lại trả `200` bình thường → key đúng, app và Redis trên cloud không lỗi.
> - Khác biệt duy nhất là câu hỏi có dấu tiếng Việt. Lỗi là "parsing the body"
>   → FastAPI không đọc được JSON, tức là byte gửi lên không phải UTF-8 hợp lệ.
>   Nguyên nhân: curl chạy trong Git Bash trên Windows chuyển chuỗi trong tham
>   số `-d` sang bảng mã của Windows trước khi gửi, nên chữ `là gì` bị hỏng.
>
> **Cách sửa:** ghi body ra một file được lưu bằng UTF-8, rồi gửi bằng
> `--data-binary @body.json` (curl gửi nguyên byte trong file, không chuyển mã).
> Chạy lại → `200 OK` kèm câu trả lời tiếng Việt đúng.
>
> **Một lỗi phòng trước được:** `railway.toml` ban đầu có
> `startCommand = "uvicorn ... --port $PORT"`. Theo tài liệu Railway, khi build
> bằng Dockerfile lệnh này có thể được chạy trực tiếp, không qua shell, nên
> `$PORT` có nguy cơ không được thay bằng số cổng thật (mình không deploy thử
> bản cũ nên không có log lỗi của trường hợp này). Mình bỏ `startCommand` để Railway dùng
> `CMD` trong Dockerfile (chạy qua `sh -c` nên có thay `$PORT`). Log trên
> Railway xác nhận app chạy ở `Uvicorn running on http://0.0.0.0:8080` — cổng
> 8080 do Railway gán, không phải 8000 mặc định.
