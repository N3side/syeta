!!! 4 порт должен быть на iptv на gpon/ethernet/adsl. Должен быть активен линк

# ADSL
Если артефакты/зависания, то сменить в профайле задержку на 255 255. pvc нет смысла пересоздавать
Если все равно не работает iptv после шагов ниже, то пересоздать pvc. Также если инет не работает, то для интернета тоже pvc пересоздать


## Алгоритм «телевизор перестал работать» — подробно на zyxel1000

## Шаг 1. MAC-адреса на порту

```
statistics mac 8
```

Смотрим, есть ли MAC приставки и в каком VLAN.

Как читать:
- MAC приставки виден в **IPTV VLAN** (у нас PVID 1201) → приставка достучалась, L2 в порядке. Идём к Шагу 2.
- MAC **нет вообще** → приставка не подключена / кабель приставка–модем / модем не бриджует / приставка выключена. Дальше DSLAM не виноват — к абоненту.
- На всех роутерах по умолчанию 4 интерфейс. RX-11000 если до заводских сбросить то по умолчанию будет 3 интерфейс для iptv.

MAC приставки есть - Дальше.

## Шаг 2. VLAN / PVID

```
switch vlan portshow 8
```

Смотрим pvid порта — должен быть **IPTV VLAN** (super channel).

**Норм:**
- PVID = IPTV VLAN (у нас 1201) и совпадает с рабочими IPTV-портами.
- PVID = интернет/управление (2846/1308) → трафик приставки уходит не туда. **Сменить:**
  ```
  switch vlan pvid 8 <IPTV_VLAN>
  ```
- Сверить с рабочим портом того же DSLAM:
  ```
  switch vlan portshow <рабочий_IPTV_порт>
  ```


## Шаг 3. Подписку чекаем

```
statistics igmpsnoop port group 8
```

Смотрим, есть ли подписка на multicast-группы.

**Как читать:**
- **Список групп есть** (например `vid 1200, 239.255.11.74, source 10.35.66.101`) → приставка прислала join, snooping видит, порт зафиксирован. Значит **со стороны DSLAM подписка работает** — идём к трафику (Шаг 5) и к абоненту.
- **Пусто** → приставка не шлёт join. Причины:
  1. **mvlan не зафиксирован** на порту — проверить:
     ```
     switch igmpsnoop mvlan show *
     ```
     Порт должен быть **F (Fixed)** на multicast-VLAN. Если `U`/`T`/`-` — зафиксировать:
     ```
     switch igmpsnoop mvlan set <mvlan_vid> 8:FU MULTICAST
     ```
  2. Приставка не в IPTV-VLAN (Шаг 2) / фильтр режет IGMP (Шаг 3).
  3. Проблема на стороне абонента: модем не бриджует IPTV-PVC, IGMP proxy в модеме выключен, приставка/кабель.

**Сверить mvlan и PVID:** у нас PVID 1201, mvlan 1200 — это нормально (super-channel VLAN порта и multicast-VLAN могут различаться), главное порт **F на mvlan**.

---

## Шаг 4 (дополнение). Трафик на IPTV-PVC

```
statistics port 8 1 32
```

Снять **дважды** с паузой 1–2 мин, сравнить.

**Как читать:**
- `tx packets`/`tx cells` **растут**, `tx rate` высокий, `errors 0` → multicast к абоненту идёт непрерывно. Со стороны DSLAM всё, идём к абоненту (модем/кабель/приставка).
- `tx rate ≈ 0`, пакеты не растут → multicast не доходит до порта: mvlan/агрегация/upstream (проверить `switch igmpsnoop mvlan show *`, `statistics vlan`, uplink).
- `rx packets` растут, `tx` нет → join идёт, ответного потока нет — upstream multicast.
- `errors` растут → проблема линии/PVC (тогда к профилю/линии).

---

## Сводная логика «не работает»

```
Нет картинки
   │
   ├─ statistics mac 8 → MAC приставки в IPTV VLAN есть?
   │        ├─ НЕТ → кабель/модем/приставка (L2 не доходит), к абоненту
   │        └─ ДА → Шаг 2
   │
   ├─ switch vlan portshow 8 → PVID = IPTV VLAN?
   │        ├─ НЕТ → сменить pvid на IPTV VLAN
   │        └─ ДА → Шаг 3
   │
   ├─ statistics igmpsnoop port group 8 → группы есть?
   │        ├─ НЕТ → проверить mvlan (порт F?) → switch igmpsnoop mvlan show *
   │        │         └─ не F → switch igmpsnoop mvlan set <vid> 8:FU MULTICAST
   │        │         └─ F, а групп нет → приставка/модем (IGMP proxy, bridge) — к абоненту
   │        └─ ДА → Шаг 4
   │
   └─ statistics port 8 1 32 (дважды) → tx растёт, errors 0?
            ├─ ДА → DSLAM ок → модем/кабель/приставка — к абоненту
            └─ НЕТ → mvlan/агрегация/upstream multicast
```

---

# CDATA

**Всегда надо заходить в enable**

Аптайм и инфо по шасси
```
show version
```

перезапустить порт:
```
config
interface gpon 0/0
ont reboot 4 13
```

или
```
ont port attribute 2 24 eth 1 operational-state disable
ont port attribute 2 24 eth 1 operational-state enable
```

Проверка сигнала на порту
```
config
interface gpon 0/0
show ont optical-info 2 1
```

Проверить мак адрес на порту
```
config
show mac-address port pon 0/0/4 ont 13
```

Маки по vlan
```
show mac-address vlan 900
```

Все вланы:
```
show vlan all
```

Состояние линка от ONT к Роутеру/ПК
```
config
interface gpon 0/0
show ont port state 2 24 eth all
```

Проверка логов:
```
show log all
show logging buffer
```

Арпы:
```
show arp all
```


Полная дока:
```
https://fibotelecom.ru/upload/iblock/acf/zwsh9i184rbpiv2e15xnwzjb1r1o6dh3.pdf
```


# L3

1. чек через vlan какой порт
2. если маки с нашей стороны приходят то ок. Если нет то проблема на настройках коммута
3. 
