# Task3. Разработка архитектуры ИИ-сервиса

## Состав директории

| Файл                               | Что внутри                                                                                 |
|------------------------------------|--------------------------------------------------------------------------------------------|
| `01_c4_container_flagman.md`       | C4 Container для флагмана (№1): контейнеры, роли, режим обработки, границы доверия TB1–TB7 |
| `02_c4_component_flagman.md`       | C4 Component для `Decision & Priority Service`: где код, где модель, почему нет retrieval  |
| `03_c4_container_documents.md`     | C4 Container для №18: извлечение показателей из документов                                 |
| `04_c4_container_routing.md`       | C4 Container для №5: маршрутизация обращений                                               |
| `05_component_responsibilities.md` | Таблица ответственности: код / модель / очередь / человек                                  |
| `06_output_contract.md`            | Выходной контракт, детерминированные проверки, поведение без данных                        |
| `07_fallback_and_automation.md`    | Уровень автоматизации и фолбэк-триггеры                                                    |
| `08_monitoring.md`                 | План мониторинга: вход, модель, очередь, бизнес-метрика                                    |
| `09_trust_boundaries_threats.md`   | Границы доверия → угрозы → меры                                                            |
| `diagrams/*.puml`                  | PlantUML-диаграммы C4                                                                      |

## Диаграммы

Файлы PlantUML в `diagrams/`:

- `01_container_flagman.puml` — Container, флагман
- `02_component_flagman.puml` — Component, `Decision & Priority Service`
- `03_container_documents.puml` — Container, №18
- `04_container_routing.puml` — Container, №5
- `05_trust_boundaries_flagman.puml` — карта границ доверия флагмана

