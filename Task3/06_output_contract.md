# Выходной контракт

## Флагман: ответ Scoring + Decision

```json
{
  "operation_id": "string, обязательный, идемпотентный ключ",
  "scored_at": "ISO-8601, обязательный",
  "model_version": "string, обязательный",
  "feature_set_version": "string, обязательный",
  "score_raw": "number 0..1, обязательный",
  "score_calibrated": "number 0..1, обязательный",
  "decision_hint": "enum: review_high | review_normal | manual_only",
  "priority_rank": "integer >= 1",
  "top_factors": [
    {
      "feature": "string",
      "value": "string",
      "contribution": "number"
    }
  ],
  "rule_hits": [
    {
      "rule_id": "string",
      "severity": "enum"
    }
  ],
  "guardrails": {
    "status": "enum: ok | fail",
    "reasons": [
      "string"
    ]
  },
  "fallback": {
    "triggered": "boolean",
    "reason": "enum: stale_features | no_history | out_of_distribution | abstention_zone | rule_model_disagreement | model_unavailable | feature_service_unavailable | contract_error | limits_exceeded"
  },
  "source_refs": [
    {
      "type": "core_txn | kyc | rule",
      "ref": "string"
    }
  ],
  "data_freshness": {
    "features_age_sec": "integer",
    "is_stale": "boolean"
  },
  "contract_version": "string"
}
```

### Детерминированные проверки до записи в очередь

1. **Валидация по JSON-схеме.** Если отсутствует обязательное поле или контракт не проходит схему — **рекомендация
   модели отклоняется**, но **исходная операция явно сохраняется в штатной очереди** с приоритетом по правилам и сумме.
   Операция не теряется и не выпадает из разбора. В очередь пишется запись с `fallback.reason = contract_error`,
   `fallback.triggered = true`, исходным `operation_id` и `source_refs` из сырых данных.
2. `score_calibrated` в [0,1]; `priority_rank` ≥ 1.
3. `guardrails.status = fail` → `fallback.triggered = true` обязателен.
4. Если `fallback.triggered = true`, `decision_hint` не может быть `review_high` без пояснения.
5. `source_refs` непустой для каждой операции; иначе — `contract_error` и та же логика сохранения в штатной очереди (не
   отбрасывание).
6. `is_stale = true` → приоритет не выше `review_normal`, кейс помечается для человека.

### Поведение при отсутствии данных

| Ситуация                                                   | Поведение                                                                                                                                                              |
|------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Нет `operation_id`                                         | Это единственный случай, когда операцию нельзя сохранить в очередь — без идентификатора её невозможно связать с исходной. Отказ, лог, метрика, алерт в мониторинг.     |
| Нет части признаков                                        | `fallback.reason = stale_features/no_history`; приоритет по правилам и сумме; кейс идёт в ручной разбор. **Операция в очереди сохраняется.**                           |
| Ошибка контракта (не прошла схема, нет обязательного поля) | `fallback.reason = contract_error`; рекомендация модели отклоняется; **операция сохраняется в штатной очереди** с приоритетом по правилам и сумме; алерт в мониторинг. |
| Модель недоступна                                          | `model_unavailable`; очередь строится правилами; инцидент в мониторинг.                                                                                                |
| Feature Store недоступен                                   | `feature_service_unavailable`; правила по сырым полям из АБС.                                                                                                          |
| Вне области применимости                                   | `out_of_distribution`; ручной разбор.                                                                                                                                  |

### Валидация до бизнес-действия

Никакой автоматической блокировки. Бизнес-действие (эскалация, отказ, снятие) выполняет сотрудник по процедуре. Контракт
валидируется **до** постановки в очередь. Если контракт невалиден — модель не влияет на приоритет, но операция остаётся
в очереди по правилам. Невалидный контракт не попадает в рабочее место как рекомендация модели, но сама операция —
попадает.

## №18: контракт карточки

```json
{
  "application_id": "string",
  "document_id": "string",
  "document_type": "enum",
  "fields": [
    {
      "name": "string",
      "value": "string|number",
      "confidence": "number 0..1",
      "page": "integer",
      "bbox": [
        0,
        0,
        0,
        0
      ],
      "validation": {
        "status": "ok|failed",
        "checks": [
          "string"
        ]
      }
    }
  ],
  "missing_required": [
    "string"
  ],
  "cross_document_conflicts": [
    "string"
  ],
  "requires_human": "boolean",
  "fallback": {
    "triggered": "boolean",
    "reason": "enum: contract_error | ocr_failed | llm_unavailable | out_of_scope"
  },
  "model_version": "string"
}
```

Проверки до записи в карточку: обязательные поля, арифметические контроли, сверка со справочником форм.
`missing_required` непустой → `requires_human = true`; документ **не отбрасывается**, а уходит андеррайтеру на ручной
разбор с пометкой. Критическое поле с низкой уверенностью не может быть записано без подтверждения.

## №5: контракт маршрутизации

```json
{
  "ticket_id": "string",
  "predicted_department": "string",
  "top3": [
    {
      "department": "string",
      "probability": "number"
    }
  ],
  "confidence": "number 0..1",
  "route": "enum: auto | manual_sort",
  "rule_override": "boolean",
  "fallback": {
    "triggered": "boolean",
    "reason": "enum: contract_error | unknown_department | empty_text | model_unavailable"
  },
  "model_version": "string"
}
```

Проверки: `predicted_department` из справочника; `confidence < threshold` → `route = manual_sort`; правило канала может
переопределить модель с `rule_override = true`. При `contract_error` тикет **не теряется**, а уходит в `manual_sort` с
пометкой.

