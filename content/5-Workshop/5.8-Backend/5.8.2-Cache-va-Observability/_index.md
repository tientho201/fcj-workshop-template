---
title: "Semantic Cache & Observability"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.8.2 </b> "
---

#### Custom Redis Client (`cache.py`)

`modules/query/lambda_src/chat_engine/cache.py`

Instead of `redis-py`, `chat_engine` uses a ~70-line **custom-implemented RESP2 client** that supports only the 3 required commands (`AUTH`/`GET`/`SETEX`) — adhering strictly to the zero-dependency principle stated on [page 5.8 Overview](../../5.8-Backend/). Authenticated via **short-lived IAM tokens** (SigV4-presigned URL, TTL 900 seconds), not static passwords.

**2 mandatory principles of this module:**

1. **Absolute best-effort** — all cache errors (connection loss, timeout, misconfiguration) are swallowed and logged as cache misses, **never failing the entire `/chat` request**.
2. **Per-invocation connections**, closed in `close()` — since Lambda "freezes" execution environments between calls, caching sockets at the module-level variable would leave a half-dead connection.

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
**Note on cache nature:** This is an **exact-match-after-normalisation** cache (`normalise_question` — lowercasing, stripping punctuation, collapsing whitespace), **not a true semantic cache**, because ElastiCache Serverless Redis lacks the RediSearch module for vector matching.
{{% /notice %}}

{{% notice note %}}
📌 **Important detail, easily misunderstood:** Caching is **not per-session** — the cache key only hashes **question content** (`hash(normalise_question(question))`), with no `session_id` in the key. This means **2 different sessions** asking the exact same question (on their first turn, when `cacheable = True`) will **hit the same cached result** — it is not cached separately per user. This is an **intentional design** to save Bedrock costs when multiple different users ask identical questions (e.g., many people asking "What is the leave policy?"), not a data isolation flaw — because the cached answer itself contains no private user information from previous askers.
{{% /notice %}}

#### Trace Mechanism (`tracing.py`)

`modules/query/lambda_src/chat_engine/tracing.py`

The `Tracer` class — a step-by-step execution timer, extremely lightweight (exactly 50 lines): each call to `.step(name, label, detail)` **closes the current step and records the actual elapsed milliseconds** since the previous step.

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
```

{{% notice note %}}
📌 This is the exact **data source for UI animations** ([Stream 2 Frontend, page 5.7.2](../../5.7-Frontend/5.7.2-Chat-Upload-UI/)) — **no step in the interface is staged**, all metrics come from Lambda's real `Tracer`.
{{% /notice %}}

`.skip()` records a step that was **deliberately not run** (e.g., query rewriting on the first turn, because there is no history) — the UI grays out that step instead of lighting up a fake status.

#### Answer Identifier (`message_id`)

Every time `/chat` responds — **even on cache hits or when blocked by Guardrails** — `chat_engine` generates a new `message_id` (`uuid4()`) and returns it in the response (`handler.py:509`).

{{% notice note %}}
📌 `message_id` **is not a conversation turn ID** (`session_id` exists for that — see [5.4.3](../../5.4-Realtime-QA/5.4.3-Cache-Lookup-Query-Rewriting/)), but an ID for **one specific answer** — allowing the UI to attach 👍/👎 buttons, and `POST /feedback` to use it as a key when writing to the `feedback` table (see [5.4.7](../../5.4-Realtime-QA/5.4.7-Alternative-Route/)).
{{% /notice %}}

`message_id` is also saved into `chat_history` when persisting a turn (`handler.py:226`) — **not to be read back immediately**, but for one specific purpose: the `chat_history` table has primary key `(session_id, timestamp)`, making it unqueryable by `message_id`. Thus a **GSI (`message_id-index`)** is needed to reverse-lookup from a `feedback` row back to the exact question/answer/context used — the sole reader is `evaluation_runner.py` ([Stream 4](../../5.6-RAGAS/5.6.3-RAGAS-Evaluation-Logic/)), via `_fetch_turn_by_message_id`.

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

#### Next Content

- [5.8.3 - Error Handling & Security](../5.8.3-Error-Handling-and-Security/)
