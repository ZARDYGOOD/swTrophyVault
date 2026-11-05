# swTrophyVault — ТЗ v0.2

**Статус:** решения зафиксированы · можно кодить после OK на срез MVP  
**Плагин:** `swTrophyVault` (Paper)  
**Язык UI / реплик:** **EN only**  
**Связь:** SpeakWest hub + юнит-плагины (первый клиент — Star Academy → **Star Trophy**)  
**Не** внутри JAR StarAcademy — отдельный плагин.

Папка на ПК (когда стартуем код):  
`D:\PROJECTS MINECRAFTSERVERS\SpeakWest\swTrophyVault\`

---

## 0. Зафиксированные решения

| Тема | Решение |
| --- | --- |
| Имя | **swTrophyVault** |
| Реплика при открытии NPC | **Да**, короткая EN (пример: `Show me your stars.`) |
| Язык | **EN only** |
| Связь юнитов | Общий контракт: юнит-плагины **регистрируют** трофеи и **сообщают earn** в Vault; Vault — единый каталог «какие трофеи есть на сервере» + коллекции игроков |
| API | **Да, лёгкий** — публичный Java API + Bukkit Service внутри плагина; дока для других разрабов. Отдельный GitHub `*-api` jar — когда появится второй внешний разраб; пакет сразу `…api` чтобы не ломать потом |
| Вынос трофея | В **hub** можно взять трофей из Vault в инвентарь; **уйти из hub нельзя**, пока не вернёшь |
| Тексты про сдачу | Если трофей **уже был** в Vault и взят наружу → *return*. Если предмет с юнита **ещё ни разу не сдавали** → *deposit* (не говорить «return») |
| Карточка трофея | **Да** в MVP (отдельный экран/панель по клику). Нарисованный UI — цель V2 под тот же layout |
| GUI MVP | Chest / double-chest; layout слотов сразу дружит с будущим painted фоном |
| Чужие коллекции | MVP: только своё + `/swtrophy peek <player>` учителю. Публичная витрина — позже |

---

## 1. Проблема

После юнита трофей (сейчас Star Trophy) падает в инвентарь → `/hub`. Его можно потерять; нет общей полки достижений и нет правила «похвастался в хабе — верни».

Нужен NPC в хабе: ПКМ → меню → коллекция трофеев всех SpeakWest-юнитов, с безопасным выносом только внутри hub.

---

## 2. Цель

1. Единое **хранилище трофеев** игрока.  
2. Единый **каталог трофеев сервера** (что зарегистрировали юнит-плагины + `trophies.yml`).  
3. **Checkout** в hub + блок выхода, пока трофей на руках.  
4. Контракт для **любого** следующего `sw*` / юнит-плагина без копипасты.

---

## 3. NPC и вход

| | |
| --- | --- |
| Тело | Citizens вручную; плагин **не** спавнит |
| Bind | `/swtrophy set keeper` рядом с NPC → link Citizens id |
| Open | ПКМ → bubble/chat line EN → GUI |
| Who | `swtrophy.use` (default true для игроков) |
| Worlds | `keeper-worlds` в конфиге (обычно `hub`) |

Пример open line (конфиг): `Show me your stars.`

---

## 4. Модель данных

### 4.1 Trophy definition (каталог сервера)

Регистрируется через API и/или `trophies.yml`:

| Поле | Пример |
| --- | --- |
| `id` | `star_academy_u1` |
| `displayName` | `Star Trophy` |
| `unitLabel` | `Star Academy · Unit 1` |
| `sourcePlugin` | `StarAcademy` |
| `icon` | material + optional CMD |
| `lore` | EN lines for GUI / card |
| `cardBlurb` | `You finished Star Academy Unit 1.` |

Vault **мержит** yaml + runtime `registerTrophy(...)`. Дубликат `id` — побеждает первая регистрация / yaml override (правило описать в доке API).

### 4.2 Player collection

На каждый `id`:

| Поле | Смысл |
| --- | --- |
| `owned` | заработан (earn / первый deposit) |
| `earnedAt` | timestamp первого получения |
| `checkedOut` | сейчас вынесен в инвентарь в hub |
| `everDeposited` | хотя бы раз лежал в Vault (для copy: return vs deposit) |

**Правила**

- Один `id` → один слот; повторный earn не плодит копии (дата первого earn сохраняется).  
- Checkout выдаёт **один** tagged item (PDC: trophy id + owner uuid).  
- Этот item нельзя задюпать: дроп вне hub запрещён или item уничтожается / варн; в hub — только у владельца.

---

## 5. Checkout / return / deposit (критично)

### 5.1 Состояния предмета у игрока

```
A. Нет трофея, не owned     → слот locked / empty
B. owned, в Vault           → иконка в GUI, можно Take
C. owned, checked out       → item в инвентаре; GUI показывает "Out"; нужен Return
D. Не owned, item с юнита   → первый Deposit → owned + everDeposited
```

### 5.2 Тексты (EN) — не путать

| Ситуация | Правильный смысл | Пример |
| --- | --- | --- |
| Первый раз сдаёт предмет с урока | **Deposit** | `Deposit your Star Trophy in the Vault.` |
| Уже owned, взял из Vault, надо вернуть | **Return** | `Return your Star Trophy to the Vault.` |
| Пытается уйти из hub с checked-out | **Return** | `You can't leave the hub until you return your trophy.` |
| Пытается уйти из hub с ещё не сданным лут-трофеем юнита | **Deposit** | `Deposit your trophy before leaving the hub.` |

### 5.3 Блок выхода из hub

Пока у игрока в инвентаре есть vault-tagged trophy **или** (опционально конфиг) unstored unit trophy, **запретить** смену мира / команды телепорта с hub:

- listen `PlayerTeleportEvent`, `PlayerPortalEvent`, смена world;
- известные exit-команды по конфигу (`spawn`, варпы и т.д.) — cancel + сообщение;
- `/hub` из hub = no-op; из юнита в hub — ок (как сейчас StarAcademy).

Разрешённые миры = `keeper-worlds`. Выход = любой мир вне списка.

Админ bypass: `swtrophy.bypass`.

### 5.4 Take / Return в GUI

- **Take** — только в keeper-world; ставит `checkedOut=true`, даёт item.  
- **Return** — клик Deposit/Return слот или ПКМ item на NPC/GUI; снимает item, `checkedOut=false`.  
- Нельзя Take второй раз, пока checked out.

---

## 6. GUI

### 6.1 MVP

Double-chest inventory:

- Trophy slots (owned / empty / out)  
- Click trophy → **Card** view (отдельный маленький GUI или тот же инвентарь с lore-панелью): name, unit, date, blurb  
- Buttons: Take / Return·Deposit / Close  
- Header: `★ Trophy Vault` + `1 / N`

### 6.2 Painted UI (V2)

Тот же slot map + resource pack / Oraxen фон. Карточка на нарисованной панели — идеальный вид; логика не меняется.

---

## 7. API для юнит-плагинов (решение: делать)

Не «огромный SDK». Минимальный контракт, чтобы любой разраб юнита подключался.

### 7.1 Что кладём в `swTrophyVault`

Пакет `…trophyvault.api` (стабильный):

```text
SwTrophyVaultAPI
  registerTrophy(TrophySpec spec)
  earn(Player player, String trophyId) -> EarnResult
  has(Player player, String trophyId) -> boolean
  isCheckedOut(Player player, String trophyId) -> boolean

Events
  TrophyRegisterEvent
  TrophyEarnEvent   (после успешного первого earn)
  TrophyCheckoutEvent
  TrophyReturnEvent
```

Регистрация через `Bukkit.getServicesManager()` + soft-depend `swTrophyVault` в `plugin.yml` юнита.

### 7.2 Документация

Короткий `docs/api-for-unit-devs.md` (EN):

1. Add soft-depend.  
2. On enable: `registerTrophy(...)`.  
3. On unit complete: `earn(player, id)` (+ optionally give display item).  
4. Vault owns persistence / hub rules.

### 7.3 GitHub

- **Сейчас:** API classes в том же JAR; дока в репо/Store.  
- **Потом:** вынести `swTrophyVault-api` thin jar + Maven/GitHub package, когда внешние разрабы реально появятся.  
Контракт с первого дня не ломать (semver API).

### 7.4 Star Academy

Мелкий follow-up патч: при выдаче Star Trophy → `earn(..., "star_academy_u1")` + PDC на item. Пока патча нет — Vault recognizer по имени/lore/PDC на Deposit.

---

## 8. Команды и права

| Команда | Кто | |
| --- | --- | --- |
| `/swtrophy` | use | открыть GUI (если keeper недоступен — всё равно своя коллекция) |
| `/swtrophy set keeper` | admin | bind Citizens |
| `/swtrophy reload` | admin | config + trophies |
| `/swtrophy peek <player>` | teacher/admin | чужая коллекция read-only |
| `/swtrophy grant <player> <id>` | admin | debug earn |
| `/swtrophy return` | use | принудительно вернуть checkout в Vault (если GUI сбоит) |

Права: `swtrophy.use`, `swtrophy.teacher`, `swtrophy.admin`, `swtrophy.bypass`.

Alias (опционально): `/vault` → `/swtrophy`.

---

## 9. Конфиг (эскиз)

```yaml
# config.yml
keeper-worlds: [hub]
open-line: "Show me your stars."
gui-title: "★ Trophy Vault"
block-leave-with-trophy: true
block-leave-with-undeposited-unit-item: true
exit-command-blocklist: []   # optional extras
```

```yaml
# trophies.yml
trophies:
  star_academy_u1:
    name: "Star Trophy"
    unit: "Star Academy · Unit 1"
    material: NETHER_STAR
    card-blurb: "You finished Star Academy Unit 1."
    lore:
      - "Star Academy"
      - "Unit 1"
```

---

## 10. Срез MVP (кодим это)

Must:

1. Citizens bind + open line + chest GUI  
2. Каталог из `trophies.yml` (+ пустой runtime register)  
3. Earn / Deposit (тексты deposit vs return)  
4. Take / Return + **block leave hub**  
5. Card screen по клику  
6. Persist YAML/SQLite  
7. Права + `/swtrophy peek`  
8. Короткий EN API doc stub  
9. StarAcademy: либо soft hook, либо deposit-only до патча  

V2: painted UI, showcase shelf, GitHub api-jar, class board.

Out of scope v1: магазин, трейд, автоспавн NPC, RU UI, рестарт Apex без запроса.

---

## 11. Критерии готовности MVP

- [ ] ПКМ keeper → EN line → GUI  
- [ ] Star Trophy можно deposit / earn и увидеть в слоте  
- [ ] Take в hub → нельзя сменить мир, пока Return  
- [ ] Сообщения: deposit для новых, return для checkout  
- [ ] Card открывается и показывает unit + date + blurb  
- [ ] Релог сохраняет owned / checkedOut  
- [ ] `registerTrophy` + `earn` вызываются из тестового мини-хука / javadoc  
- [ ] Teacher note ½ страницы EN+RU ok для учителя, UI EN  

---

## 12. Именование линейки SpeakWest

Префикс **`sw`**: `swTrophyVault`, дальше юниты/сервисы в том же стиле. Vault — общий хаб-сервис; юниты — клиенты API.

---

*v0.2 · следующий шаг: OK на срез §10 → скелет Gradle-плагина + PC-папка `swTrophyVault`.*
