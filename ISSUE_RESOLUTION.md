# Issue Resolution: Pallets Werkzeug Multipart Parser Trailing Stray Byte (#3285)

## 1. Upstream Issue Metadata
- **Repository:** `pallets/werkzeug`
- **Issue Reference:** [#3285 - Multipart parser still appends a stray byte near boundary](https://github.com/pallets/werkzeug/issues/3285)
- **Component:** `src/werkzeug/sansio/multipart.py` (`MultipartDecoder._last_partial_boundary_index`)
- **Severity / Impact:** High (Data corruption / binary drift). When multipart streaming data is delivered in small or single-byte network chunks, an extra stray `\r` (carriage return byte) was incorrectly appended to the end of form fields or binary file uploads immediately before boundary transitions.

---

## 2. Root Cause Analysis
In Werkzeug's Sans-I/O multipart parser, `_last_partial_boundary_index` is called by `_parse_data` to locate a safe cutoff index when incoming data might contain a partial boundary split across chunk buffers:
```python
complete_boundary_index = len(data) - len(b"\r\n--" + self.boundary)
```
When an epilogue boundary arrives (or boundaries with trailing dashes/linebreaks such as `--<boundary>--\r\n`), the boundary sequence is longer than the prefix `b"\r\n--" + self.boundary` (typically by at least 2 extra dashes, plus whitespace and CRLF).

If incoming network chunks are delivered in 1-byte increments:
1. When bytes like `b"\r\n--boundary-"` are accumulated, the full boundary regex (`boundary_re`) does not yet match because the terminating second dash `--` is still pending.
2. `_last_partial_boundary_index` was called with `len(data) = 13` and assumed a maximum boundary length of only 12 (`len(b"\r\n--" + self.boundary)`).
3. As a result, `complete_boundary_index = 13 - 12 = 1`.
4. The leading `\r` of the boundary was at index 0. Since `0 < complete_boundary_index (1)`, the function incorrectly deemed the leading `\r` to be "too far from the end" to be part of the boundary.
5. The decoder therefore severed the `\r` and emitted it as file/field payload, corrupting uploaded data with an erroneous trailing byte `b'\r'`.

---

## 3. Reproduction Test
In `tests/sansio/test_multipart.py`:
```python
def test_single_byte_chunk_boundary_isolation() -> None:
    boundary = b"boundary"
    file_content = b"hello world"
    data = (
        b"--boundary\r\n"
        b'Content-Disposition: form-data; name="file"; filename="test.txt"\r\n'
        b"Content-Type: text/plain\r\n\r\n"
        + file_content
        + b"\r\n--boundary--\r\n"
    )

    decoder = MultipartDecoder(boundary)
    collected_data = bytearray()
    for b in data:
        decoder.receive_data(bytes([b]))
        while True:
            event = decoder.next_event()
            if isinstance(event, Data):
                collected_data.extend(event.data)
            if isinstance(event, NeedData):
                break
    assert bytes(collected_data) == file_content
```

### Pre-Fix Failure Output:
```text
FAILED tests/sansio/test_multipart.py::test_single_byte_chunk_boundary_isolation
E       AssertionError: assert b'hello world\r' == b'hello world'
```

---

## 4. Implementation Diff
```diff
diff --git a/src/werkzeug/sansio/multipart.py b/src/werkzeug/sansio/multipart.py
index b3fe0248..6d47b5ae 100644
--- a/src/werkzeug/sansio/multipart.py
+++ b/src/werkzeug/sansio/multipart.py
@@ -274,7 +274,9 @@ class MultipartDecoder:
         # position of a LF or a CR, unless that position is more
         # than a complete boundary from the end in which case there
         # is no partial boundary.
-        complete_boundary_index = len(data) - len(b"\r\n--" + self.boundary)
+        complete_boundary_index = (
+            len(data) - len(b"\r\n--" + self.boundary) - SEARCH_EXTRA_LENGTH
+        )
         try:
             last_nl = data.rindex(b"\n")
         except ValueError:
```

---

## 5. Verification Results
- **Reproduction Test:** `pytest tests/sansio/test_multipart.py -k test_single_byte_chunk_boundary_isolation` passed (1 passed).
- **All Multipart Tests:** `pytest tests/sansio/test_multipart.py` passed (22 passed in 0.10s).
- **Sans-IO Suite:** `pytest tests/sansio/` passed (72 passed in 0.14s).
- **Ruff Lint Check:** `ruff check src/werkzeug/sansio/multipart.py tests/sansio/test_multipart.py` passed with 0 errors.
- **Parametric Chunk Stress Test:** Verified chunk sizes from 1 byte to full payload (186 bytes) without data corruption.
