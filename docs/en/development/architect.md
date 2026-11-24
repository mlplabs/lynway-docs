## Ecosystem architecture

```text
[ Android App ] ────(REST / JSON)────┐
  (QR camera, inventory balance,     ▼
  transfers, printing)        [ Go WMS Core ] ◄───► [ PostgreSQL ]
                                     ▲  │              (pg_trgm, JSONB,
[ Vue 3 Web UI ] ───(REST / JSON)────┘  │ (SSE / Poll)  storage ledger)
  (Receiving, searching,                ▼
   topology, admin)             [ CUPS Print Agent ] ──► [ Printers ]
```