# NimbusPay Ticket Triage — Experiment Report

## Task and data audit

The task is to convert each support ticket into exactly eight JSON fields: category, priority, amount in paise, transaction date, transaction ID, payment channel, language, and whether a human should review it.

The raw training file contained 1,445 rows. Auditing found malformed assistant JSON, inconsistent keys and types, invalid dates and transaction IDs, and labels inconsistent with the supplied schema. The cleaned training file contains 1,418 rows. We dropped 25 rows with malformed JSON labels and two unusable placeholder tickets. Fix and drop actions are recorded in `cleaning_log.csv` (238 entries). Repairs included correcting keys and types, normalizing dates and transaction IDs, converting amounts to paise, and correcting priority and human-review labels. The 200-row dev set was not used for training.

## Baselines

Both base-model baselines were evaluated on the 200 labeled dev tickets.

| Model/prompt | Valid JSON | Exact match | Mean field accuracy |
|---|---:|---:|---:|
| Base model, fixed prompt | 0.0% | 0.0% | 0.0% |
| Base model, rules in prompt | 8.0% | 0.5% | 4.75% |

## Fine-tuning experiments

All fine-tuned runs used `Qwen/Qwen2.5-1.5B-Instruct`, a 2,304-token sequence limit, and fewer than 50M trainable parameters. Dev metrics are on 200 examples. GPU memory is peak allocated/reserved.

| Experiment | Change | Exact match | Mean field accuracy | Minutes | Peak GPU GB (allocated/reserved) |
|---|---|---:|---:|---:|---:|
| Raw QLoRA, rank 16 | Raw versus cleaned data | 62.0% | 93.81% | 35.0 | 7.11 / 14.20 |
| Clean QLoRA, rank 16 | Cleaned data | 77.5% | 96.25% | 34.7 | 6.22 / 14.22 |
| Clean FP16 LoRA, rank 16 | FP16 LoRA versus 4-bit QLoRA | 80.5% | 96.69% | 25.2 | 8.71 / 13.64 |
| Clean FP16 LoRA, rank 8 | Rank 8 versus rank 16 | **83.0%** | **97.31%** | 24.5 | 9.08 / 14.00 |
| Priority-upsampled FP16 LoRA, rank 8 | Duplicate P1 and P2 training examples once | 81.5% | 96.90% | 24.9 | 6.97 / 13.18 |

The priority experiment changed P1/P2/P3 counts from 281/346/791 to 562/692/791. It did not improve dev performance; the clean rank-8 adapter was selected. All fine-tuned runs produced valid JSON on the dev set. The selected adapter has 9,232,384 trainable parameters.

## Error analysis of the best dev model

Field accuracy was: category 97.5%, priority 92.0%, amount 96.0%, date 99.0%, transaction ID 100%, channel 99.5%, language 100%, and needs_human 94.5%. Common issues were priority thresholds, interpreting prior ticket references as repeat contact, and converting rupees to paise.

Examples (expected → predicted):

- **dv-00001, needs_human:** false → true. An earlier ticket number was treated as repeat contact about this issue.
- **dv-00007, amount_paise:** 87,700 → 877. The model omitted the ×100 rupees-to-paise conversion.
- **dv-00010, category/priority/needs_human:** payment_failed/P3/false → fraud/P1/true. “I didn’t see the money I added” was mistaken for an unauthorized transaction.
- **dv-00011, priority/needs_human:** P1/true → P2/false. ₹84,137 meets the ₹50,000 P1 threshold.
- **dv-00018, priority:** P3 → P2. ₹4,200 is below the ₹5,000 refund threshold for P2.
- **dv-00022, amount_paise:** 167,700 → 1,677. The model omitted the ×100 conversion.
- **dv-00030, needs_human:** false → true. A prior ticket number caused another repeat-contact false positive.
- **dv-00037, priority/needs_human:** P1/true → P3/false. The ₹284,669 amount meets the P1 threshold.
- **dv-00042, category/channel:** other/null → refund/card. Card delivery was confused with a refund and payment channel.
- **dv-00044, priority:** P2 → P3. The ₹8,377 refund exceeds the P2 threshold.
- **dv-00046, amount_paise:** 53,700 → 537. The model omitted the ×100 conversion.
- **dv-00053, priority:** P2 → P3. The failed ₹9,601 transfer meets the P2 threshold.

With another week, I would add and review examples for amount conversion, priority boundaries, transaction IDs versus order/reference numbers, card delivery versus payment channels, and the repeat-contact rule. I would compare changes on dev before selecting a model.

## Test predictions and AI assistance

The selected clean rank-8 adapter generated predictions for all 400 test inputs. Test labels were not provided, so test accuracy is unknown. Structural validation found four schema-invalid values: an invalid date for `ts-00001`, a malformed transaction ID for `ts-00241`, and unsupported channel values for `ts-00367` and `ts-00386`. These raw outputs were left unchanged because post-generation field repair is prohibited.

OpenAI Codex helped inspect the data, write and debug code, and review errors. The experiment choices and final outputs were checked against the supplied schema and scorer.
