# Лаб 2: Гүйцэтгэлийн хэмжүүрийг k6-аар хэмжих

F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)

- Нэр: Г.Эрхэмбаяр
- Оюутны код: B232270055

Бай: зөвхөн `https://test.k6.io` (зааврын албан ёсны дадлагын сайт). Өөр сайт руу тест хийгээгүй.

## Орчин

`k6 version`:

```
k6.exe v2.4.0 (commit/dd9395906d, go1.26.11, windows/amd64)
```

OS: Windows 11, PowerShell. Гаралтыг `k6 run ... 2>&1 | Out-File -Encoding utf8 results\...` командаар хадгалсан.

## Файлууд

| Файл | Тайлбар |
|---|---|
| `script.js` | Үндсэн тест (Алхам 2), 5 VU / 30с. Алхам 3-т `--vus`, `--duration`-ийг CLI-аар дарж ажиллуулсан |
| `stages.js` | Ачааллыг шатлан өсгөх хувилбар (5 → 30 → 100 → 0 VU) |
| `thresholds.js` | SLO threshold-тай хувилбар (30 VU, 1м) |
| `results/run-05vu.txt`, `run-30vu.txt`, `run-100vu.txt` | Гурван түвшний бүтэн гаралт |
| `results/run-stages.txt` | Stages ажиллуулалтын бүтэн гаралт |
| `results/run-threshold-pass.txt`, `run-threshold-fail.txt` | Threshold PASS ба FAIL бүтэн гаралт (төгсгөлд нь `exit code`) |
| `screenshots/` | k6 summary-н дэлгэцийн зургууд |

## Алхам 2: Анхны тест (baseline, 5 VU)

`k6 run --vus 5 --duration 1m script.js` (`results/run-05vu.txt`). Энэ ачаалал багатай ажиллуулалт бол BASELINE.

| Хэмжүүр | Утга |
|---|---|
| `http_req_duration` avg / p90 / p95 | 163.06 ms / 281.65 ms / 306.34 ms |
| `http_reqs` (throughput) | 448 нийт, 7.33 req/s |
| `http_req_failed` (error rate) | 0.00% (0 / 448) |
| `checks` | 224 / 224 амжилттай |

`http_reqs` (448) нь `iterations` (224)-ийн яг 2 дахин байна: нэг iteration-д k6 2 хүсэлт тоолсон. Шалтгаан нь redirect байж магадгүй, үүнийг шалгаагүй. Throughput гэж `http_reqs/s`-ийг авсан.

## Алхам 3: 5 / 30 / 100 VU харьцуулалт

Түвшин тус бүрийг `script.js`-ээр тусад нь 1 минут ажиллуулсан:

```
k6 run --vus 5   --duration 1m script.js
k6 run --vus 30  --duration 1m script.js
k6 run --vus 100 --duration 1m script.js
```

| VU | p90 | p95 | avg | max | throughput (http_reqs/s) | error rate |
|---|---|---|---|---|---|---|
| 5 | 281.65 ms | 306.34 ms | 163.06 ms | 778.51 ms | 7.33 | 0.00% |
| 30 | 235.97 ms | 240.11 ms | 150.06 ms | 1.42 s | 44.79 | 0.00% |
| 100 | 238.95 ms | 248.83 ms | 152.18 ms | 1.21 s | 148.38 | 0.00% |

Тоонууд `results/run-05vu.txt`, `run-30vu.txt`, `run-100vu.txt` файлуудаас авсан.

**Stages ажиллуулалт** (`stages.js`, `results/run-stages.txt`, 2м30с, оргил 100 VU): p90 = 241.74 ms, p95 = 250.8 ms, avg = 157.46 ms, max = 2.06 s, 6896 хүсэлт (дундаж 45.74 req/s), error rate 0.00%. Нэг ажиллуулалт нэг нэгтгэсэн summary өгдөг тул хүснэгтэд ашиглаагүй, зөвхөн ерөнхий зургийг ажигласан.

**Хэрэглэгчийн туршлага муудах цэг:** 100 VU хүртэл олдсонгүй. p95 нь 5 VU-ээс 100 VU хүртэл өсөөгүй, error rate 0.00% хэвээр.

## Алхам 4: Thresholds (SLO)

Baseline p95 = 306.34 ms. SLO-г тэр baseline-аас 1.5 дахин нөөцтэйгөөр тогтоосон: 306.34 × 1.5 ≈ 460 ms. Шалтгаан: нөөц нь сүлжээний жирийн хэлбэлзлийг тэсвэрлэх боловч хариу бараг хоёр дахин удаашрах бодит муудалтыг барина. Error rate-ийг 1%-иас бага байхаар авсан.

```js
thresholds: {
  http_req_duration: [`p(95)<${__ENV.P95 || 460}`], // SLO: baseline p95 306ms x 1.5
  http_req_failed: ['rate<0.01'],
},
```

FAIL хувилбарыг тусдаа файл болгохгүй, `-e P95=50` env хувьсагчаар ажиллуулсан.

| Ажиллуулалт | Threshold | p95 | Error rate | Үр дүн |
|---|---|---|---|---|
| `k6 run thresholds.js` | `p(95)<460`, `rate<0.01` | 246.86 ms | 0.00% | PASS (`✓` хоёул), exit code 0 |
| `k6 run -e P95=50 thresholds.js` | `p(95)<50`, `rate<0.01` | 248.66 ms | 0.00% | FAIL (`✗ 'p(95)<50'`), exit code 99 |

FAIL үед k6 нь `thresholds on metrics 'http_req_duration' have been crossed` гэсэн алдаа гаргаж, 0-ээс өөр exit code буцаасан. CI pipeline-д quality gate яг ингэж ажилладаг.

## Дүгнэлт

5-аас 100 VU хүртэл ачаалал 20 дахин өсөхөд throughput 7.33-аас 148.38 req/s болж бараг шугаман өссөн. Харин p95 latency өссөнгүй: 306.34, 240.11, 248.83 ms. Error rate бүх түвшинд 0.00% байсан. Иймд test.k6.io дээр 100 VU хүртэл гүйцэтгэл муудах цэг олдсонгүй, лекцийн "10 хэрэглэгчтэй үед 2с, 100 хэрэглэгчтэй үед 4с" гэсэн зөрчил энэ сайт дээр давтагдсангүй. 5 VU дээрх p95 өндөр байсан нь цөөн дээж (448 хүсэлт) болон цөөн удаан хүсэлтийн нөлөө байж магадгүй, энэ нь таамаг. Дундаж latency 150-163 ms байхад хамгийн удаан хүсэлт 0.78-2.06 s хүрсэн нь дундаж нь tail latency-г нууцалдаг гэсэн лекцийн ойлголтыг, мөн p95/p99 хэрэгтэйг батална. Throughput ба latency нь өөр хэмжүүр бөгөөд throughput өсөхөд latency тогтвортой байж болохыг харсан. Stages ажиллуулалтад оргил 100 VU үед ч error rate 0.00%, p95 250.8 ms байв. Тодорхойлсон SLO (p95 < 460 ms, error < 1%) хангагдсан: p95 = 246.86 ms, error 0.00%. Threshold-ыг санаатайгаар p(95)<50 болгоход тест FAIL болж exit code 99 буцаасан нь CI-ийн quality gate-тэй ижил зарчим. Хэмжилт миний машин, сүлжээ, цагаас хамаарах тул давтахад тоо өөр гарч болно.
