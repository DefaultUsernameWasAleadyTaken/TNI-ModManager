# Справочник: продвинутые темы

[← 06 Масштабирование](./06-scaling-and-expansion.md) · [Оглавление](./README.md) · [08 Troubleshooting](./08-troubleshooting.md)

Registry, VLAN, RIP, диагностика, бэкапы, Morris.

---

## Гарантия, смерть железа и замена

После окончания **warranty** устройство может сломаться. Конфиги (routes, netaddr, firewall rules, программы) живут на железе — без бэкапа замена = настройка с нуля, а при блочной схеме это больно, но локализуемо.

### Что держать в запасе (режим B)

| Запас | Зачем |
|-------|--------|
| 1× edge-роутер того же класса | Быстрая замена `@c1/bN` |
| 1× Blade5 | Умер switch этажа |
| 1× Boulder/Boulder+ | DNS/DHCP/money |
| Кабели нужных длин + питание | Не ждать доставку в аварии |
| Datawiper USB | Сброс firewall, если сам себя отрезал |
| Место под NAS / storage | Цель для sftp-бэкапов |

Имена `@c1/bN/…` при замене **не меняй** — повесь тот же netaddr на новое железо (`ncall` / DHCP bind), восстанови конфиг из бэкапа. Маршруты соседей и DNS map остаются валидны.

### Порядок замены вышедшего из строя устройства

1. **Изолируй** (вынь uplink’и), чтобы не плодить хаос / не заразить чистое.  
2. Запиши порты и цвета кабелей (фото / заметка) — куда что было воткнуто.  
3. Поставь новое железо, питание, те же кабели в те же порты по возможности.  
4. Debugger → `ncall @старое_имя @dns HARDWARE_ID` (тот же `@`).  
5. Восстанови конфиг из бэкапа (`sftp cp` обратно) **или** вручную: routes / firewall / programs.  
6. Верни линки, `ping`/`trace` от клиентов блока и из ядра.  
7. Если это был DNS/DHCP блока — проверь, что клиенты снова получают адрес/резолв; money-домены на DNS на месте.

**Приоритет восстановления:** ядро/`@c1/svc` → edge мёртвого блока → DNS/DHCP блока → этажные switch → клиенты. Остальные блоки можно не трогать.

### Пока sftp ещё не открыт

Secretariat → proposal **Remote Backups** (в гайдах ~450$, «3-2-1 let's back it up!») даёт команду `sftp`. До unlock:

- Документируй имена, порты, `route show`, список `dns map`, алиасы.  
- Не экономь на сроках warranty критичных роутеров/DNS.  
- Блочная схема всё равно спасает: умер b2 — b1 и svc живут.

---

---

## Бэкапы (sftp)

### Требования

- Unlock **Remote Backups** → `sftp` в netshell.  
- Источник: роутер / firewall / сервер с конфигом.  
- Приёмник: устройство со **свободным storage** (NAS, запасной Boulder и т.п.).

### Что бэкапить в первую очередь

| Устройство | Типичные пути / смысл | Зачем |
|------------|----------------------|--------|
| Роутер | `/etc/routes.conf` (и родственные конфиги) | Вся таблица маршрутов блока/ядра |
| Firewall | `/etc/nftables.conf` | Правила; можно копировать FW→FW |
| Сервер | программы + свои конфиги | DNS/DHCP/money не собирать заново |
| Полезный трюк | `sftp cp … from @fw1 to @fw2` | Клон правил на запасной FW |

Точные имена файлов смотри через `sftp ls on @device` и `man sftp` — сборки отличаются.

### Ручной бэкап (идея)

```text
sftp ls on @c1/b1
sftp cp /etc/routes.conf on @c1/b1 to @nas rename /backups/c1-b1/routes.conf
```

Для firewall (из гайда по FW):

```text
sftp cp /etc/nftables.conf from @fw1 to @fw2
```

### Автобэкап через cron (когда cron открыт)

Строки `drstart`, `drtest`, `crdr`, `crping` — секция SFTP/DR в [`alias-pack.txt`](./alias-pack.txt) (идея Hitchhiker: разбудить NAS → скопировать → усыпить).

Пример смысла: `crdr routes @c1/b1 @nas` — раз в час бэкап конфига роутера на NAS. Имя файла без `.conf` в аргументе — алиас дописывает сам (как в гайде).

Мониторинг живости: `crping @c1/b1` (см. pack).

### Восстановление из бэкапа

1. Новое/чистое устройство с тем же `@`.  
2. `sftp cp` **с NAS на устройство** (обратное направление к бэкапу).  
3. Перезапуск/проверка `route show` / `firewall show` / сервисов.  
4. Не подключай «грязный» линк к чистому ядру, пока не уверен.

Правило **3-2-1** по духу proposal: копия не только «рядом с роутером», а на отдельном storage; лучше две цели (основной NAS + запас), когда появится место.

---

---

## Morris, firewall и «отрезал сам себя»

### Firewall

- Из коробки FW почти прозрачный (default allow) — от Morris почти не спасает.  
- Для режима B: **default deny** + явный allow нужных портов; сначала разреши **tcp/23** себе (управление), иначе залочишься.  
- Залочился → **Datawiper USB** = factory reset firewall.  
- После sftp можно клонировать `nftables.conf` на второй FW — держи прогретый запас.

Минимум deny (если ещё не готов полный whitelist): scraper `tcp/8034`, Morris `tcp/510–519` на пути к роутерам/серверам (см. Hitchhiker `fcmal` / Steam Firewalls guide).

### Чистка Morris (кратко)

1. Определи заражённое; **отключи от сети**.  
2. Сервер: `program list` → `program uninstall morris…`  
3. Роутер: `sftp ls` → `sftp rm /bin/morris…` (нужен sftp).  
4. Собирай снизу вверх: серверы → их роутер → выше. **Не** втыкай заражённый роутер в чистое ядро.  
5. Без sftp восстановление роутеров после Morris сильно хуже — ещё один аргумент за Remote Backups.

Блочная схема: заразился b2 — режешь uplink b2, чистишь блок, svc и b1 продолжают зарабатывать.

---

---

## Ещё полезное под долгую башню (краткий указатель)

Ниже по файлу — развёрнутые главы: Registry/ISP, телефоны и камеры, VLAN, RIP, pcap, blackhole, сводная таблица портов, Tower Link / Socketeer.

Кратко:

| Тема | Суть |
|------|------|
| **Несколько ЦОД** | New Data Center (+10% admin); `@c2` + линк к `@c1`; edge верхних блоков выше |
| **Рост / ПС** | Блоки + апгрейд Tower Link + не один Blade на всё; см. главу выше |
| **Цвета кабелей** | Скорость не зависит; синий патч / жёлтый uplink / оранжевый ствол / красный money |
| **RIP** | Unlock → advertise/listen; endpoint’ы вручную, середина сама |
| **Питание** | Платное; не сажай весь ЦОД на одну цепь |
| **HA / LB** | Пока = блоки + запас + бэкапы + второй ЦОД как площадка |
| **Документируй порты** | Riser, цвета, какой порт ядра = какой блок / какой ЦОД |
| **Алиасы** | Бэкапь `settings.json` / Alias Studio |

### Мини-чеклист «готов к аварии»

- [ ] Remote Backups открыт, `sftp` работает  
- [ ] Есть NAS/storage под `/backups/…`  
- [ ] Свежий бэкап routes (+ nftables) каждого edge и ядра  
- [ ] Запасной роутер/switch/сервер или план заказа  
- [ ] Datawiper под рукой  
- [ ] Записаны порты/riser/цвета  
- [ ] Знаешь, как вырезать один блок без даунтайма svc  

---

---

## Registry, PPU и деньги (ISP)

Связь этажей сама по себе не кормит башню. Деньги идут, когда **потребители** доходят до **продюсеров/твоих сервисов** по DNS-имени, за которое ты выставил **PPU** (price per unit / consumption).

### Как это устроено

1. В **Rocket Store** на телефоне ставишь приложение **The Registry**.  
2. Регистрируешь домен (часто берут `.none`, чтобы не путать с реальным DNS).  
3. Назначаешь **Associate Usage** — тип услуги (STREAM-VOICE, UPDATE-SOFTWARE, Store-Text, Facilitate-P2P-Transaction, …).  
4. Ставишь **PPU** в разумных пределах (в гайдах типично ~1.1 для VOIP/Git/Padu, ~1.2 для live video, ~0.2 для Decentro P2P).  
5. В netshell: `dns map домен as @сервис on @твой_dns`.  
6. Клиенты этажа должны резолвить через этот DNS и иметь маршрут до `@сервиса`.

Без п.5 (у тебя авто-DNS **выкл**) домен в Registry «пустой»: жильцы не найдут сервер.

### Типичный стартовый набор ISP

| Usage / роль | Программа | Трафик | PPU (ориентир) |
|--------------|-----------|--------|----------------|
| STREAM-VOICE | voip-server + **физический телефон** на этаже | udp/5060 | ~1.1 |
| UPDATE-SOFTWARE | gitcoffee (+ часто padu) | tcp/443 | ~1.1 |
| Store-Text / image / video | padu / poems-db | tcp/80 | ~1.1 (text выгоднее) |
| Stream-Live-Video | rtsp-diva-r | udp/554 | ~1.2 |
| Facilitate-P2P-Transaction | decentro-node | tcp/8333 | ~0.2 |

Дополнительно пассив: просто соединять consumers↔producers на этажах (ISP-трафик по SLA) — для этого достаточно маршрутов и DNS map имени продюсера на его `@`.

### Пошагово «первый чек»

1. `pivoip @c1/svc/voip` (и запуск, автостарта нет).  
2. Registry: `voip.none` → STREAM-VOICE → PPU 1.1.  
3. `dmap voip.none @c1/svc/voip @c1/b1/dns`.  
4. Подключи **публичный телефон** на этаже к switch, дай ему `@` / DHCP.  
5. На пути телефона → VOIP: traffic-route `udp/5060` к порту сервисов (см. главу про телефоны).  
6. То же для `git.none` → GitCoffee / tcp/443.

Surveyor показывает, кто на этаже продюсер/консьюмер и когда они «онлайн» — map и пинги делай, когда цель не спит.

---

---

## Телефоны и камеры (udp/5060 и udp/554)

### Почему отдельно от обычного DNS-трафика

- **Accept-VOIP-Phone-Connection** / встроенный телефонный модуль этажа шлёт **udp/5060** и **не ищет hostname** так же, как браузер — нужен **traffic route** (или удачный default до VOIP-сервера).  
- Камеры этажа / Accept-CCTV — **udp/554**, та же логика.  
- Отдельный сервис Stream-Voice / Stream-Live-Video уже может идти по DNS на voip-server / rtsp — но «железная» трубка/камера на этаже всё равно требует маршрута трафика.

Правило из гайдов: **нет публичного телефона на сети — нет VOIP-выручки** с Accept-VOIP. Ориентир: 1 телефон ≈ 2–3 endpoint’а спроса.

### Как подключить

1. Найди телефон/камеру на этаже (Surveyor / осмотр).  
2. Патч в switch этажа (или сразу в роутер).  
3. Имя: `ncall @c1/b1/f2/phone @c1/b1/dns HW` (или DHCP).  
4. На edge / промежуточных роутерах направь трафик к серверу:

```text
rcat udp/5060 PORT_TOWARD_SVC @c1/b1
rcat udp/554 PORT_TOWARD_SVC @c1/b1
```

На серверном роутере — на порт VOIP/CCTV:

```text
rcat udp/5060 PORT_VOIP @c1/svc
rcat udp/554 PORT_CAM @c1/svc
```

Частый паттерн Hitchhiker: phone → traffic udp/5060 вверх к A1, дальше default в ядро, на `@c1/d1` уже `rcat udp/5060` на порт voip-сервера.

5. FW на пути: `fcavoip` / `fcacam` или полный `fcall`.  
6. Проверка: трафик на `watch` сервера, деньги/usage в UI, `pcap` на uplink.

Камеры можно VLAN’ом «прибить» ближе к CCTV-серверу (см. VLAN), чтобы не гонять udp/554 через весь consumer-broadcast switch.

---

---

## VLAN (когда один uplink и много логических сетей)

### Зачем

При **конечной ПС** и одном физическом uplink’е VLAN режет broadcast-домены: телефоны/камеры/клиенты/аплинк не обязаны быть в одном «хабовом супе». Нужен **managed switch с VLAN** (Blade12/15/88/… — смотри фичи в магазине) и при router-on-a-stick — VLAN-роутер (в обсуждениях — Kilo и т.п.).

### Термины в TNI (по дев-ответам + man vlan)

| Понятие | Как в игре |
|---------|------------|
| Access-порт | Один tag на порту → устройства этой VLAN |
| Trunk | **Несколько** tag на одном порту → несколько VLAN вверх |
| Subinterface | На роутере `port1.1`, `port1.24` с tag — маршрут `via port1.1` |

### Базовый рецепт router-on-a-stick

1. На switch: access — `vlan tag port2 with #10 on @sw` (клиенты VLAN 10).  
2. Trunk к роутеру: `vlan tag port0 with #10 #20 on @sw` (несколько VLAN).  
3. На VLAN-роутере: subinterface с тем же tag, линк в trunk.  
4. `route add @prefix10 via port0.1 on @router` (и аналоги для других VLAN).  
5. `vlan show on @sw` — проверить.

Примеры из `man`/каталога:

```text
vlan show on @mysw1
vlan tag port1 with #vlan1 on @mysw
vlan tag port3 with #vlan1 #vlan2 on @mysw
vlan tag port3.10 with #vsub1
vlan untag port2 with #vlan2 on @mysw
vlan clear on @mysw
```

Алиасы-помощники (`vshow`, `vtag1`, `vuntag`, `vclear`) — секция VLAN в [`alias-pack.txt`](./alias-pack.txt).

Для новичка: **сначала блоки без VLAN**; VLAN — когда упрёшься в ПС/изоляцию phone-cam от клиентов на одном Blade.

---

---

## RIP — автораздача маршрутов

### Unlock

Secretariat proposal вроде **router configuration** → появляется `rip` в netshell.

### Как работает

1. Ты по-прежнему задаёшь **конечные** route на «листьях» (сервер `@c1/svc/voip` via portX на ближайшем роутере).  
2. Включаешь на роутерах **advertise** (объявляю свои route соседям) и **listen** (слушаю чужие).  
3. Соседи подхватывают префиксы/хосты → не нужно вручную прописывать всю середину цепочки `@b1`→`@c1`→`@svc` на каждом хопе.

Пока RIP нет — живи **prefix routing** вручную (глава выше).

### Команды

```text
rip show on @router
rip advertise on @router
rip stop advertise on @router
rip listen on @router
rip ignore on @router
```

Опционально advertise только в сторону: `rip advertise via portN on @router` (см. man).

### Алиасы

`ripsh`, `ripa`, `ripao`, `ripl`, `ripi`, `ripup`, `ripdown`, `rip4` — секция RIP в [`alias-pack.txt`](./alias-pack.txt).

Практика: на `@c1`, `@c1/b1`, `@c1/svc` выполни `ripup`, endpoint’ы оставь `rca`/`rcat`. После Lab `init_f2`: `rip4 @c1/b1/f2 @c1/b1/f1 @c1/b1 @c1`. Проверка: `rip show` + `trace`.

---

---

## pcap и диагностика перегруза

### Когда нужен

- Link/Tower Link в UI «красный» на 100% traversals.  
- Blade «задыхается», клиенты тормозятся.  
- Непонятно, какой класс трафика забивает uplink.  
- Ловишь malicious / странный tcp.

Нужен **ethernet tap** (или firewall как tap) на интересующем сегменте — `pcap` идёт **on &lt;tap_address&gt;**.

### Команды

```text
pcap on @tap
pcap =udp/53 on @tap
pcap exclude =tcp/23 =udp/67 =udp/53 on @tap
pcap dump =udp/53 on @tap
```

### Алиасы

`pcapa`, `pcape`, `pcapd`, `pcapf` — секция pcap в [`alias-pack.txt`](./alias-pack.txt).

### Методика

1. Поставь tap на uplink этажа или перед edge.  
2. `pcape @tap` — отсей свой netshell/DNS/DHCP шум.  
3. Смотри, что осталось: VOIP? scrape? broadcast?  
4. Лечение: апгрейд Tower Link (cat1→cat5…), разнести VLAN, вынести phone на отдельный путь, FW/blackhole на мусор, не вешать 6 этажей на один Blade.  
5. `dstat @device` — счётчики портов; `dstat clear @device` — сброс для нового замера.  
6. `watch @server` — видишь ли сервис реальный трафик usage.

### Tower Link перегружен

В приложении View Links → Manage: если average traversals ~100%, нужен более быстрый cat. Смена скорости: deactivate → upgrade → reactivate (**краткий outage**). Стоимость/дневка растут с этажом и классом линка (cat1 ≈ 15 trav/tick, cat5 ≈ 30, … — сверяй UI).

---

---

## Blackhole-маршруты (вместо или вместе с FW)

### Идея

`route add traffic tcp/8034 via portX on @router`, где **portX никуда не воткнут** — трафик «уходит в никуда». Hitchhiker так глушит text scrapers без отдельного deny на FW.

Плюсы: быстро на любом роутере.  
Минусы: легко забыть «дырявый» порт; не заменяет политику FW на границе блока; ошибочный blackhole боевого трафика = тихий outage.

### Практика

`rcbh` — секция pcap/blackhole в [`alias-pack.txt`](./alias-pack.txt). Пример:

Примеры:

```text
rcbh tcp/8034 5 @c1/b1
rcbh tcp/510 5 @c1/b1
```

(Порт 5 свободен.) Для диапазона Morris надёжнее `fcmal` / `fcsafe` на FW — blackhole по одному порту # не закроет 510–519 одним махом, если не делать пачку `rcbh`.

Комбо: на edge `fcmal` или blackhole scraper; на svc — полный `fcall`.

---

---

## Сводная таблица портов / трафика

| Порт / класс | Зачем | Где учесть |
|--------------|-------|------------|
| **tcp/23** | Управление netshell/FW | FW allow **первым**; exclude в pcap |
| **udp/53** | DNS | DHCP option dns, FW, route к DNS |
| **udp/67** | DHCP | `rcat udp/67`, FW, broadcast |
| **tcp/80** | Padu / store HTTP | Money + FW |
| **tcp/443** | GitCoffee / updates | Money + FW |
| **udp/5060** | VOIP / телефоны этажа | `rcat`, телефон, FW |
| **udp/554** | CCTV / RTSP | `rcat`, камеры, FW |
| **udp/1194** | Instruct / VPN-like usage | FW whitelist |
| **tcp/8333** | Decentro | FW; иногда двусторонний путь через switch |
| **tcp/8034** | Text scraper | `fcmal` / blackhole |
| **tcp/510–519** | Morris worm | `fcmal` / FW deny |
| **icmp** | ping/diag | FW allow в whitelist |
| **tcp/3306 / 5432** | БД (padu и др.) | Иногда `rcat` на серверном роутере |

---

---

## Tower Link и Socketeer (физика ЦОД/этажей)

### Tower Link (ещё раз пошагово)

1. Floor from + serial порта (4 буквы на розетке).  
2. Floor to + serial.  
3. Класс скорости (cat) по деньгам и traversals.  
4. Request link → link lights.  
5. Перегруз → View Links → Manage → deactivate / upgrade / reactivate.

### Socketeer (~500$ в Rocket Store)

Ставит **дополнительные розетки** (copper/fiber) в мире — чтобы не тянуть кабели через весь ЦОД. Типы: copper / fiber; remove — за отдельную плату. Полезно, когда портов на стіне не хватает под блоки и NAS.

---

---

## Источники

- [tni-unofficial-docs](https://avril112113.github.io/tni-unofficial-docs/) ([GitHub](https://github.com/Avril112113/tni-unofficial-docs)) — сген. данные устройств/доки (часто beta)
- Steam: [Hitchhiker's Guide](https://steamcommunity.com/sharedfiles/filedetails/?id=3651464033) (DHCP, DR/sftp/Morris, RIP, VOIP/phone, VLAN aliases, Tower Link)
- Steam: [Firewalls - Basics and Traffic Types](https://steamcommunity.com/sharedfiles/filedetails/?id=3548511586) (Datawiper, nftables copy)
- Steam discussions: VLAN trunk / router-on-a-stick (dev)
- Pocosia roadmap: warranty, sftp, RIP, power events
- HackMD Aliases / [device-tables](https://hackmd.io/@tower-network/device-tables); tutorial Riser Setup Across Floors

Сверяй `man route`, `man net`, `man dns`, `man dhcp`, `man firewall`, `man sftp`, `man rip`, `man vlan`, `man pcap` на своей сборке.
