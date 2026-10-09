# Template Pantum Printer — краткий README

Объединённый шаблон для принтеров Pantum (7100, 6500 и др.). Zabbix 7.4.

## Что внутри

| Айтем | Источник | Триггер |
|---|---|---|
| `pantum.uptime` (5 мин) | SNMP `1.3.6.1.2.1.1.3.0` | nodata 30 мин → HIGH «Принтер не отвечает по SNMP» |
| `pantum.drum` (1 ч) | SNMP `1.3.6.1.4.1.40093.10.1.5.3.1.2` | < 12% → WARNING; скачок >70% → INFO «Драм заменен» |
| `pantum.toner[{#SNMPINDEX}]` (1 ч, discovery) | SNMP `43.11.1.1.9.{#SNMPINDEX}` | < 12% → WARNING (recovery > 20%); скачок >70% → INFO «Новый картридж» |
| `pantum.toner.days` (1 ч) | CALCULATED | — дней с замены тонера |
| `pantum.drum.days` (1 ч) | CALCULATED | — дней с замены драма |
| `pantum.pages` (1 ч) | SNMP `1.3.6.1.4.1.40093.1.1.3.12` | — суммарно отпечатано страниц (trends 365 дн) |
| `pantum.serial` / `pantum.model` / `pantum.firmware` (1 сут) | SNMP `40093.1.1.1.5` / `.10` / `.1` | — инвентарка |

**Счётчик страниц:** Pantum не отдаёт стандартный `prtMarkerLifeCount`
(43.10.2.1.4 → No Such Instance). В дампе M7100DW найден вендорный счётчик
`40093.1.1.3.12` = 64397 (дублируется в `40093.10.3.1.1`). **Проверь на
дисплее принтера** (меню отчётов → «Всего отпечатков»): если там другое
число — скажи, подставим `40093.1.1.3.3` (63619) или `40093.1.1.3.11` (8297,
вероятно копии/сканы).

**Про 6500 (проверено дампом 6500.xml, Pantum-3EEA94, enterprise 26696):**
вендорское дерево `1.3.6.1.4.1.26696` пустое, стандартного счётчика `43.10`
нет, серийника `43.5.1.1.17` нет. Что работает: `pantum.uptime` (sysUpTime)
и тонер через discovery (картридж pc211, уровень `43.11.1.1.9.1.1`). Страниц
по SNMP у 6500 **нет** — только в веб-панели принтера. На хосте 6500 отключи
айтемы: `pantum.pages`, `pantum.drum`, `pantum.serial`, `pantum.model`,
`pantum.firmware` (уйдут в unsupported).

## Миграция со старых шаблонов

1. Импортируй `template-pantum.yaml`.
2. На хосте принтера: отвяжи `pantum7100`/`pantum6500`, привяжи «Template Pantum Printer».
3. У хоста должен быть **SNMP-интерфейс** (IP принтера, 161, SNMPv2, community).
4. На модели **без драма** (6500) отключи айтем `pantum.drum` на хосте.
5. Настрой action для «дней с замены» (ниже) — без него даты не появятся.

## «Дней с замены»: единоразовая настройка action

Zabbix не умеет «время с последнего события» в чистом виде, поэтому дату замены
ставит **trigger action** через `zabbix_sender`. Делается один раз на весь сервер:

1. Проверь, что на Zabbix-сервере есть `zabbix_sender` (`which zabbix_sender`;
   нет — `apt/yum install zabbix-sender`).
2. Alerting → Actions → **Trigger actions** → Create action:
   - **Name:** Pantum: фиксация замены картриджа/драма
   - **Conditions:** Trigger name | contains | `картридж` (зацепит оба триггера:
     «Новый картридж» и «Драм заменен») — либо два отдельных action точнее.
   - **Operations:** Operation type — **Remote command**; Target — **Current host**... 
     но команда нам нужна на сервере: Target = Zabbix server (или выбери «Custom script»
     с выполнением на Zabbix server).
     Команда для тонера:
     ```
     zabbix_sender -z 127.0.0.1 -s "{HOST.NAME}" -k pantum.toner.replaced -o $(date +%s)
     ```
     для драма (второй action, условие по триггеру «Драм заменен»):
     ```
     zabbix_sender -z 127.0.0.1 -s "{HOST.NAME}" -k pantum.drum.replaced -o $(date +%s)
     ```
3. Готово: при следующей замене trapper-айтем получит дату, `*.days` посчитаются.
   До первой замены `*.days` будут unsupported — это норма.

Если action настраивать не хочется — даты замен всегда видны в Monitoring → Problems
по INFO-событиям «Новый картридж» / «Драм заменен» (время события = дата замены).

## Пороги (макросы шаблона)

| Макрос | По умолчанию | Что меняет |
|---|---|---|
| `{$PANTUM.TONER.MIN.WARN}` | `12` | WARNING «тонер заканчивается» |
| `{$PANTUM.TONER.RECOVER}` | `20` | Закрытие проблемы (гистерезис) |
| `{$PANTUM.DRUM.MIN.WARN}` | `12` | WARNING «драм заканчивается» |
| `{$PANTUM.SNMP.NODATA}` | `30m` | Молчание SNMP = принтер пропал |

## Оговорки

- Иконка хоста честно загорается (SNMP-интерфейс меняет availability).
- Цветной Pantum: discovery создаст айтем на каждый картридж автоматически.
- Старые `7100.yaml`/`6500.yaml` не удалял — лежат до успешной миграции.

## Файлы

- `zabbix\template-pantum\template-pantum.yaml` — экспорт шаблона.
