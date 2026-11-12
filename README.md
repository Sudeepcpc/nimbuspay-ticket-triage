# NimbusPay Ticket Triage

Fine-tuning project that converts NimbusPay support tickets into the required eight-field JSON triage record.

- **Base model:** `Qwen/Qwen2.5-1.5B-Instruct`
- **Best method:** 16-bit LoRA, rank 8; the base model was not loaded in 4-bit.
- **Best dev result:** 83.0% exact match, 97.31% mean field accuracy, and 100% valid JSON on 200 dev tickets.
- **Adapter:** `adapter/` (PEFT format)
- **Test predictions:** `predictions_test.jsonl` (400 rows)

Training data and the base model are not included.
