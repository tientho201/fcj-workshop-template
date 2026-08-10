---
title: "Semantic Cache & Observability"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.8.2 </b> "
---

#### Client Redis tự viết (`cache.py`)

`modules/query/lambda_src/chat_engine/cache.py`

Thay vì `redis-py`, `chat_engine` dùng một **RESP2 client ~70 dòng tự cài** chỉ hỗ trợ đúng 3 lệnh cần dùng (`AUTH`/`GET`/`SETEX`) — giữ đúng nguyên tắc zero-dependency đã nói ở [trang Tổng thể 5.8](../../5.8-Backend/). Xác thực bằng **IAM token ngắn hạn** (SigV4-presigned URL, TTL 900 giây), không phải mật khẩu tĩnh.

**2 nguyên tắc bắt buộc của module này:**

1. **Best-effort tuyệt đối** — mọi lỗi cache (mất kết nối, timeout, sai cấu hình) đều bị nuốt và log lại thành cache miss, **không bao giờ làm fail cả request `/chat`**.
2. **Kết nối theo từng lần invoke**, đóng lại ở `close()` — vì Lambda "đóng băng" môi trường thực thi giữa các lần gọi, nếu cache socket vào biến module-level sẽ để lại 1 kết nối nửa sống nửa chết.

```python
_CONNECT_TIMEOUT_SECONDS = 2.0
_IAM_TOKEN_TTL_SECONDS = 900


def _iam_auth_token(cache_name, user_name, region):
    """Generate a short-lived ElastiCache IAM auth token.

    The token is a SigV4-presigned URL with its scheme stripped; Redis AUTH
    takes it as the password. Note ElastiCache requires the user's UserId and
    UserName to be identical for IAM auth, which is why the Terraform sets
    both to the same value.
    """
    session = boto3.Session()
    signer = RequestSigner(
        ServiceId("elasticache"),
        region,
        "elasticache",
        "v4",
        session.get_credentials(),
        session.events,
    )
    query = urlencode({"Action": "connect", "User": user_name})
    presigned = signer.generate_presigned_url(
        request_dict={
            "method": "GET",
            "url": f"https://{cache_name}/?{query}",
            "body": {},
            "headers": {},
            "context": {},
        },
        operation_name="connect",
        expires_in=_IAM_TOKEN_TTL_SECONDS,
        region_name=region,
    )
    return presigned.removeprefix("https://")


class _Resp:
    """Minimal RESP2 client over a TLS socket."""

    def __init__(self, host, port, timeout=_CONNECT_TIMEOUT_SECONDS):
        raw_socket = socket.create_connection((host, port), timeout=timeout)
        # ElastiCache Serverless always enforces in-transit encryption.
        context = ssl.create_default_context()
        self._socket = context.wrap_socket(raw_socket, server_hostname=host)
        self._reader = self._socket.makefile("rb")

    def command(self, *args):
        parts = [f"*{len(args)}\r\n".encode()]
        for arg in args:
            encoded = arg.encode() if isinstance(arg, str) else arg
            parts.append(f"${len(encoded)}\r\n".encode())
            parts.append(encoded)
            parts.append(b"\r\n")
        self._socket.sendall(b"".join(parts))
        return self._read_reply()

    def _read_reply(self):
        line = self._reader.readline()
        if not line:
            raise ConnectionError("Redis closed the connection")

        kind, payload = line[:1], line[1:-2]
        if kind == b"+":
            return payload.decode()
        if kind == b"-":
            raise RuntimeError(payload.decode())
        if kind == b":":
            return int(payload)
        if kind == b"$":
            length = int(payload)
            if length == -1:
                return None
            return self._reader.read(length + 2)[:-2]
        if kind == b"*":
            count = int(payload)
            if count == -1:
                return None
            return [self._read_reply() for _ in range(count)]
        raise RuntimeError(f"Unexpected RESP reply: {line!r}")

    def close(self):
        try:
            self._reader.close()
            self._socket.close()
        except Exception:  # noqa: BLE001
            pass


def normalise_question(question):
    """Collapse cosmetic differences so trivially-reworded repeats still hit.

    This is exact-match-after-normalisation, NOT true semantic similarity:
    ElastiCache Serverless does not offer the RediSearch module, so there is
    no vector index to compare against inside Redis. See the comment on
    aws_elasticache_serverless_cache in modules/query/main.tf for the upgrade
    path (Amazon MemoryDB) if real similarity matching is needed.
    """
    lowered = question.strip().lower()
    without_punctuation = re.sub(r"[^\w\s]", "", lowered, flags=re.UNICODE)
    return re.sub(r"\s+", " ", without_punctuation).strip()


def cache_key(question):
    digest = hashlib.sha256(normalise_question(question).encode("utf-8")).hexdigest()
    return f"semcache:{digest}"


class SemanticCache:
    def __init__(self):
        self._client = None
        self._host = os.environ.get("REDIS_ENDPOINT")
        self._port = int(os.environ.get("REDIS_PORT", "6379"))
        self._cache_name = os.environ.get("REDIS_CACHE_NAME")
        self._user_name = os.environ.get("REDIS_USER_ID")
        self._region = os.environ.get("AWS_REGION")
        self._ttl = int(os.environ.get("CACHE_TTL_SECONDS", "3600"))

    def _connect(self):
        if self._client is not None:
            return self._client
        if not (self._host and self._cache_name and self._user_name):
            raise RuntimeError("Semantic cache is not configured")

        client = _Resp(self._host, self._port)
        token = _iam_auth_token(self._cache_name, self._user_name, self._region)
        client.command("AUTH", self._user_name, token)
        self._client = client
        return client

    def get(self, question):
        try:
            value = self._connect().command("GET", cache_key(question))
            return value.decode("utf-8") if isinstance(value, bytes) else value
        except Exception:  # noqa: BLE001 - a cache miss is always an acceptable outcome
            logger.warning("Semantic cache read failed; treating as a miss", exc_info=True)
            return None

    def set(self, question, answer):
        try:
            self._connect().command("SETEX", cache_key(question), str(self._ttl), answer)
        except Exception:  # noqa: BLE001
            logger.warning("Semantic cache write failed; continuing", exc_info=True)

    def close(self):
        if self._client is not None:
            self._client.close()
            self._client = None

```

{{% notice warning %}}
**Lưu ý về bản chất cache:** đây là cache **exact-match sau khi chuẩn hóa** (`normalise_question` — hạ chữ thường, bỏ dấu câu, gộp khoảng trắng), **không phải semantic cache thật**, vì ElastiCache Serverless Redis không có module RediSearch để so khớp vector.
{{% /notice %}}

{{% notice note %}}
📌 **Chi tiết quan trọng, dễ hiểu nhầm:** cache **không theo từng session** — khóa cache chỉ băm **nội dung câu hỏi** (`hash(normalise_question(question))`), không có `session_id` nào trong khóa. Nghĩa là **2 phiên khác nhau** hỏi đúng y hệt 1 câu (ở lượt đầu tiên của mỗi phiên, khi `cacheable = True`) sẽ **cùng trúng 1 kết quả cache** — không phải cache riêng cho từng người. Đây là **thiết kế có chủ đích** để tiết kiệm chi phí Bedrock khi nhiều người dùng khác nhau hỏi trùng câu hỏi (ví dụ nhiều người cùng hỏi "Chính sách nghỉ phép là gì?"), không phải lỗ hổng cô lập dữ liệu — vì bản thân câu trả lời được cache không chứa thông tin riêng tư của người hỏi trước.
{{% /notice %}}

#### Cơ chế Trace (`tracing.py`)

`modules/query/lambda_src/chat_engine/tracing.py`

Class `Tracer` — bộ đếm thời gian từng bước, cực kỳ tối giản (đúng 50 dòng): mỗi lần gọi `.step(name, label, detail)` sẽ **đóng bước hiện tại lại và ghi số mili-giây thật đã trôi qua** kể từ bước trước.

```python
class Tracer:
    def __init__(self):
        self._started_at = time.perf_counter()
        self._step_started_at = self._started_at
        self.steps = []

    def step(self, name, label, detail="", status="ok"):
        """Close off the current step and record how long it took."""
        now = time.perf_counter()
        self.steps.append(
            {
                "name": name,
                "label": label,
                "ms": int((now - self._step_started_at) * 1000),
                "detail": detail,
                "status": status,
            }
        )
        self._step_started_at = now

    def skip(self, name, label, detail=""):
        """Record a step that was deliberately not run (e.g. rewriting on a
        first turn). The UI greys these out instead of lighting them up."""
        self.steps.append(
            {"name": name, "label": label, "ms": 0, "detail": detail, "status": "skipped"}
        )
        self._step_started_at = time.perf_counter()

    def fail(self, name, label, detail=""):
        self.step(name, label, detail=detail, status="error")

    @property
    def total_ms(self):
        return int((time.perf_counter() - self._started_at) * 1000)

    def to_dict(self):
        return {"steps": self.steps, "total_ms": self.total_ms}
)
```

{{% notice note %}}
📌 Đây chính là **nguồn dữ liệu cho animation phía UI** ([Luồng 2 Frontend, trang 5.7.2](../../5.7-Frontend/5.7.2-Chat-Upload-UI/)) — **không có bước nào trong giao diện là dàn dựng**, toàn bộ số liệu đến từ `Tracer` thật của Lambda.
{{% /notice %}}

`.skip()` ghi nhận một bước **chủ động không chạy** (ví dụ query rewriting ở lượt hỏi đầu tiên, vì chưa có lịch sử) — UI tô xám bước đó thay vì sáng đèn giả.

#### Định danh câu trả lời (`message_id`)

Mỗi lần `/chat` trả lời — **kể cả khi cache hit hay bị Guardrail chặn** — `chat_engine` sinh một `message_id` mới (`uuid4()`) và trả về cùng response (`handler.py:509`).

{{% notice note %}}
📌 `message_id` **không phải id của lượt hội thoại** (đã có `session_id` cho việc đó — xem [5.4.3](../../5.4-Realtime-QA/5.4.3-Cache-Lookup-Query-Rewriting/)), mà là id của **đúng 1 câu trả lời cụ thể** — để UI gắn nút 👍/👎 vào, và `POST /feedback` dùng nó làm khóa ghi vào bảng `feedback` (xem [5.4.7](../../5.4-Realtime-QA/5.4.7-Alternative-Route/)).
{{% /notice %}}

`message_id` cũng được lưu kèm vào `chat_history` khi ghi lượt hội thoại (`handler.py:226`) — **không phải để đọc lại ngay**, mà để phục vụ đúng 1 việc: bảng `chat_history` có khóa chính `(session_id, timestamp)`, không tra được theo `message_id`, nên cần một **GSI (`message_id-index`)** để nối ngược từ 1 dòng `feedback` về đúng câu hỏi/câu trả lời/ngữ cảnh đã dùng — bên đọc duy nhất là `evaluation_runner.py` ([Luồng 4](../../5.6-RAGAS/5.6.3-RAGAS-Evaluation-Logic/)), qua hàm `_fetch_turn_by_message_id`.

```python
    now_seconds = int(time.time())
    chat_history_table.put_item(
        Item={
            "session_id": session_id,
            "timestamp": int(time.time() * 1000),
            "message_id": message_id,
            "question": question,
            "answer": answer,
            "retrieved_context": [p.get("parent_text", "") for p in parents],
            "source_document_ids": [p.get("document_id", "") for p in parents],
            "expires_at": now_seconds + CHAT_HISTORY_TTL_DAYS * 86400,
        }
    )
```

---

#### Nội dung tiếp theo

- [5.8.3 - Xử lý lỗi & Bảo mật](../5.8.3-Xu-ly-loi-va-Bao-mat/)
