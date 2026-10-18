# HTTP contract

All inventory routes require a verified cookie session. JSON requests use Content-Type application/json; photo uploads use FormData (let the browser set its multipart boundary). Success is HTTP 200 `{ "data": ..., "request_id": "server-generated-uuid" }`; errors are `{ "error": "Actionable message", "request_id": "uuid" }`. Auth/config responses omit request ID. All product mutations are non-cacheable. Foreign Origin mutation requests are rejected (403); same-origin browser cookies are managed by Supabase SSR.

| Method / path | Input | data on success |
| --- | --- | --- |
| POST `/api/auth` | `{email,password}` or `{action:"logout"}` | `{ok:true}` and cookie changes |
| GET `/api/config` | none | `{configured,email,defect}`; email null when signed out |
| GET `/api/products` | none | own product array, optional signed image_url/photo_error |
| POST `/api/products` | product below | saved product incl UUID/quantity |
| PUT `/api/products?id=UUID` | product below, opening_quantity 0 | updated product |
| PATCH `/api/products` | `{id:"UUID",action:"archive"}`; restore/delete also supported | `{ok:true}` |
| GET `/api/movements?product_id=UUID` | optional filter | newest 500 own movements |
| POST `/api/movements` | movement below | movement with before/after balances |
| POST `/api/photos` | multipart field `file` | `{path:"owner-uuid/generated-uuid.png"}` |
| DELETE `/api/photos` | `{path:"owner-uuid/generated-uuid.png"}` | `{ok:true}` after unused-object removal |

```json
{"sku":"PEN-01","name":"Blue ballpoint pens","category":"Writing","price_centavos":1250,"opening_quantity":10,"low_stock_threshold":3,"image_path":null}
```

```json
{"product_id":"00000000-0000-4000-8000-000000000001","type":"OUT","quantity":3,"note":"Released three","request_id":"00000000-0000-4000-8000-000000000002"}
```

UUIDs above are examples, not valid saved records. Use real returned IDs. A movement success contains `previous_balance:15,resulting_balance:12,quantity:3,type:"OUT"` plus IDs/note/time. Reusing a movement request ID with the same payload returns that movement without applying stock twice. Changed payload + same key returns 409. Product edits may not rewrite stock.

| Status | Meaning / recovery |
| --- | --- |
| 400 | malformed JSON/request; correct syntax |
| 401 | missing/expired login; sign in |
| 403 | cross-site mutation blocked; open app directly |
| 404 | unknown or other-owner product; no ownership detail leaked |
| 409 | duplicate SKU, insufficient stock, archived product, history prevents delete, or idempotency conflict; resolve then retry appropriately |
| 422 | field/file validation failure; correct input |
| 503 | unconfigured/unavailable service or simulated storage failure; restore dependency, retry unchanged movement key |

The UI shows safe problem/action text. Development logs contain operation, server request ID, success/failure, status, not passwords, tokens, raw payloads, or secrets. The body `request_id` for a movement is an idempotency key; the envelope request_id is a separate per-HTTP diagnostic ID. Never confuse them in a report.

test diff
