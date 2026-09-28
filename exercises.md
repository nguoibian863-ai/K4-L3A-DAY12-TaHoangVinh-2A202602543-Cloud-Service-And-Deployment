# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tạ Hoàng Vinh  Mã học viên: 2A202602543

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy service lên cloud (như Railway hoặc Render), nếu tôi sơ suất quên khai báo biến môi trường `AGENT_API_KEY` trong phần cấu hình dashboard:
- Nếu để giá trị mặc định là `"changeme"`: Service vẫn khởi động thành công và báo trạng thái Healthy. Khi đó, các bot quét tự động hoặc bất kỳ ai cũng có thể gọi API bằng khóa `"changeme"` để thực hiện hàng nghìn request LLM. Hậu quả là tài khoản bị trừ cạn kiệt ngân sách hoặc lộ dữ liệu nhạy cảm mà tôi chỉ phát hiện ra khi nhận hóa đơn thanh toán.
- Khi không để giá trị mặc định (fail fast): Service sẽ lập tức crash ngay lúc container khởi động với lỗi `pydantic_core._pydantic_core.ValidationError: Field required`. Việc service "chết sớm" này khiến quá trình deploy báo đỏ ngay trước mắt tôi trên màn hình dashboard, buộc tôi phải bổ sung API key chính xác trước khi service có thể nhận traffic công khai từ Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:43:38.868205+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.000114}`

Hai việc làm được với định dạng log JSON này mà `print("đã trả lời xong")` không thể làm được:
1. **Truy vấn, lọc và thống kê định lượng tự động bằng công cụ log aggregator (Datadog, Loki, CloudWatch):** Máy tính có thể parse JSON để tính tổng chi phí theo ngày/tháng (`sum(cost_usd)`), tìm top người dùng tiêu tốn tài nguyên nhất (`group by user_id`), hoặc thống kê lượng token tiêu thụ theo thời gian thực mà không phải viết regex mong manh để bóc tách chuỗi.
2. **Cấu hình cảnh báo tự động (alerting) dựa trên ngưỡng:** Có thể thiết lập quy tắc tự động gửi tin nhắn cảnh báo tới Slack/Telegram khi phát hiện request có `cost_usd > 0.05` bất thường hoặc khi tỷ lệ event lỗi trong cửa sổ 5 phút vượt ngưỡng cho phép, điều hoàn toàn bất khả thi với log text tự do không cấu trúc.

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
| 1 stage (bản đầu) | ~1015 MB |
| Multi-stage | ~189 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~826 MB) bao gồm:
1. **Trình biên dịch và bộ công cụ xây dựng mã nguồn:** Bản base `python:3.11` đầy đủ chứa `gcc`, `g++`, `make`, `build-essential`, các header file C/C++ (`/usr/include`) dùng để biên dịch các package native (C-extensions). Sang multi-stage, stage builder cài đặt xong thì toàn bộ compiler toolchain này bị loại bỏ, stage runtime `python:3.11-slim` chỉ kế thừa file thư viện đã biên dịch sẵn trong `/usr/local`.
2. **Các gói tiện ích và bộ nhớ đệm dư thừa:** Image đầy đủ chứa các tiện ích hệ điều hành Debian không cần thiết cho runtime (như manual pages, documentation, các package quản lý hệ thống) và cache của pip (`~/.cache/pip`). Multi-stage dùng base `slim` kết hợp `--no-cache-dir` giúp loại bỏ toàn bộ phần thừa thãi này, mang lại image siêu gọn nhẹ và bảo mật hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Docker kiểm tra mã băm checksum của từng layer. Do `requirements.txt` không đổi nên các layer `COPY requirements.txt .` và `RUN pip install ...` ở stage builder đều được tái sử dụng hoàn toàn từ cache (`CACHED`). Ở stage runtime, layer `COPY --from=builder` cũng được cache. Chỉ từ layer `COPY app ./app` trở đi mới bị vô hiệu hóa cache và phải chạy lại, quá trình rebuild chỉ mất vỏn vẹn 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa một ký tự trong `app/main.py`, checksum của layer `COPY . .` thay đổi làm mất hiệu lực toàn bộ cache của tất cả các layer phía sau nó. Kết quả là Docker buộc phải thực thi lại lệnh `RUN pip install` từ đầu, tải và cài đặt lại toàn bộ thư viện qua mạng, khiến thời gian build tăng từ 2 giây lên vài phút trong mỗi chu kỳ phát triển.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công leo thang:
1. Ứng dụng Python có một lỗ hổng thực thi mã từ xa (RCE) do lỗi xử lý input hoặc từ một thư viện bên thứ ba.
2. Kẻ tấn công gửi payload khai thác thành công để mở một interactive shell bên trong container.
3. Nếu container chạy với user mặc định là `root` (UID 0), tiến trình shell của kẻ tấn công có toàn quyền quản trị tối cao trong container (sửa đổi file hệ thống, cài đặt rootkit, truy cập socket). Nếu container được mount volume nhạy cảm (như `/var/run/docker.sock`) hoặc host kernel có lỗ hổng (container breakout / privilege escalation), kẻ tấn công sẽ thoát ra khỏi container và ngay lập tức có quyền `root` (UID 0) trên máy host do mặc định UID trong container ánh xạ trực tiếp sang UID trên host.
4. Điểm cắt đứt: Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tấn công ngay tại bước 3. Khi kẻ tấn công thực thi được lệnh, tiến trình shell chỉ chạy với quyền của một người dùng thông thường bị hạn chế tối đa: không thể ghi vào các thư mục hệ thống của container, không có các Linux capabilities nguy hiểm, và không đủ quyền để tương tác với Docker daemon socket hay khai thác container escape sang máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được:
- Với fixed window (đếm theo phút đồng hồ), bộ đếm tự động reset về 0 tại mỗi giây `00`.
- Kẻ tấn công hoặc người dùng gửi dồn 10 request vào giây `10:00:59` (giây cuối cùng của phút). Hệ thống ghi nhận 10 request trong phút 10:00, vẫn nằm trong hạn mức cho phép.
- Ngay tại giây tiếp theo `10:01:00` (hoặc `10:01:01`), đồng hồ bước sang phút mới và bộ đếm reset về 0. Người dùng lập tức gửi tiếp 10 request nữa.
- Như vậy, chỉ trong khoảng 2 giây liên tiếp (từ 10:00:59 đến 10:01:01), hệ thống đã phải gánh chịu tổng cộng 20 request (gấp đôi hạn mức quy định), gây ra hiện tượng spike traffic đột ngột có thể làm nghẽn server. Thuật toán Sliding Window loại bỏ hoàn toàn kẽ hở này vì nó luôn tính tổng request trong đúng 60 giây trôi ngược từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt cốt lõi:
- **Rate Limit** bảo vệ hạ tầng máy chủ khỏi bị quá tải mạng và tài nguyên tính toán (CPU/RAM) bằng cách giới hạn **số lượng request trong chu kỳ ngắn** (ví dụ 10 request/phút).
- **Cost Guard** bảo vệ ngân sách tài chính khỏi bị thâm hụt bằng cách giới hạn **tổng chi phí tiền tệ/token trong chu kỳ dài** (ví dụ 10.0 USD/tháng).

Tình huống ví dụ:
1. **Rate limit cho qua nhưng Cost guard chặn:** Một người dùng cả tháng mới quay lại gọi API đúng 1 lần (tần suất 1 request/phút, hoàn toàn qua được Rate limit). Tuy nhiên, tháng trước người dùng này đã tiêu hết 10.0 USD ngân sách và chu kỳ tháng chưa reset. Cost guard kiểm tra thấy `spent >= budget` nên lập tức chặn và trả về lỗi `402 Payment Required`. (Hoặc người dùng gửi 1 request nhưng kèm văn bản khổng lồ khiến chi phí ước tính vượt quá số dư ngân sách còn lại).
2. **Cost guard cho qua nhưng Rate limit chặn:** Một người dùng mới toanh chưa tiêu đồng nào (ngân sách còn nguyên 10.0 USD), nhưng dùng script gửi liên tục 15 câu hỏi ngắn "hi" trong vòng 3 giây. Tổng chi phí của 15 câu này chỉ khoảng 0.001 USD (rất nhỏ so với budget 10 USD), nhưng Rate limiter sẽ lập tức kích hoạt từ request thứ 11 và trả về `429 Too Many Requests` do vi phạm tốc độ gọi API tối đa.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện cascading failure (sụp đổ dây chuyền):
1. Redis gặp sự cố mạng tạm thời hoặc restart, không thể kết nối trong 30 giây.
2. Endpoint gộp (đóng vai trò liveness probe) kiểm tra Redis và thấy thất bại nên trả về HTTP 503.
3. Bộ điều phối (Container Orchestrator như Docker Swarm, Kubernetes, Railway, Cloud Run) thấy liveness probe trả về lỗi 503 nên kết luận cả 3 container agent đều đã rơi vào trạng thái dead/treo tiến trình.
4. Orchestrator lập tức gửi tín hiệu SIGKILL / restart đồng loạt cả 3 container cùng một lúc để cố gắng tự phục hồi.
5. Trong khi cả 3 container đang bị khởi động lại, toàn bộ hệ thống rơi vào trạng thái "mất trắng" (100% outage), không còn instance nào tồn tại để nhận traffic của người dùng, trả về lỗi 502 Bad Gateway hàng loạt.
6. Khi Redis vừa hồi phục sau 30 giây, các container mới vẫn đang chật vật khởi động, tải thư viện và đồng thời ồ ạt kết nối lại Redis (hiện tượng thundering herd), khiến hệ thống tê liệt kéo dài hơn rất nhiều so với thời gian sự cố thực tế của Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis tập trung (Stateless architecture): Dù mỗi request được load balancer phân phối tới container ngẫu nhiên (A, B hoặc C), tất cả đều cùng truy cập một Redis key `history:<user_id>`, nên `history_length` tăng đều đặn và liên tục: 0 -> 2 -> 4 -> 6 -> 8...
- Nếu lưu lịch sử trong một dict Python nội bộ của tiến trình:
  Do mỗi container sở hữu một vùng nhớ RAM tách biệt hoàn toàn, load balancer phân phối request theo cơ chế Round-Robin hoặc Least Connections khiến request rơi rải rác vào các container khác nhau. Giá trị `history_length` sẽ nhảy lộn xộn, ngắt quãng: ví dụ request 1 vào container A (`history_length = 0`), request 2 vào container B (`history_length = 0` do B chưa từng nói chuyện với user này), request 3 vào container C (`history_length = 0`), request 4 lại vào container A (`history_length = 2`). Người dùng sẽ thấy agent có hiện tượng "mất trí nhớ ngẫu nhiên", câu trước vừa nói câu sau đã quên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Health check timeout / Service failed to bind to assigned port khi deploy lên Railway/Render.
- **Thông báo lỗi trong deploy log:** `Timed out waiting for container to become healthy` hoặc `Connection refused at 127.0.0.1:8000`.
- **Cách tìm ra nguyên nhân:** Khi xem runtime log của container trên dashboard của platform, tôi nhận thấy lệnh khởi động ban đầu cố định cổng `uvicorn app.main:app --host 127.0.0.1 --port 8000`. Trên các nền tảng đám mây hiện đại, platform tự động cấp phát cổng động ngẫu nhiên thông qua biến môi trường `$PORT` và kiểm tra sức khỏe từ bên ngoài qua network bridge. Việc app chỉ lắng nghe ở `127.0.0.1` khiến traffic từ mạng ngoài không thể chạm tới container, và việc cố định cổng 8000 khiến load balancer của platform (vốn kỳ vọng app chạy trên `$PORT`) không nhận được phản hồi.
- **Cách khắc phục:**
  1. Cập nhật CMD trong Dockerfile thành: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để bind ra tất cả các card mạng (`0.0.0.0`) và ưu tiên lấy cổng từ biến môi trường `${PORT}`.
  2. Cập nhật câu lệnh HEALTHCHECK tương ứng để đọc `$PORT`: `python -c "import urllib.request, os; port = os.environ.get('PORT', '8000'); urllib.request.urlopen(f'http://127.0.0.1:{port}/health').read()"`.
  Sau khi cập nhật và redeploy, service đã vượt qua kiểm tra sức khỏe và nhận traffic thành công.
