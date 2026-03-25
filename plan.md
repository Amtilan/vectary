# План реализации интерактивного 3D-стенда через Vectary (v2)

> Обновлённый план с учётом CSV Data Import, TRIGGER(), Selections и других ускорителей

---

## 1. Проблема и подход

### 1.1. Исходная ситуация

- 3D-модель стенда из Fusion 360 — слишком большая для веба
- ТЗ требует 4 режима: Exploration, Practice, Theory, Guided Demo
- ~20–30 компонентов, каждому нужна карточка, подсветка, анимации, логика

### 1.2. Ключевое решение: data-driven архитектура

Вместо ручной настройки каждого компонента по отдельности — **весь контент живёт в одной Google-таблице**, а Vectary подтягивает данные через CSV Import + Variables + Expressions.

Это меняет всё:
- 1 интеракция вместо 30 (через TRIGGER() + Selections)
- 1 универсальная карточка вместо 30 отдельных Floating UI
- Обновление контента → правка таблицы, а не лазание по Vectary
- Добавление нового компонента → новая строка в таблице + объект в Selection

### 1.3. Рекомендации Vectary по размерам

| Параметр | Лимит | Наш ориентир |
|----------|-------|-------------|
| Полигоны | До 100K | 50K–80K (запас на UI элементы) |
| Объекты | До 100 | 60–80 (компоненты + провода + UI) |
| Размер файла | До 5 МБ | 3–4 МБ |
| Текстуры | До 100 | 30–50 |
| FPS | > 24 fps | > 30 fps |

---

## 2. Подготовка модели (Этап 1)

### 2.1. Упрощение в Fusion 360

- Удалить невидимые внутренности (платы, внутренние провода PLC)
- Упростить резьбу, мелкие скругления, фаски
- Объединить группы мелких клемм в один mesh
- Mesh → Reduce для снижения полигонов

### 2.2. Экспорт по группам

| Группа | Состав | Формат экспорта |
|--------|--------|----------------|
| Каркас стенда | DIN-рейки, корпус, дверцы, панель | OBJ или FBX |
| PLC | Контроллер + модули | OBJ или FBX |
| HMI | Дисплей, рамка | OBJ или FBX |
| VFD | Частотный преобразователь | OBJ или FBX |
| Клеммники | Все ряды (2–3 группы) | OBJ или FBX |
| Датчики | Индуктивные, ёмкостные, оптические | OBJ или FBX |
| Кнопки и лампы | Кнопки, переключатели, сигнальные лампы | OBJ или FBX |
| Автоматы | Автоматические выключатели, УЗО | OBJ или FBX |
| Провода (упрощённые) | Кабель-каналы, основные жгуты | OBJ или FBX |

### 2.3. Оптимизация текстур

- Мелкие компоненты: 512×512 или 1024×1024 px
- Крупные (корпус, HMI экран): 2048×2048 px
- Формат JPEG где возможно
- Повторное использование материалов: один материал на все однотипные объекты

### 2.4. Альтернатива: Vectary Processor

Десктопное приложение для конвертации CAD → mesh с контролем качества. Файлы не покидают компьютер. Загрузка в Vectary одной кнопкой. Доступ: sales@vectary.com

**Срок: 1–2 недели**
**Результат:** набор оптимизированных файлов, каждый < 5 МБ

---

## 3. Подготовка данных — Google-таблица (Этап 2)

> Это новый этап, которого не было в старом плане. Он критичен для всей дальнейшей работы.

### 3.1. Структура основной таблицы «components»

Создайте Google Sheets с листом **components**:

```
id          | name            | type     | type_icon | description                          | role                    | connection        | signal_type | power   | range    | max_current | protection | datasheet_url         | manual_url            | manufacturer_url      | theory_section     | camera_preset
PLC_01      | Siemens S7-1200 | ПЛК      | plc       | Программируемый логический контроллер | Основной контроллер     | Все модули I/O    | Digital/Analog | 24V DC | —        | 1.5A        | IP20       | https://...           | https://...           | https://...           | PLC архитектура    | cam_plc
HMI_01      | KTP700 Basic    | ЧМИ      | hmi       | Панель оператора с сенсорным экраном  | Визуализация процесса   | PLC через Ethernet | Ethernet   | 24V DC  | —        | 0.5A        | IP65 front | https://...           | https://...           | https://...           | —                  | cam_hmi
VFD_01      | Sinamics V20    | Частотник | vfd       | Преобразователь частоты               | Управление двигателем   | PLC через AI/AO   | Analog     | 380V AC | 0-50Hz   | 6.5A        | IP20       | https://...           | https://...           | https://...           | Аналоговые сигналы | cam_vfd
SENSOR_01   | SICK IME12      | Датчик   | sensor    | Индуктивный датчик приближения        | Детекция металла        | PLC вход %I0.0.2  | DI         | 24V DC  | 0-4mm    | 200mA       | IP67       | https://...           | https://...           | https://...           | Дискретные сигналы | cam_sensors
BTN_01      | Schneider XB5   | Кнопка   | button    | Кнопка управления с фиксацией        | Пуск/стоп оборудования  | PLC вход %I0.0.0  | DI         | 24V DC  | —        | 10mA        | IP65       | https://...           | https://...           | https://...           | Дискретные сигналы | cam_buttons
LAMP_01     | Schneider XB5   | Лампа    | lamp      | Сигнальная лампа зелёная              | Индикация работы        | PLC выход %Q0.0.8 | DO         | 24V DC  | —        | 20mA        | IP65       | https://...           | https://...           | https://...           | Дискретные сигналы | cam_buttons
TERM_01     | Phoenix Contact | Клемма   | terminal  | Клемма проходная 2.5мм²              | Коммутация сигналов     | Между PLC и полем | —          | —       | —        | 24A         | —          | https://...           | https://...           | https://...           | Общие провода      | cam_terminals
MCB_01      | Schneider iC60  | Автомат  | mcb       | Автоматический выключатель 6A         | Защита цепей питания    | Вводное питание   | —          | 230V AC | —        | 6A          | IP20       | https://...           | https://...           | https://...           | Питание            | cam_mcb
```

### 3.2. Лист «theory» для Theory Mode

```
section_id        | title               | short_text                                      | image_url      | sources                    | checklist                                          | related_components
digital_signals   | Дискретные сигналы   | Сигнал имеет два состояния: 0 и 1...            | schema_di.png  | https://..., https://...   | Проверить уровень напряжения; Проверить полярность | SENSOR_01,BTN_01,LAMP_01
analog_signals    | Аналоговые сигналы   | Непрерывный сигнал в диапазоне 0-10V или 4-20mA | schema_ai.png  | https://...                | Проверить диапазон; Калибровать датчик             | VFD_01
power             | Питание              | Источники 24V DC и 230/380V AC...               | schema_pwr.png | https://...                | Проверить заземление; Убедиться в автомате         | MCB_01
common_wires      | Общие провода        | Общий провод (COM, 0V) замыкает цепь...         | schema_com.png | https://...                | Маркировать провода; Проверить сечение             | TERM_01
shielding         | Экранирование        | Экран кабеля защищает от помех...               | schema_shd.png | https://...                | Заземлять экран с одной стороны                    | SENSOR_01
plc_architecture  | PLC архитектура      | PLC состоит из CPU, модулей DI/DO/AI/AO...      | schema_plc.png | https://...                | Проверить адресацию; Проверить конфигурацию        | PLC_01
error_protection  | Защита от ошибок     | КЗ, перегрузка, обрыв провода...                | schema_err.png | https://...                | Проверить предохранители; Тестировать цепь         | MCB_01
```

### 3.3. Лист «practice_steps» для Practice Mode

```
practice_id | step_num | instruction                                       | target_terminal | correct_terminal | highlight_color | wire_object
P001        | 1        | Подключите провод от кнопки 1 к входу %I0.0.0     | BTN_01          | TERM_DI_00       | yellow          | wire_btn1_di0
P001        | 2        | Подключите общий провод кнопки 1 к 0V             | BTN_01          | TERM_0V_01       | yellow          | wire_btn1_0v
P001        | 3        | Подключите провод от выхода %Q0.0.8 к лампе 1     | LAMP_01         | TERM_DO_08       | yellow          | wire_do8_lamp1
P001        | 4        | Подключите общий провод лампы 1 к 0V              | LAMP_01         | TERM_0V_02       | yellow          | wire_lamp1_0v
```

### 3.4. Лист «demo_steps» для Guided Demo Mode

```
demo_id | step_num | action_type      | target_object | camera_preset | instruction_text                    | animation_name    | repeat_count
D001    | 1        | button_press     | BTN_01        | cam_btn_close | Нажмите зелёную кнопку «Пуск»      | anim_btn_press    | 3
D002    | 1        | toggle_switch    | SW_01         | cam_sw_close  | Переключите тумблер в позицию «ON»  | anim_toggle       | 3
D003    | 1        | hmi_touch        | HMI_01        | cam_hmi_close | Коснитесь иконки «Настройки» на HMI | anim_hmi_touch    | 2
D004    | 1        | mcb_on           | MCB_01        | cam_mcb_close | Поднимите рычаг автомата вверх      | anim_mcb_on       | 3
```

### 3.5. Публикация таблицы

- Google Sheets → File → Share → Publish to web → Comma-separated values (.csv)
- Для каждого листа — отдельная ссылка
- В Vectary: Variables & Data → новая вкладка → CSV from link → вставить ссылку

**Срок: 3–5 дней**
**Результат:** полная база данных проекта, подключённая к Vectary в реальном времени

---

## 4. Сборка сцены в Vectary (Этап 3)

### 4.1. Импорт и позиционирование

1. Создать новый проект в Vectary Studio
2. Импортировать каркас стенда (drag-and-drop OBJ/FBX/GLB)
3. Поочерёдно добавить остальные группы
4. Позиционировать каждую группу на своё место

### 4.2. Именование объектов (критично!)

Каждый объект в дереве сцены должен называться **точно как `id` в Google-таблице**:
- PLC_01
- HMI_01
- VFD_01
- SENSOR_01
- BTN_01
- LAMP_01
- TERM_01
- MCB_01

Это позволит функции `TRIGGER()` возвращать имя, по которому мы найдём данные в CSV.

### 4.3. Группировка и Selection

- Сгруппировать объекты логически (Ctrl+G): `Group_PLC`, `Group_Sensors`, `Group_Buttons`
- Создать **Selection «all_clickable_components»** — все компоненты, на которые можно кликнуть в Exploration Mode
- Создать **Selection «all_terminals»** — все клеммы для Practice Mode
- Создать **Selection «all_demo_targets»** — все объекты для Guided Demo

### 4.4. Камерные пресеты

Создать камеры с именами из таблицы:
- `cam_overview` — общий вид стенда
- `cam_plc` — приближение к PLC
- `cam_hmi` — приближение к HMI
- `cam_vfd` — приближение к VFD
- `cam_sensors` — зона датчиков
- `cam_buttons` — зона кнопок и ламп
- `cam_terminals` — клеммные ряды
- `cam_mcb` — автоматы

### 4.5. Материалы и освещение

- Применить PBR-материалы из библиотеки Vectary
- Настроить HDRI-окружение (нейтральное студийное)
- Проверить через Performance Analyzer: Objects, Textures, FPS

### 4.6. Подготовка «скрытых» объектов

Для Practice Mode заранее создать:
- Провода для каждого шага практики (wire_btn1_di0, wire_btn1_0v и т.д.) — **скрытые по умолчанию**
- Через Variants или Visibility сделать их невидимыми, чтобы показывать по мере прохождения шагов

**Срок: 1–1.5 недели**
**Результат:** собранная 3D-сцена с правильными именами, камерами, Selections

---

## 5. Система Variables (Этап 4)

> Центральная нервная система проекта. Всё остальное зависит от переменных.

### 5.1. Глобальные переменные

```
active_mode           (text)     = "exploration"    // текущий режим
selected_component    (text)     = ""                // id кликнутого компонента
show_card             (boolean)  = false             // показывать ли карточку
```

### 5.2. Переменные Practice Mode

```
practice_active       (boolean)  = false
current_practice      (text)     = "P001"
current_step          (number)   = 0
total_steps           (number)   = 4
practice_complete     (boolean)  = false
last_click_terminal   (text)     = ""
```

### 5.3. Переменные Guided Demo

```
demo_active           (boolean)  = false
current_demo          (text)     = "D001"
demo_step             (number)   = 0
```

### 5.4. CSV-таблицы подключены как вкладки

- Вкладка `components` → данные из Google Sheets листа components
- Вкладка `theory` → данные из листа theory
- Вкладка `practice_steps` → данные из листа practice_steps
- Вкладка `demo_steps` → данные из листа demo_steps

**Срок: 2–3 дня**
**Результат:** все переменные и данные на месте

---

## 6. Exploration Mode (Этап 5)

### 6.1. Архитектура — одна интеракция на все компоненты

**Interaction «exploration_click»:**
```
Trigger:    Click → Selection «all_clickable_components»
Condition:  active_mode == "exploration"
Actions:
  1. Set Variable «selected_component» = TRIGGER()
  2. Set Variable «show_card» = true
  3. Highlight → объект TRIGGER() → жёлтый glow
  4. Cameras → переключить на камеру (из CSV: camera_preset)
  5. Visibility → показать Floating UI «card_panel»
```

### 6.2. Универсальная карточка (один Docked Floating UI)

Один Floating UI **«card_panel»** (тип: Docked, справа), содержит:

**Блок 1 — Заголовок:**
- Text: `${components.name}` (подтягивается по selected_component)
- Text: `${components.type}`
- Image: иконка типа

**Блок 2 — Назначение:**
- Text: `${components.description}`
- Text: `${components.role}`
- Text: `Подключение: ${components.connection}`
- Text: `Тип сигнала: ${components.signal_type}`

**Блок 3 — Технические параметры:**
- Text: `Питание: ${components.power}`
- Text: `Диапазон: ${components.range}`
- Text: `Макс. ток: ${components.max_current}`
- Text: `Класс защиты: ${components.protection}`

**Блок 4 — Документация:**
- Button «Datasheet» → Link action → `${components.datasheet_url}`
- Button «Руководство» → Link action → `${components.manual_url}`
- Button «Производитель» → Link action → `${components.manufacturer_url}`

**Блок 5 — Для больших компонентов (PLC/VFD/HMI):**
- Условная видимость: `IF(components.type == "ПЛК" OR components.type == "Частотник" OR components.type == "ЧМИ")`
- Button «Открыть теорию по I/O»

**Interaction «exploration_close»:**
```
Trigger:    Click → Button «Закрыть» (внутри card_panel)
Actions:
  1. Set Variable «show_card» = false
  2. Visibility → скрыть «card_panel»
  3. Highlight → убрать подсветку
  4. Cameras → cam_overview
```

### 6.3. Итого ручной работы

| Что делаем | Количество |
|-----------|-----------|
| Floating UI (карточка) | 1 штука (универсальная) |
| Интеракции | 2 штуки (открыть + закрыть) |
| Selection | 1 штука (all_clickable_components) |
| Камеры | 8–10 пресетов |
| Данные | Google-таблица (заполняется один раз) |

**Было бы без оптимизации:** 30 Floating UI + 60 интеракций + 30 настроек подсветки

**Срок: 1–1.5 недели**
**Результат:** кликаешь на любой компонент → боковая панель с полной информацией

---

## 7. Theory Mode (Этап 6)

### 7.1. Навигация по разделам

Floating UI **«theory_menu»** (тип: Docked, слева):
- 7 кнопок (по одной на каждый раздел теории)
- Каждая кнопка = Interaction: Set Variable `selected_theory_section`

### 7.2. Универсальная карточка теории

Floating UI **«theory_card»** (тип: Docked, справа):
- Text: `${theory.title}` (по selected_theory_section)
- Text: `${theory.short_text}`
- Image: `${theory.image_url}` (загруженные PNG схемы)
- Button «Источники» → Link → `${theory.sources}`
- Container с чеклистом: `${theory.checklist}` (разбивка текста по «;»)
- Button **«Показать на стенде»**

### 7.3. Интеракция «Показать на стенде»

```
Trigger:    Click → Button «Показать на стенде»
Actions:
  1. Cameras → переключить на камеру зоны связанных компонентов
  2. Highlight → подсветить related_components из CSV
```

**Интеракций: 3 (выбор раздела + показать карточку + показать на стенде)**

**Срок: 1 неделя**
**Результат:** раздел теории с 7 темами, связанными с 3D-моделью

---

## 8. Guided Demo Mode (Этап 7)

### 8.1. Подготовка анимаций

В режиме Animate создать базовые анимации:
- `anim_btn_press` — Translation Y кнопки вниз-вверх
- `anim_toggle` — Rotation Z тумблера
- `anim_hmi_touch` — Material change (подсветка экрана HMI)
- `anim_mcb_on` — Rotation X рычага автомата

При необходимости импортировать упрощённую 3D-модель руки для визуального показа.

### 8.2. Логика пошагового показа

```
Variable: current_demo (text)
Variable: demo_step (number)

Interaction «demo_play»:
  Trigger:    Click → Button «Далее» (или автозапуск)
  Condition:  active_mode == "guided_demo"
  Actions:
    1. Cameras → камера из CSV (demo_steps.camera_preset по current_demo + demo_step)
    2. Highlight → объект demo_steps.target_object
    3. Animation → play demo_steps.animation_name (loop: repeat_count раз)
    4. Visibility → показать Floating UI с текстом инструкции
    5. Обновить текст: demo_steps.instruction_text
```

### 8.3. Floating UI инструкции

Floating UI **«demo_instruction»** (позиция: bottom center):
- Text: `Шаг ${demo_step}: ${demo_steps.instruction_text}`
- Button «Далее» → Set Variable demo_step + 1
- Button «Повторить» → replay animation
- Button «К практике» → Set Variable active_mode = "practice"

**Интеракций: 3–4 (play + next + repeat + go_to_practice)**

**Срок: 2 недели**
**Результат:** пошаговые мини-туториалы с автоматической камерой и анимацией

---

## 9. Меню и переключение режимов (Этап 8)

### 9.1. Верхнее меню

Floating UI **«main_menu»** (позиция: top center, всегда видим):
- 4 кнопки: Изучение | Практика | Теория | Демонстрация
- Активная кнопка = другой цвет (через Condition на active_mode)

### 9.2. Логика переключения

```
Interaction «switch_to_exploration»:
  Trigger:    Click → Button «Изучение»
  Actions:
    1. Set Variable «active_mode» = "exploration"
    2. Visibility → показать exploration UI, скрыть остальные
    3. Cameras → cam_overview
    4. Reset: убрать подсветки, скрыть карточки
```

Аналогично для каждого режима.

### 9.3. Кнопка «Назад»

В каждой карточке:
```
Interaction «back_button»:
  Trigger:    Click → Button «Назад»
  Actions:
    1. Visibility → скрыть текущую карточку
    2. Cameras → cam_overview
    3. Highlight → убрать все
```

### 9.4. Adaptive UI (desktop + mobile)

- Создать два варианта main_menu: desktop (горизонтальный) и mobile (иконки)
- Interaction: Trigger = On update, Condition = Breakpoint → показать нужный вариант

**Интеракций: 6 (4 переключения + назад + adaptive)**

**Срок: 3–4 дня**
**Результат:** единая навигация между всеми режимами

---

## 10. Practice Mode — MVP (Этап 9)

> Самая сложная часть. Делается последней.

### 10.1. Вариант А: внутри Vectary (no-code)

**Подготовка:**
- Каждая клемма = отдельный кликаемый объект с уникальным именем (TERM_DI_00, TERM_0V_01...)
- Selection «all_terminals» — все клеммы
- Провода-соединения заранее созданы, но скрыты (Visibility: hidden)

**Логика:**

```
Interaction «practice_click_terminal»:
  Trigger:    Click → Selection «all_terminals»
  Condition:  active_mode == "practice" AND practice_active == true
  Actions:
    1. Set Variable «last_click_terminal» = TRIGGER()

Interaction «practice_check_answer»:
  Trigger:    On update (переменная last_click_terminal изменилась)
  Condition:  last_click_terminal == practice_steps.correct_terminal[current_step]
  Actions:
    1. Visibility → показать wire_object текущего шага (провод появляется)
    2. Highlight → зелёный на подключённой клемме
    3. Set Variable current_step = current_step + 1
    4. Обновить текст инструкции

Interaction «practice_wrong_answer»:
  Trigger:    On update
  Condition:  last_click_terminal != practice_steps.correct_terminal[current_step]
              AND last_click_terminal != ""
  Actions:
    1. Highlight → красная вспышка на неправильной клемме
    2. Set Variable current_step = 0
    3. Visibility → скрыть все провода
    4. Highlight → убрать все зелёные
    5. Floating UI → «Ошибка! Начните сначала.»

Interaction «practice_complete»:
  Trigger:    On update
  Condition:  current_step >= total_steps
  Actions:
    1. Set Variable practice_complete = true
    2. Floating UI → «Практика выполнена!»
    3. Highlight → все провода зелёные
```

### 10.2. Цветовая индикация клемм

```
Interaction «highlight_inactive»:
  Condition: component_state == "inactive"  →  серый outline

Interaction «highlight_next»:
  Condition: current_step == N  →  жёлтый outline на целевой клемме

Interaction «highlight_done»:
  Condition: step_N_complete == true  →  зелёный outline

Interaction «highlight_error»:
  Condition: wrong_click == true  →  красная вспышка
```

### 10.3. Safety Screen

Floating UI **«safety_warning»** (показывается перед началом практики):
- Text: «Внимание! Убедитесь что питание отключено перед подключением»
- Button «Понял, начать» → скрыть safety_warning, Set Variable practice_active = true

### 10.4. Вариант Б: Vectary API + внешний сайт (для расширения)

Если логика станет слишком сложной для no-code:

```javascript
// На вашем сайте: слушаем клик по клемме
viewer.addEventListener("mouse_click", (result) => {
  const terminal = result.objectName;
  
  if (terminal === expectedTerminal[currentStep]) {
    viewer.sendEvent("show_wire", currentStep);
    viewer.sendEvent("highlight_green", terminal);
    currentStep++;
  } else {
    viewer.sendEvent("reset_practice");
    currentStep = 0;
  }
});
```

Требует Business план Vectary.

**Срок: 2–3 недели (MVP на 1–2 практики)**
**Результат:** рабочая практика «подключи кнопку и лампу» с проверкой правильности

---

## 11. Система состояний компонентов (сквозная)

Реализуется через Variables + Conditions + Highlight / Materials actions:

| Состояние | Variable condition | Визуал |
|-----------|-------------------|--------|
| OFF | `state == "off"` | Базовый серый материал |
| ON | `state == "on"` | Яркий/светящийся материал (Materials action) |
| ERROR | `state == "error"` | Красный outline + мигающая анимация |
| HIGHLIGHTED | `state == "highlighted"` | Жёлтый glow + затемнение остальных |
| LOCKED | `state == "locked"` | Полупрозрачный материал + клики игнорируются (Condition) |

---

## 12. Тестирование и полировка (Этап 10)

### 12.1. Чеклист

- [ ] Performance Analyzer: FPS > 24 на среднем ноутбуке
- [ ] Загрузка < 3 секунд
- [ ] Все камерные пресеты работают корректно
- [ ] Все карточки показывают правильные данные из CSV
- [ ] Переключение режимов без перезагрузки
- [ ] Practice Mode: правильный клик → зелёный провод, неправильный → сброс
- [ ] Кнопка «Назад» работает из любого состояния
- [ ] Adaptive UI: desktop + mobile
- [ ] Все ссылки на datasheet открываются в новой вкладке

### 12.2. Устройства для тестирования

- Desktop: Chrome, Firefox, Safari
- Mobile: iPhone Safari, Android Chrome
- Планшет: iPad Safari

**Срок: 1 неделя**

---

## 13. Итоговый план-график

| Этап | Что делаем | Срок | Ключевая функция Vectary |
|------|-----------|------|--------------------------|
| **1. Модель** | Упрощение, экспорт, оптимизация | 1–2 нед. | Vectary Processor / Simplify |
| **2. Данные** | Google-таблица со всем контентом | 3–5 дней | CSV Data Import |
| **3. Сцена** | Сборка, именование, камеры, Selections | 1–1.5 нед. | Selections, Camera presets |
| **4. Variables** | Глобальные переменные + подключение CSV | 2–3 дня | Variables & Expressions |
| **5. Exploration** | Универсальная карточка + 2 интеракции | 1–1.5 нед. | TRIGGER(), Docked UI |
| **6. Theory** | Меню разделов + карточка теории | 1 нед. | Floating UI, CSV lookup |
| **7. Guided Demo** | Анимации + пошаговый показ | 2 нед. | Animations timeline, Cameras |
| **8. Навигация** | Верхнее меню, переключение, Adaptive | 3–4 дня | Variants, Breakpoints |
| **9. Practice** | MVP 1–2 практики | 2–3 нед. | Variables, Conditions, API |
| **10. Тест** | Проверка, оптимизация, polish | 1 нед. | Performance Analyzer |

**Итого: 10–14 недель (2.5–3.5 месяца)**

> Сравните со старым планом: было 12–18 недель. Экономия 2–4 недели за счёт data-driven подхода.

---

## 14. Количество ручных настроек: старый план vs новый

| Элемент | Старый план (ручной) | Новый план (data-driven) | Экономия |
|---------|---------------------|-------------------------|----------|
| Floating UI карточки | 30 штук | 1 универсальная | 97% |
| Интеракции Exploration | 60+ (открыть + закрыть × 30) | 2 | 97% |
| Ввод текстов карточек | Вручную в каждой карточке | Google-таблица | 90% |
| Обновление контента | Редактировать каждый Floating UI | Изменить строку в таблице | 95% |
| Добавление компонента | Новый hotspot + UI + интеракции | Строка в CSV + объект в Selection | 90% |
| Камерные переходы | Вручную для каждого клика | Камерные пресеты + CSV lookup | 80% |

---

## 15. Минимально необходимый тариф

| Функция | Нужна? | Минимальный план |
|---------|--------|-----------------|
| Interactions, Floating UI, Hotspots | Да | Pro (Grow) |
| Variables & Expressions | Да | Pro (Grow) |
| CSV Data Import | Да | Pro (Grow) |
| Animations | Да | Pro (Grow) |
| Embed без водяного знака | Желательно | Business |
| Model API (для сложных практик) | Позже | Business |
| Vectary Processor (CAD→mesh) | Опционально | Business (по запросу) |

**Для старта достаточно Pro (Grow) плана. Business — когда дойдёте до Practice Mode с API.**
