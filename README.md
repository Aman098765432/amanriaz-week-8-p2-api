# Week 8: Plant Leaf Disease API (FastAPI)

This folder serves my Week 6 leaf-disease image model and my Week 7 text evidence as a web service.

## What it does

| Endpoint | Input | Output |
|---|---|---|
| `GET /health` | none | status, model loaded, classes (`diseased`, `healthy`), positive class |
| `POST /predict` | leaf image (JPEG or PNG) | label, class probabilities, latency in ms |
| `POST /analyze` | leaf image + a field note | masked note, extracted fields, urgent words, decision (`REFER` or `ROUTINE`) with reasons |

`/analyze` masks the CNIC, phone number and email in the note **before** any extraction, so no personal identifier appears anywhere in the response. A `REFER` decision is raised when the image model gives P(diseased) of at least 0.5 or the note mentions a warning word such as spreading, blight, wilting or yellow.

Error handling (no stack traces): wrong file type returns **415**, a file over 5 MB returns **413**, a corrupt image returns **400**, and a missing file or a note over 2,000 characters returns **422**.

## Files

- `app.py`: the API
- `test_api.py`: 8 automated tests
- `bench.py`: latency benchmark
- `requirements.txt`: packages
- `sample.jpeg`: a test leaf image (tomato late blight)
- `week6_model.pt`: **not stored here** (44 MB). Copy the trained model from Week 6 into this folder before running.

## How to run

```
cd api
python -m venv .venv
.venv\Scripts\activate
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
uvicorn app:app
```

Open http://127.0.0.1:8000/docs and use "Try it out", or call it with curl:

```
curl -X POST http://127.0.0.1:8000/predict -F "file=@sample.jpeg;type=image/jpeg"

curl -X POST http://127.0.0.1:8000/analyze -F "file=@sample.jpeg;type=image/jpeg" -F "note=Brown patches spreading on tomato leaves. Call 0300-1234567"
```

On Mac or Linux activate with `source .venv/bin/activate` and continue lines with `\` instead of `^`.

## Tests

`pytest -q test_api.py` gave **8 passed**. The tests cover the health check, the response schema and probabilities, parity with calling the model directly, rejection of a wrong type (415), an oversized file (413), a corrupt image (400), a missing file (422), and a PII test that fails if any part of the phone number or CNIC appears in the `/analyze` response.

![pytest output](pytest.png)

A successful prediction and a rejected input:

![predict](predict.png)
![415 response](rejected-415.png)

## Benchmark

50 requests to `/predict` against a running server, on [CPU model] with [RAM] GB RAM, CPU only, one worker:

| p50 | p95 | max | throughput |
|---|---|---|---|
| 45 ms | 70 ms | 206 ms | about 19.5 requests/s |

The maximum comes from the first request, while the model is still warming up.

![benchmark](bench.png)

### Capacity (Little's law)

At p50 = 45 ms one request takes 0.045 s, so one worker can handle 1 / 0.045 = 22.2 requests per second. Over a day this is 22.2 x 86,400 = about 1.92 million requests at full load. A busy server should not run at 100%, so at 65% utilisation I would plan for about 1.2 million requests per day per worker.

## An extra bad input

[Write what you tried and what the API returned. Example: I sent a 3,000-character note to `/analyze`; the API returned 422 "Note longer than 2,000 characters" with no stack trace.]

## Check Your Understanding

1. **A PDF sent to /predict.** It should get **415 Unsupported Media Type**. The client sent the wrong kind of input, which is a client error (4xx). A 500 means a bug in the server, and it would hide a clear, fixable message behind a crash.
2. **Why load the model in `lifespan`.** Loading the weights takes seconds and memory. Inside the endpoint it would repeat on every request. In `lifespan` it happens once at startup and all requests share the same model.
3. **p50 = 40 ms and p95 = 400 ms.** A typical request is fast, but about 1 in 20 is ten times slower (a heavy tail). I would check for first-call warm-up, requests queuing behind each other, very large images, other programs using the CPU, and blocking code on the event loop.
4. **Workers needed.** 500,000 / 86,400 = 5.8 requests/s on average. Peak is 5 times that, about 29 requests/s. With 25 requests/s per worker, 29 / 25 = 1.16, so **2 workers** (3 would leave a safety margin).
5. **Inference inside `async def`.** Model inference is heavy CPU work that never hands control back. While it runs, the event loop cannot answer other requests, even a quick `/health`. Running it in a thread pool (`run_in_threadpool`) keeps the loop free.
6. **Channels in a 4-person team.** n(n - 1) / 2 = 4 x 3 / 2 = **6**.

## Note

Decision support only. A qualified agronomist makes the final call.
