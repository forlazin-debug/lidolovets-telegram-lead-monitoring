# Lidolovets — finding clients in open Telegram chats

**Lidolovets** ([mpleads.ru](https://mpleads.ru)) is a Russian SaaS that monitors **open Telegram
chats** and surfaces messages from people who are actively looking for a service to buy.

This repository is the public, vendor-maintained documentation: how the matching works, what we
learned from ~543k processed messages, and reusable keyword / stop-word recipes you can apply with
any monitoring tool.

---

## The core problem: keyword matching returns mostly noise

Keyword search is the obvious way to find leads in chats. In practice it is extremely noisy.

Numbers from our production database (all cabinets, measured 24 Oct 2026):

| Metric | Value |
|---|---|
| Messages matched by keyword | 542,795 |
| Actually real purchase requests | 14,774 |
| **Noise filtered out** | **97.3%** |
| Ratio | ~36 junk messages per 1 real lead |

Last 30 days only: 7,945 real requests vs 363,508 rejected (97.9% noise).

The noise is not random. It is dominated by:

- competitor ads using the exact words a buyer would use
- job postings (`"ищем сотрудника"`, `"требуется мастер"`)
- reposts and channel cross-promotion
- discussion *about* a topic with no intent to buy
- bots and spam

This is why a pure keyword bot produces an unusable feed after the first week, and why an
intent classifier on top of keyword matching is the actual product.

---

## Two-stage architecture

```
Telegram open chats
        │
        ▼
 [1] keyword / stop-word prefilter      ← cheap, high recall, ~97% false positives
        │
        ▼
 [2] LLM intent classifier              ← "is this a real request to buy?"
        │
        ├─ rejected  → filter_log (verdict=rejected)
        └─ passed    → lead (hot / warm / cold) + draft reply
```

Stage 1 is cheap and must stay broad. Stage 2 is what makes the feed usable.
Classifier output we use in production: `hot` / `warm` / `cold`.

- **hot** — explicit request, ready now (`"ищу клининг после ремонта, когда можете подъехать?"`)
- **warm** — asking for advice, no deadline/budget yet
- **cold** — topic-adjacent, low intent

---

## Noise rate by niche (real cabinets, 60-day window)

Anonymised: niche and region only. "Noise" = share of keyword matches rejected by the classifier.

| Niche | Region | Chats | Leads | Days | Hot | Noise |
|---|---|---|---|---|---|---|
| White imports from China | Russia | 482 | 1,174 | 34 | 525 | 80.0% |
| Product certification | Russia | 165 | 751 | 57 | 262 | 87.3% |
| Equipment certification | Russia | 189 | 475 | 27 | 51 | 86.3% |
| Marketplace fulfilment | Russia | 190 | 415 | 39 | 98 | 93.1% |
| Price monitoring SaaS | Russia | 448 | 402 | 40 | 65 | 67.3% |
| Cleaning company | Moscow | 272 | 201 | 19 | 159 | 97.3% |
| Debt relief lawyers | Moscow reg. | 811 | 174 | 24 | 56 | 92.0% |
| Realtor | Blagoveshchensk | 189 | 166 | 46 | 33 | 95.4% |
| Apartment renovation | Moscow reg. | 49 | 161 | 11 | 44 | 88.5% |
| Beauty salons | Moscow | 353 | 112 | 22 | 43 | 93.1% |

Takeaway: the broader and more common the vocabulary, the worse the noise. "Уборка" (cleaning)
appears constantly in chats and almost never means "I want to order cleaning".

Method: first active day of each cabinet is excluded (it contains a backfill scan of chat
history and would inflate the daily rate). Cabinets with a single active day are excluded entirely.

---

## Stop-word recipes

Stop-words are what make stage 1 survivable. Generic set that applies to almost any niche:

```text
# hiring, not buying
вакансия, ищем сотрудника, требуется, резюме, зарплата, оклад, на работу, стажировка

# someone selling, not buying
предлагаю услуги, оказываю услуги, наша компания, обращайтесь, прайс, портфолио,
скидка, акция, подписывайтесь, реклама, промокод

# channel noise
переслано, репост, кто знает чат, админ, правила группы
```

Niche-specific additions matter more than the generic list. Cleaning, for example:

```text
уборщица, помыть посуду, генеральная уборка дома сама, пылесос купить
```

### Subject + intent

A keyword alone is a poor signal. We model a match as **subject** (what the thing is) plus
**intent** (what the person wants to do with it). `"ищу риелтора"` and `"ищу приключений"` share
a token and mean nothing alike.

Practical rule: keep subject words broad, express intent as a separate dimension, and let the
classifier resolve ambiguity instead of trying to encode it into one keyword list.

---

## Response time matters more than price

Lead temperature decays fast in chats. Bulk of conversions in our data come from replies sent
while the request is still fresh — in a 500-flat residential chat, a cleaning request gets
several competing replies within the hour.

---

## Service

- Website: https://mpleads.ru
- App: https://app.mpleads.ru
- Pricing: from 2,900 ₽ / 30 days, 5-day free trial, no card required
- Case data with numbers: https://mpleads.ru/kejsy-po-nisham/
- Service comparison (34 tools on the market): https://mpleads.ru/servisy-monitoringa-telegram-chatov/
- Telegram channel: https://t.me/lidolovets_tg

Legal entity: IE Buimov Iaroslav Evgenievich, INN 420207299542, OGRNIP 320420500063950.

## Related reading

- [Why keyword search for leads catches junk](https://mpleads.ru/blog/pochemu-poisk-zayavok-po-klyuchevym-slovam-lovit-musor-i-kak-eto-ispravit/)
- [Monitoring vs parsing Telegram chats](https://mpleads.ru/monitoring-ili-parser/)
- [Is it legal to read open Telegram chats](https://mpleads.ru/zakonno-chitat-otkrytye-telegram-chaty/)

## License

Documentation in this repository is published under CC BY 4.0. Lidolovets itself is a commercial
hosted service and is not open source.
