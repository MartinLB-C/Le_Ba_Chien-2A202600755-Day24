# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Lê Bá Chiến  
**Ngày:** 30/06/2026

---

## Guard Stack Architecture

```
User Input
    |
    v (~2490.74ms P95)
[Presidio PII Scan]
    | block if: VN_CCCD / VN_PHONE / EMAIL detected
    | action:   return 400 + "PII detected in query"
    v (~1493.12ms P95)
[NeMo Input Rail]
    | block if: off-topic / jailbreak / prompt injection
    | action:   return 503 + refuse message
    v
[RAG Pipeline (Day 18)]
    | M1 Chunk -> M2 Search -> M3 Rerank -> qwen-flash
    v
[NeMo Output Rail]
    | flag if:  PII in response / sensitive content
    | action:   replace with safe response
    v
User Response
```

---

## Latency Budget

*(Điền từ kết quả Task 12 - measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---:|---:|---:|---|
| Presidio PII | N/A | 2490.74 | N/A | <10ms |
| NeMo Input Rail | N/A | 1493.12 | N/A | <300ms |
| RAG Pipeline | N/A | N/A | N/A | <2000ms |
| NeMo Output Rail | N/A | N/A | N/A | <300ms |
| **Total Guard** | N/A | **3983.86** | N/A | **<500ms** |

**Budget OK?** [ ] Yes / [x] No  
**Comment:** Total Guard P95 = 3983.86ms, vượt xa budget 500ms. Bottleneck chính là Presidio PII Scan và NeMo Input Rail. Cần cache/reuse `AnalyzerEngine`, `AnonymizerEngine`, `LLMRails` thay vì khởi tạo lại nhiều lần, đồng thời ưu tiên deterministic rules trước khi gọi LLM rail để giảm latency.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải >= 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | 0.8700 |
| Worst metric | faithfulness = 0.8519 |
| Dominant failure distribution | factual / answer_relevancy |
| Cohen's kappa | 0.000 (placeholder run) |
| Adversarial pass rate | 20 / 20 |
| Guard P95 latency | 3983.86 ms |

---

## Nhận xét & Cải tiến

Pipeline đã generate đủ 50 answers, Phase A chạy xong với RAGAS avg_score 0.8700 và adversarial suite đạt 20/20, nên phần quality/guardrail logic cơ bản hoạt động tốt. Điểm yếu RAGAS tổng hợp thấp nhất là faithfulness, trong khi failure cluster nổi bật là factual / answer_relevancy, cho thấy prompt trả lời và retrieval context vẫn cần cải thiện để bám sát câu hỏi hơn. Guard latency hiện vượt budget rất nhiều, đặc biệt ở Presidio và NeMo Input Rail, nên production cần cache guard engines, warm start service, và dùng rule-based checks trước LLM guard. Hai PDF scan bị bỏ qua vì không có text layer; nếu deploy thật cần thêm OCR để tăng coverage dữ liệu và giảm miss context.
