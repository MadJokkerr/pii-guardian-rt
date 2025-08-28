# Real‑time PII Defense — Project Guardian 2.0

This repo contains a **self‑contained Python detector** and a **deployment plan** to catch and redact PII from JSON records and CSV streams.

## Files
- `detector_full_candidate_name.py` — main script (no external deps)
- `redacted_output_candidate_full_name.csv` — sample output (generated)
- `deployment_strategy.md` — where/how to deploy for low latency

## Run
```bash
python3 detector_full_candidate_name.py iscp_pii_dataset.csv
```
Input CSV must have: `record_id, data_json`.

### Output
`redacted_output_candidate_full_name.csv` with columns:
- `record_id`
- `redacted_data_json` (masked JSON string)
- `is_pii` (True/False)

## What counts as PII
- **Standalone:** phone(10d), aadhar(12d), passport(A1234567), upi_id(user@bank)
- **Combinatorial (need ≥2):** full name, email, physical address, device_id/ip **when in user context**

## Redaction
- Phone: `98XXXXXX10`
- Aadhar: `1234XXXX9012`
- Passport: `PXXXXXXX`
- UPI/Email: `abXXX@domain`
- Name: `JXXX SXXXXX`
- Address: `123... , 560001` (PIN preserved)
- IP: `a.XXX.XXX.d`
- Device: `abcXXXxyz`

## Notes
- Avoids false positives: ignores 10/12‑digit numbers in product/order IDs.
- Handles malformed JSON gracefully (passes through, `is_pii=False`).
