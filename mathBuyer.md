# 📊 Логика расчёта метрик — Buyer Analysis

## 1. Источники данных

| Файл | Что берём |
|------|-----------|
| **leads** (снимок 14:01) | affiliate_id, geo, Lead State, Sale Status, Injection, Rejection Reason, payout, revenue |
| **conversions** | UUID лидов с `Goal Type = ftd` |
| **sales** | UUID лидов с `Sale Status Registry = ftd` |

**FTD UUID** (CPA-конверсии — клиент сделал первый депозит) — объединение из трёх источников:
- leads → `Lead State = ftd`
- conversions → `Goal Type = ftd`
- sales → `Sale Status Registry = ftd`

---

## 2. Разделение на потоки

Каждый лид попадает **ровно в один** поток:

```
Лид пришёл
 ├─ Added To Injection = yes?  ──→ 🔵 INJECT
 ├─ Время в рабочих часах КЦ? ──→ 🟢 LIVE (основной)
 └─ Иначе                     ──→ ⚪ OFF-TIME (не считается)
```

### Рабочие часы КЦ (GMT+3)

| Гео | Часы | Гео | Часы |
|-----|------|-----|------|
| JP | 02–14 | CH | 10–21 |
| KR | 02–15 | FR | 11–20 |
| AU | 03–11 | GB | 11–20 |
| SG | 05–17 | IE | 12–20 |
| IT | 11–19 | AT | 12–19 |
| PT | 12–21 | NL | 13–20 |
| ES | 11–20 | NO | 12–21 |
| SE | 12–20 | DK | 12–19 |
| FI | 12–21 | CA | 18–02 (через полночь) |
| TW | 00–24 (круглосуточно) | | |

> Если гео нет в расписании → считаем как Live (не фильтруем).

---

## 3. Классификация Lead Sale Status

```
Sale Status → classify_status()
  ├─ пустой / "empty" / "none"  →  EMPTY
  ├─ совпал с VALID_KEYWORDS    →  VALID
  ├─ совпал с INVALID_KEYWORDS  →  INVALID
  └─ не совпал ни с чем         →  INVALID (fallback)
```

### VALID (~170 ключевых слов)

Любой контакт или попытка контакта:

| Категория | Примеры статусов |
|-----------|-----------------|
| FTD / Deposit | `ftd`, `deposit`, `depositor`, `deposited`, `converted` |
| Callback / Follow-up | `callback`, `call back`, `call again`, `follow up`, `recall`, `schedule` |
| New / Initial | `new`, `initial call`, `firstattempt`, `1stassign`, `newlead` |
| Reassign / Reshuffle | `reassign`, `reshuffle`, `shuffle`, `lead shuffle` |
| In Work / Processing | `in work`, `inwork`, `processing` |
| Hot | `hot`, `nahot`, `hotnaau`, `hot reassign g` |
| Contact attempts | `oncall`, `calling`, `dialing`, `speaking`, `busy` |
| No Answer | `no answer`, `noanswer`, `never answer` (все вариации) |
| Hung Up | `hungup`, `hung up`, `hang up`, `clienthungup` |
| Not Interested | `not interested`, `no interest`, `closed not interested` |
| Low / No Potential | `lowpotential`, `nopotential`, `not potential` |
| Voicemail | `voicemail`, `voice mail`, `dvm`, `directvm` |
| Denied Registration | `denied registration`, `deny registration`, `never registered` |
| No Money / No Funds | `no money`, `nomoney`, `nofunds`, `no funds` |
| Decline | `decline`, `declined`, `cc declined` |
| DNC / Unreachable | `donotcall`, `not reachable`, `unavailable` |
| Misc | `na`, `system`, `mixed`, `disqualified`, `slip away` |

### INVALID (~120 ключевых слов)

Только ошибки данных/гео — лид изначально нерабочий:

| Категория | Примеры статусов |
|-----------|-----------------|
| Wrong Number | `wrong number`, `wrongnumber`, `bad number`, `invalidnumber` |
| Wrong Info | `wrong info`, `wrongdetails`, `wrong data`, `invaliddata` |
| Wrong Person | `wrong person`, `wrongperson` |
| Language | `wrong language`, `invalid language`, `no language`, `language barrier` |
| Country | `wrong country`, `invalid country`, `forbiddencountry` |
| Immigrant | `immigrant`, `ukrainiancitizenship` |
| Age | `overage`, `underage`, `under 18`, `no age` |
| Trash / Test / Fake | `trash`, `test`, `test lead`, `fake`, `junk` |
| Duplicate | `duplicate`, `dublicate`, `cross lead`, `cross traffic` |
| Blacklist | `blacklisted`, `spam` |
| Disability | `disability`, `disable`, `unabletotalk` |
| Other company | `other company`, `madebyanotherplatform`, `called by others` |

### EMPTY

Лиды с пустым Sale Status (ещё не обработаны КЦ).  
Входят в `pushed`, влияют на знаменатель CR, но **не входят** в `total` (valid + invalid).

---

## 4. Подсчёт дублей

Дубли определяются **отдельно от Sale Status**, по колонке `Rejection Reason`:

```
Lead State = "lead-declined"
  └─ Rejection Reason содержит "DUPLICATION" или "DUPLICATE"
       └─ ДА → дубль
```

Дубли — лиды, отклонённые CRM до попадания к оператору. Они не имеют Sale Status.

> ⚠️ Статусы `duplicate` / `dublicate` из Sale Status → классифицируются как INVALID, но **не считаются дублями** в формуле CR.

---

## 5. Счётчики на баера (LIVE поток)

| Счётчик | Условие |
|---------|---------|
| `pushed` | `Lead State` ∈ {`lead-pushed` (CPL), `ftd` (CPA)} |
| `valid` | pushed + Sale Status → valid |
| `invalid` | pushed + Sale Status → invalid |
| `empty` | pushed + Sale Status → empty |
| `ftd` | UUID ∈ FTD_set **или** Lead State = `ftd` — это CPA-конверсии (клиент внёс депозит) |
| `declined` | `Lead State` = `lead-declined` |
| `duplicates` | declined + Rejection Reason содержит DUPLICATE |
| `payout` | сумма Lead Payout (только pushed) |
| `revenue` | сумма Lead Revenue (только pushed) |
| `margin` | `revenue − payout` (маржа на баера) |

> `pushed = valid + invalid + empty`

---

## 6. Формулы метрик

### Net CR (главная метрика)

```
CR = FTD / (pushed + declined - duplicates) × 100%
```

- **Числитель**: FTD — CPA-конверсии (клиент сделал первый депозит)
- **Знаменатель**: все лиды (CPL `lead-pushed` + CPA `ftd` + `lead-declined`) минус дубли
- Empty лиды входят в знаменатель через `pushed`

### Invalid Rate

```
invalid_rate = invalid / (valid + invalid) × 100%
```

Только среди размеченных лидов (empty не участвуют).

### Duplicate Rate

```
dup_rate = duplicates / (pushed + declined) × 100%
```

### Inject CR (отдельно)

```
inj_CR = inj_FTD / inj_valid × 100%
```

---

## 7. Scoring (0–100 баллов)

```
score = CR_score + Vol_score - Inv_penalty - Dup_penalty
```

| Компонент | Формула | Диапазон |
|-----------|---------|----------|
| CR_score | `min(CR, 60) / 60 × 50` | 0–50 |
| Vol_score | `log₁₀(volume + 1) / log₁₀(1000) × 30` | 0–30 |
| Inv_penalty | `invalid_rate / 100 × 10` | 0–10 |
| Dup_penalty | `dup_rate / 100 × 10` | 0–10 |

- `volume` = `valid + invalid` (total без empty)
- Если volume < 5 → score = 0
- Итоговый score обрезается до [0, 100]

---

## 8. Action Framework

| Net CR | Доп. условие | Действие |
|--------|-------------|----------|
| — | volume < 15 | **МАЛО ДАННЫХ** |
| > 10% | invalid_rate < 50% | **SCALE SPL** |
| ≥ 6% | — | **НАБЛЮДАТЬ** |
| 3–6% | volume ≥ 50 | **MOVE TO SPA** |
| 3–6% | volume < 50 | **WATCH** |
| < 3% | — | **STOP** |

---

## 9. Исключения (новые / тестовые)

Баер исключается из анализа, если выполняется **хотя бы одно**:

| Условие | Что значит |
|---------|----------|
| `aff_sub2` содержит `test` | Тестовый трафик |
| Первый лид в последние **3 дня** периода | Слишком новый баер |
| ≤ 5 pushed **и** ≤ 10 total | Слишком мало данных |

> ℹ️ Проверка идёт по **всем** лидам баера (кроме inject), без фильтрации по рабочим часам КЦ.
> `total` = все не-inject лиды, `pushed` = Lead State ∈ {lead-pushed, ftd}.

---

## 10. Тип лида (CPL / CPA)

**На уровне лида:**
- `Lead State = lead-pushed` → **CPL** (оплата за лид)
- `Lead State = ftd` → **CPA** (оплата за депозит/конверсию)

> Тип определяется **только** по `Lead State`, порог payout не используется.

**На уровне баера:**
```
lead_type = CPA, если CPA-лидов больше, чем CPL
lead_type = CPL, иначе
```

> ℹ️ `lead_type` вычисляется в данных баера, но пока не отображается в таблицах отчёта и CSV.

---

## 11. Geo Drill-Down

Для каждого баера считаются **те же метрики** в разрезе по гео:

- valid, invalid, ftd, dups, pushed, declined
- CR = FTD / (pushed + declined - dups)
- invalid_rate = invalid / (valid + invalid)
- Inject: inj_valid, inj_invalid, inj_ftd, inj_cr
- Top-3 рекламодателей на гео

---

## 12. Выходные файлы

| Файл | Содержимое |
|------|----------|
| `sheet1_live_leads.csv` | Buyer, Geo, Valid, Invalid, Total, FTD, CR%, Duplicates, Dup%, Advertiser, Score, Action |
| `sheet2_inject_leads.csv` | Buyer, Valid Inject, Invalid Inject, Total Inject, FTD Inject, CR Inject% |
| `sheet3_summary.csv` | Buyer, Total Leads, Total FTD, CR%, Invalid Rate%, Dup Rate%, Inject Share%, Score, Action |
| `buyer_report.html` | Интерактивный HTML-отчёт: KPI, Decision Table, Top/Bottom-20, Inject, все баеры с гео-детализацией, Geo Explorer, связки |

> ℹ️ `margin` и `lead_type` вычисляются в коде, но пока не выводятся в CSV/HTML.
