# Алиасы

[← 02 Основы](./02-network-fundamentals.md) · [Оглавление](./README.md) · [04 Day-1](./04-day1-walkthrough.md)

Полные строки для netshell / Alias Studio — только в [`alias-pack.txt`](./alias-pack.txt).

## Tiers в pack

| Tier | Назначение | Примеры |
|------|------------|---------|
| **Core** | `init_dc`, `init_f1`, `init_f2`, naming, DHCP helpers | `adebug`, `dhup`, `dhprod` |
| **Day-1** | dns-lite path | `ncall`, `dhdns2`, `pidns1`, `fcmal` |
| **Lab** | dns-server+padu path | `init_*`, `ripup`, `rip4` |
| **Advanced** | SFTP/DR, VLAN, RIP, pcap | `bkroutes`, `vshow`, `rip4` |

## Минимум для Day-1 (секции pack)

| Нужно | Алиасы |
|-------|--------|
| Старт | `setdbg`, `ncall`, `nca`, `rca`, `rcd`, `rcb`, `dmap`, `rsh`, `pdev` |
| Программы | `pidns1`, `pidns2`, `pstart`, `pview`, `pidhcp1`, `dhdns`, `dhdns2` |
| FW | `fcmal`, `fcall` / `fcsafe` |
| Lab | `adebug`, `init_dc`, `init_f1`, `init_f2`, `ripup`, `rip4`, `dhup`, `dhprod` |

Создание: вставь строку из pack целиком. Удаление: `alias имя`.

---

## Алиасы и примеры

**Канон строк:** [`alias-pack.txt`](./alias-pack.txt) — копируй оттуда; здесь только **как вызывать**. Удалённые из pack: `ra`/`rrm` (используй `rca`/`rcat`/`rr`).

### База: debugger, netaddr, просмотр

Примеры:

```text
setdbg 37157
ncall @c1/b1/f1/c1 @c1/b1/dns 12001
rsh @c1/b1
```

### Маршруты

Примеры под блок:

```text
rca @c1/b1/f1 0 @c1/b1
rca @c1/b1/f2 1 @c1/b1
rca @c1 7 @c1/b1
rcd 7 @c1/b1
rcat udp/67 3 @c1/b1
rcat udp/5060 2 @c1/svc
```

Смысл `rca`: `$1` = куда слать, `$2` = номер порта без слова port, `$3` = роутер.

### DNS

Пример:

```text
dmap voip.none @c1/svc/voip @c1/b1/dns
dmap git.none @c1/svc/git @c1/b1/dns
dmap instan_blind.xxx @c1/b1/f2/p1 @c1/b1/dns
```

Если `on $3` ругается без третьего аргумента — используй `dmapd` при уже выбранном DNS/`always on`.

### DHCP (нужен установленный dhcp-сервер)

Типичный блок:

```text
dhprefix @c1/b1/ @c1/b1/dhcp
dhdns @c1/b1/dns @c1/b1/dhcp
dhbind 55421 @c1/b1/f2/p1 @c1/b1/dhcp
rcat udp/67 4 @c1/b1
```

Day-1 с двумя DNS клиентам: `dhdns2 @c1/b1/dns @c1/dns @c1/b1/dhcp` (см. day-1 starter).

### Firewall — длинные пресеты и поштучно

**Важно:** `fcall` = whitelist. Порядок: allow’ы, **последним** `default deny`. Сначала всегда закладываем **tcp/23** (управление).

Пример: `fcall @c1/b1/fw` или `fcsafe @c1/svc/fw`. Поштучные правила — секция Firewall в [`alias-pack.txt`](./alias-pack.txt).

### Программы (install)

После install у тебя нет автозапуска — проверь сервис руками (`pstart` / `pview`).

### Скелет блока: имена edge/dns (ручной каркас)

Практика без мега-алиаса (надёжнее пошагово):

```text
ncall @c1/b1 @c1/b1/dns EDGE_HW
ncall @c1/b1/dns @c1/b1/dns DNS_HW
ncall @c1/b1/dhcp @c1/b1/dns DHCP_HW
pidns2 @c1/b1/dns
pidhcp1 @c1/b1/dhcp
dhprefix @c1/b1/ @c1/b1/dhcp
dhdns @c1/b1/dns @c1/b1/dhcp
```

Или одной строкой: `mkedge @c1/b1 @c1/b1/dns EDGE_HW` (см. pack).

### Бэкапы / DR (нужны sftp + желательно cron + try)

Если `try` в билде «неизвестная процедура» — не ставь `drtest`/`crping`, пока не появится; используй ручной `bkroutes`.

### QoL

`q`, `cls` — см. секцию QoL в [`alias-pack.txt`](./alias-pack.txt).

---
