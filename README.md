# AXENIX_HSE_CASE — AI‑сервис генерации презентаций для российских вузов (B2B2C, Track 2: Prompt Engineering)

Проект: **“Brand‑Locked AI”** — сервис, который превращает текст/источники в **брендированные** академические презентации (PPTX/PDF), **не позволяя “сломать”** стиль вуза (brand‑lock), и даёт вузу управляемость (RBAC, орг‑единицы, дашборды, аудит).

---

## TL;DR для жюри (в каком порядке смотреть)

1) **PRD** — что и зачем строим: `PRD_AXENIX.pdf`  
2) **SRS** — требования/стори/AC/NFR: `SRS_AXENIX.pdf`  
3) **System Architecture** — контейнеры/флоу/безопасность: `System_Architecture_AXENIX.pdf`  
4) **Prompt Architecture** — P0–P8, JSON‑схемы, guardrails, rubric: `Prompt_Architecture_AXENIX.pdf`  
5) **API Spec** — REST контракты MVP + идемпотентность/лимиты: `API_Spec_AXENIX.pdf`  
6) **QA Plan** — quality rubric 20 баллов, quality gate, тест‑брифы, метрики: `QA_Plan_AXENIX.pdf`  
7) **UI/UX** — экраны продукта: `UI.pdf`

---
## Диаграмма схема логики промптинга команды
<img width="991" height="641" alt="Диаграмма без названия-Страница-1 drawio" src="https://github.com/user-attachments/assets/e19da983-ce53-43e7-9f01-1babe16d32ca" />

---
## 1) Проблема и решение (коротко)

### Проблема
В вузах презентации делаются в разрозненных инструментах (PowerPoint/аналогах), из‑за чего:
- брендбук не соблюдается, качество нестабильно;
- нет единой отчётности и метрик цифровизации;
- зарубежные AI‑сервисы зачастую не подходят для институциональной закупки (комплаенс/юрисдикция/договоры).

### Решение
Единая институциональная платформа:
- **SSO вход** (студенты/преподаватели);
- генерация презентации **строго из источников** (anti‑hallucination);
- **brand‑lock**: шрифты/цвета/логотипы/макеты защищены;
- **частичная регенерация** (правим один слайд, не пересобирая всю деку);
- **экспорт PPTX/PDF** + QA‑гейт;
- админ‑контур: шаблоны/брендбук, RBAC + орг‑единицы, дашборды, аудит.

---

## 2) End‑to‑End (как продукт работает)

### 2.1 Student / Teacher (основной пользовательский поток)
1. **SSO вход** → попадаю в пространство тенанта (вуз).
2. **Создать презентацию** → заполняю brief (тема, цель, аудитория, время, кол‑во слайдов, тон).
3. **Источники**: добавляю текст/таблицу/ссылку (факты разрешены только из источников).
4. Если данных не хватает — **модал уточнений** (до 7 вопросов) с дефолтами.
5. **Генерация**: пайплайн P0–P7 (прогресс/ETA).
6. **Редактор**: вижу слайды, пометки `NEEDS_SOURCE`, QA‑оценку, issues.
7. **Правка слайда**: “изменить по инструкции” → preview diff → apply (P8).
8. **Экспорт**: доступен только при прохождении quality gate (QA>=порог и без блокеров); если есть `NEEDS_SOURCE` — предупреждение.
9. Получаю **PPTX/PDF**, открываемый без критических артефактов.

### 2.2 Admin / Buyer (управление и закупка)
- Admin загружает **брендбук/шаблон**, включает `Brand‑lock: strict`, настраивает org‑units.
- Admin смотрит **дашборды**: BPC, activation, time‑to‑first‑deck, export conversion, качество.
- Admin просматривает **audit log**: login/generate/edit/export/template_update.
- Buyer использует демо/документы для оценки (упрощение закупки).

---

## 3) Архитектура (высокоуровнево)

Ключевая идея: **Orchestrator** управляет состояниями Deck/Run/Export и пайплайном; **LLM Gateway** — единая точка вызова моделей (валидация JSON, лимиты, redaction); **Renderer** собирает PPTX/PDF из `render_spec`.

### C4 L2 (Containers) — кратко
- **UI (Web App)**: brief‑форма, прогресс, редактор, admin‑панель.
- **Orchestrator API**: Deck/Run/Export, RBAC, политики, идемпотентность.
- **LLM Gateway**: routing по моделям, schema validation, cost control, PII redaction.
- **Storage**: Postgres (сущности/версии/аудит), S3/MinIO (артефакты JSON, exports, шаблоны).
- **Renderer**: генерация PPTX/PDF.
- **Asset Service**: иконки/графики (из таблиц).
- **Observability**: метрики/логи/трейсы + audit.

### Диаграммы


Подробно: `System_Architecture_AXENIX.pdf`

---

## 4) Prompt‑архитектура (P0–P8) — “сердце” трека

**Цель:** воспроизводимая генерация через строго структурированные JSON‑артефакты + проверка качества.

- P0: normalize brief → `brief_v1`
- P1: clarifying questions (до 7) → `clarifications`
- P2: 3 storyline options → `storyline_selected`
- P3: slide blueprint → `blueprint`
- P4: copy + speaker notes → `slides_draft`
- P5: visual plan (icons/charts from sources) → `visual_spec`
- P6: QA rubric + auto‑fix → `qa_report` + `slides_fixed`
- P7: render_spec для PPTX/PDF → `render_spec`
- P8: per‑slide patch (diff) → `slide_patch`

Guardrails:
- **anti‑hallucination**: “факты только из источников”, иначе `NEEDS_SOURCE`;
- **brand‑lock**: запрещены изменения locked‑параметров (шрифты/цвета/логотип/макеты);
- **length limits**: буллеты ≤12 слов, 3–5 буллетов, 1 takeaway на слайд;
- schema validation на каждом шаге.

Подробно: `Prompt_Architecture_AXENIX.pdf`

---

## 5) API (MVP) — quick reference

Базовый префикс: `/api/v1`

### Endpoints
- `POST /brief/normalize` — сырой бриф → `brief_v1` + missing_info + questions
- `POST /deck/generate` — запускает пайплайн P0–P7 и возвращает `deck_id`/статус
- `POST /deck/{id}/edit` — частичная регенерация (P8), поддержка `base_version_id`
- `GET  /deck/{id}` — получить деку/статус/версии/QA
- `POST /deck/{id}/export` — экспорт PPTX/PDF (job/cached), quality gate

### Идемпотентность и ретраи (MVP defaults)
- Для POST, где это важно: `Idempotency-Key` обязателен (должен возвращать тот же `run_id`/`export_id` при повторе).
- Ретраи клиента: 429/503/504 с exponential backoff + jitter.
- Экспорт кэшируется по (deck_id, version_id, format, options_hash).

Полные контракты: `API_Spec_AXENIX.pdf`

---

## 6) Качество и тестирование

### 6.1 Quality Rubric (20 баллов)
10 критериев × 0/1/2 = 20. Применяется на шаге P6 (QA & Fix).  
Ключевые: flow, заголовки, краткость буллетов, takeaway, термины, `NEEDS_SOURCE`, brand compliance, export readiness.

### 6.2 Quality Gate (перед экспортом)
Экспорт разрешён, если:
- `Total >= 14/20`
- **нет блокеров** по ключевым критериям (anti‑hallucination / brand‑lock / export readiness).

### 6.3 Тестовые брифы (минимум 5, включая edge)
См. `QA_Plan_AXENIX.pdf` (включая: missing info, lecture from notes, strict brand‑lock попытка, contradicting constraints, chart binding, concurrency conflict).

### 6.4 Логирование для воспроизводимости
Логируем:
- на каждый `GenerationRun`: tenant_id, user_id, role, org_unit, promptset_version;
- на каждый шаг P0–P8: prompt_id/version, model_id, input_hash/output_hash, artifact_ref (S3), tokens, cost, latency;
- audit log generate/edit/export (append‑only);
- redaction ПДн в логах, запрет логировать секреты.

Подробно: `QA_Plan_AXENIX.pdf`

---

## 7) Безопасность и комплаенс (MVP)

- **Tenant isolation**: разделение данных по tenant_id (DB + S3 prefixes/buckets).
- **RBAC + org_unit**: права по ролям + видимость по орг‑единицам.
- **PII minimization**: хранить минимум атрибутов; маскирование при логировании.
- **Secrets management**: ключи LLM/интеграций — только в secret store; не логировать.
- **Audit log**: неизменяемые записи ключевых событий (login/generate/edit/export/template_update/role_change).

---

## 8) Состав репозитория (все файлы и зачем)

### 8.1 Финальные документы (рекомендуемые для показа жюри)
| Файл | Зачем нужен |
|---|---|
| `PRD_AXENIX.pdf` | Финальный PRD: проблема, пользователи/JTBD, value prop, use cases, KPI, MVP vs future, риски. |
| `SRS_AXENIX.pdf` | Финальный SRS: глоссарий, user stories, acceptance criteria, ограничения, NFR. |
| `System_Architecture_AXENIX.pdf` | Финальная системная архитектура: C4 L1/L2, sequence flows, технологии, безопасность. |
| `Prompt_Architecture_AXENIX.pdf` | Финальная prompt‑архитектура: P0–P8, JSON‑схемы, guardrails, HITL, rubric. |
| `API_Spec_AXENIX.pdf` | Финальный REST API контракт MVP: request/response/error/idempotency/limits. |
| `QA_Plan_AXENIX.pdf` | Финальный QA Plan: rubric 20, gate, тест‑брифы, логирование, прод‑метрики. |
| `UI.pdf` | Пак UI/UX экранов (Student/Teacher/Admin) для демо и защиты UX части. |



### 8.2 UI‑ассеты (PNG) — отдельные экраны 
| Файл | Экран |
|---|---|
| `a_screenshot_of_an_educational_saas_platform_login.png` | Login / SSO |
| `a_screenshot_of_a_web_based_russian_language_prese.png` | Student: “Мои презентации” |
| `a_screenshot_of_a_web_application_interface_shows.png` | Student: “Создать презентацию” (форма + sources + preview) |
| `a_screenshot_of_an_ai_powered_presentation_creatio.png` | Модал уточнений (до 7 вопросов) |
| `a_screenshot_of_an_online_presentation_generation.png` | Экран генерации (progress/ETA; может содержать промежуточное состояние) |
| `a_screenshot_showcases_an_ai_powered_presentation.png` | Главный редактор деки (thumbnails + canvas + QA панель) |
| `a_screenshot_of_a_slide_editing_interface_designed.png` | “Изменить по инструкции” + diff preview |
| `a_digital_screenshot_of_an_export_modal_window_wit.png` | Модал экспорта + quality gate + предупреждения |
| `a_screenshot_of_an_administrator_settings_page_tit.png` | Admin: “Брендбук и шаблоны” |
| `a_digital_screenshot_of_an_administrative_analytic.png` | Admin: “Дашборды” |
| `a_screenshot_of_an_audit_log_page_within_an_admini.png` | Admin: “Аудит” |

### 8.3 Исследования/входные материалы (TXT) — источники для PRD/позиционирования
| Файл | Что внутри / зачем |
|---|---|
| `product_doc_claude_sonnet 4.6.txt` | Сырой продуктовый документ/заметки (input для PRD). |
| `v1_functions_claude_sonnet 4.6.txt` | Список функций MVP/Future (input для scope). |
| `risks_claude_sonnet 4.6.txt` | Риски/assumptions (input для PRD/risks). |
| `value_propose_ sonnet 4.6.txt` | Наброски value proposition/дифференциаторов. |
| `competitors_claude sonnet 4.6.txt` | Анализ конкурентов (черновой). |
| `Competitors analysis chatgpt.txt` | Альтернативный анализ конкурентов (для сверки). |
| `Pains&Needs chatgpt.txt` | Pains/Needs B2C/B2B (JTBD input). |
| `cjm_claude_sonnet 4.6.txt` | CJM/пользовательские ожидания/триггеры. |
| `arch_claude_sonnet 4.6.txt` | Черновая архитектурная аналитика (input). |
| `Market analysis PAM, TAM, SAM, SOM google ai gemini.txt` | Рыночные оценки/структура рынка (если нужно для pitch). |

### 8.4 Промпт‑пак и материалы по “промптингу” (для Track 2)
| Файл | Зачем нужен |
|---|---|
| `main_decomposition_chatgpt5.2pro_request.txt` | Исходный запрос на декомпозицию (как показывать “исходные запросы”). |
| `main_decomposition_chatgpt5.2pro_response.txt` | Ответ‑декомпозиция: как упаковать Prompt Pack, структуры промптов и т.д. |
| `slides_decomposition.pdf` | Набор системных промптов для генерации **презентации о проекте** (pitch deck): нормализация брифа, storyline, blueprint и т.п. |
| `AXENIX_Hackathon_PromptEngineering_Report.docx` | Итоговый отчёт по prompt‑engineering (консолидация подхода/шагов/аргументов). |
| `product_doc.docx` | Сборный продуктовый документ (источник/черновик). |


---



## 10) Команда / статус
- Статус: **документация MVP завершена** (PRD/SRS/Architecture/Prompts/API/QA/UI).

---

### Лицензии
- UI‑скриншоты — **макеты**, без реальных брендов/логотипов, с заглушками персональных данных.
