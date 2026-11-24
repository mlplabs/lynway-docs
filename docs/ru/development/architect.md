## Ecosystem architecture

```text
[ Android App ] ────(REST / JSON)────┐
  (Камера QR, остатки,               ▼
   перемещения, печать)        [ Go WMS Core ] ◄───► [ PostgreSQL ]
                                     ▲  │              (pg_trgm, JSONB,
[ Vue 3 Web UI ] ───(REST / JSON)────┘  │ (SSE / Poll)  storage ledger)
  (Приемка, поиск,                      ▼
   топология, админка)         [ CUPS Print Agent ] ──► [ Принтеры ]
```