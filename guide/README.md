# Гайд по Tower Networking Inc.

Единый пошаговый гайд для игры. **Не входит в scope Mod Manager** ([ADR-002](../docs/decisions.md)), но лежит в форке для удобства.

**Канон строк алиасов:** [`alias-pack.txt`](./alias-pack.txt) — полные `alias …` только там; в главах — примеры вызова и таблицы портов.

Справочник команд игры (не алиасы): [`../game-command-ref/`](../game-command-ref/).

---

## Две линии DNS (не путать)

| Путь | Глава | DNS на этаже | Upstream |
|------|-------|--------------|----------|
| **Day-1 lite** | [04-day1-walkthrough.md](./04-day1-walkthrough.md) | f1: `dns-lite`+`dnsmasq`; f2/f3: `dns-lite` | Клиентам `dhdns2`: блок + `@c1/dns` |
| **Lab** | [05-lab-walkthrough.md](./05-lab-walkthrough.md) | `dns-server`+`padu` + `dnsmasq` | f1 → `@c1/dns`; f2 → `@c1/b1/f1/dns` |

Оба пути валидны под разные пресеты. Строки алиасов — только из [`alias-pack.txt`](./alias-pack.txt).

---

## Быстрый старт

| Цель | С чего начать |
|------|----------------|
| **Классический Day-1** (ЦОД + 3 этажа, без читов) | [01 Введение](./01-introduction.md) → [02 Основы](./02-network-fundamentals.md) → [03 Алиасы](./03-aliases.md) → [04 Day-1 walkthrough](./04-day1-walkthrough.md) |
| **Lab Mode** (песочница, `init_dc` / `init_f1`) | [01 Введение](./01-introduction.md) → [03 Алиасы](./03-aliases.md) → [05 Lab walkthrough](./05-lab-walkthrough.md) |
| **Рост башни / справочник** | [06 Масштабирование](./06-scaling-and-expansion.md) · [07 Справочник](./07-advanced-reference.md) |
| **Что-то сломалось** | [08 Troubleshooting](./08-troubleshooting.md) |

1. Скопируй нужные секции из [`alias-pack.txt`](./alias-pack.txt) в netshell или **Alias Studio** в Mod Manager.
2. Lab после патча этажа: `init_fN …`, затем `rip4 @c1/b1/f2 @c1/b1/f1 @c1/b1 @c1` (или по одному `ripup`).
3. Day-1: минимум `setdbg`, `ncall`, `rca`/`rcd`, `dmap`, `pidns1`, `fcmal` — см. [04-day1-walkthrough.md](./04-day1-walkthrough.md).

---

## Оглавление

- [01-introduction.md — Введение](./01-introduction.md)
  - [Что это за игра (30 секунд)](./01-introduction.md#что-это-за-игра-30-секунд)
  - [Имена (заучи сразу)](./01-introduction.md#имена-заучи-сразу)
  - [Цвета кабелей](./01-introduction.md#цвета-кабелей)
  - [Список покупок на старт](./01-introduction.md#список-покупок-на-старт)
  - [Пресет Lab Mode](./01-introduction.md#пресет-lab-mode)
- [02-network-fundamentals.md — Основы сети](./02-network-fundamentals.md)
  - [Твои настройки (сводка)](./02-network-fundamentals.md#твои-настройки-сводка)
  - [Большая картина](./02-network-fundamentals.md#большая-картина)
  - [Как устроена сеть в двух словах](./02-network-fundamentals.md#как-устроена-сеть-в-двух-словах)
  - [Анатомия блока: цепочка этажных роутеров](./02-network-fundamentals.md#анатомия-блока-цепочка-этажных-роутеров)
  - [Два режима: минимум сейчас vs «не перестраивать потом»](./02-network-fundamentals.md#два-режима-минимум-сейчас-vs-не-перестраивать-потом)
  - [Режим A — минимальная комплектация (день 1–несколько этажей)](./02-network-fundamentals.md#режим-a-минимальная-комплектация-день-1несколько-этажей)
  - [Режим B — нормальная эксплуатация (блоки по 3)](./02-network-fundamentals.md#режим-b-нормальная-эксплуатация-блоки-по-3)
  - [Несколько ЦОД (второй и дальше)](./02-network-fundamentals.md#несколько-цод-второй-и-дальше)
  - [Когда этажей станет много — да, упрёшься в ПС](./02-network-fundamentals.md#когда-этажей-станет-много-да-упрёшься-в-пс)
  - [Сводка: ЦОД vs этаж vs блок](./02-network-fundamentals.md#сводка-цод-vs-этаж-vs-блок)
  - [Prefix routing (кратко)](./02-network-fundamentals.md#prefix-routing-кратко)
  - [Как это работает (подробно)](./02-network-fundamentals.md#как-это-работает-подробно)
- [03-aliases.md — Алиасы](./03-aliases.md)
  - [Алиасы и примеры](./03-aliases.md#алиасы-и-примеры)
- [04-day1-walkthrough.md — Day-1 walkthrough](./04-day1-walkthrough.md)
  - [Порядок подключений: что к чему](./04-day1-walkthrough.md#порядок-подключений-что-к-чему)
  - [Шаг 0–10 (пошагово)](./04-day1-walkthrough.md#шаг-0-осмотрись)
- [05-lab-walkthrough.md — Lab walkthrough](./05-lab-walkthrough.md)
  - [Пресет Lab Mode](./05-lab-walkthrough.md#пресет-lab-mode)
  - [ИТОГО: ЦОД под оптику](./05-lab-walkthrough.md#итого-цод-сразу-под-оптику-без-переделки-розеток)
  - [Канон прогона Lab](./05-lab-walkthrough.md#канон-прогона-lab-чеклист-одной-страницей)
  - [Этаж 1 / Этаж 2](./05-lab-walkthrough.md#этаж-1-патч-netshell-факт)
- [06-scaling-and-expansion.md — Масштабирование](./06-scaling-and-expansion.md)
  - [Обзор приоритетов](./06-scaling-and-expansion.md#обзор-приоритетов)
  - [b2, FW, sftp, Jailbreak, Git, VLAN, RIP, второй ЦОД](./06-scaling-and-expansion.md#1-новый-блок-b2-этажи-46)
- [07-advanced-reference.md — Справочник](./07-advanced-reference.md)
  - [Registry, телефоны, VLAN, RIP, pcap, порты, Tower Link](./07-advanced-reference.md#registry-ppu-и-деньги-isp)
  - [Источники](./07-advanced-reference.md#источники)
- [08-troubleshooting.md — Troubleshooting](./08-troubleshooting.md)
  - [Чеклист «дырок»](./08-troubleshooting.md#что-ещё-легко-забыть-чеклист-дырок)
  - [Если что-то не работает](./08-troubleshooting.md#если-что-то-не-работает)
  - [Частые ошибки](./08-troubleshooting.md#частые-ошибки-на-твоём-пресете)

---

## Связь с Mod Manager

- Autocomplete в Alias Studio: `alias_helper_catalog.json` (команды игры), **не** `alias-pack.txt`.
- Установленные алиасы пишутся в `settings.json` userdata игры.
