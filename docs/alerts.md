# Template Alert va Runbook

Moi alert phai dua tren trieu chung nguoi dung hoac SLO, khong dua truc tiep vao ten implementation noi bo.

## Alert 1

- Ten: high_latency_p95
- Severity: critical
- Duration: 5m
- Kenh thong bao: Slack
- SLI/SLO lien quan: fast_successful_requests (99.5% requests co latency <= 3000ms trong 28 ngay)
- Dieu kien va thoi gian duy tri: p95_latency_ms > 3000 lien tuc trong 5 phut
- Anh huong toi nguoi dung: Nguoi dung phai cho qua lau de nhan phan hoi tu AI, trai nghiem giam nghiem trong
- Ba buoc kiem tra dau tien:
  1. Kiem tra dashboard panel latency de xac dinh thoi diem bat dau va muc do anh huong
  2. Loc data/logs.jsonl theo khoang thoi gian do, tim cac request co latency_ms cao va lay correlation_id
  3. Tim trace tuong ung tren Langfuse bang correlation_id, xac dinh span nao chiem nhieu thoi gian nhat (retrieval hay LLM generation)
- Mitigation tam thoi: Tang timeout, kiem tra va restart service RAG neu bi slow, chuyen sang fallback prompt ngan hon de giam token generation
- Owner: on-call-sre

## Alert 2

- Ten: high_error_rate
- Severity: critical
- Duration: 3m
- Kenh thong bao: Slack
- SLI/SLO lien quan: fast_successful_requests SLO va guardrail error_rate_pct_max = 2%
- Dieu kien va thoi gian duy tri: error_rate_pct > 2 lien tuc trong 3 phut
- Anh huong toi nguoi dung: Nguoi dung nhan loi thay vi cau tra loi, mat tin tuong vao he thong
- Ba buoc kiem tra dau tien:
  1. Kiem tra dashboard panel errors de xac dinh loai loi (error_type) va ty le
  2. Loc data/logs.jsonl tim cac event "request_failed", doc error_type va correlation_id
  3. Mo trace tren Langfuse bang correlation_id, kiem tra span nao that bai (retrieval tool_fail, LLM error, v.v.)
- Mitigation tam thoi: Kiem tra ket noi toi vector store, restart service neu can, bat circuit breaker de tra fallback answer thay vi loi 500
- Owner: on-call-sre

## Alert 3

- Ten: quality_degradation
- Severity: warning
- Duration: 10m
- Kenh thong bao: Slack
- SLI/SLO lien quan: guardrail quality_score_avg_min = 0.75
- Dieu kien va thoi gian duy tri: quality_score_avg < 0.75 lien tuc trong 10 phut
- Anh huong toi nguoi dung: Cau tra loi cua AI co chat luong thap, khong dung ngu canh hoac khong day du, nguoi dung khong hai long
- Ba buoc kiem tra dau tien:
  1. Kiem tra dashboard panel quality de xac dinh xu huong va thoi diem bat dau giam
  2. Loc logs tim cac request co quality_score thap, kiem tra retrieval co tra ve documents hay khong (doc_count trong trace metadata)
  3. So sanh prompt version hien tai voi version truoc do tren Langfuse, kiem tra xem co thay doi prompt nao gay giam chat luong khong
- Mitigation tam thoi: Rollback prompt ve version truoc do (dat label production ve version cu), kiem tra retrieval corpus co day du khong
- Owner: ml-team
