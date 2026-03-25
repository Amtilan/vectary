# Функции Vectary для повышения эффективности работы

> Что конкретно изучить и использовать, чтобы не тонуть в ручной работе

---

## Главная проблема: ручная работа

У вас ~20–30 компонентов на стенде. Для каждого нужно: хотспот, карточка (Floating UI), интеракция (клик → показать карточку), подсветка, камера, анимация. Если делать «в лоб» — это 5–7 действий на каждый компонент, умноженные на 4 режима. Получается 400–600 ручных настроек.

Ниже — конкретные функции Vectary, которые эту работу сокращают в разы.

---

## 1. CSV Data Import + Variables (главный ускоритель)

**Что это:** Vectary позволяет импортировать данные из CSV-файла или Google Sheets напрямую в переменные проекта. Данные из Google Sheets обновляются в реальном времени.

**Как подключить:**
- Variables & Data → новая вкладка → CSV from file / CSV from link (Google Sheet)
- Для Google Sheets: File → Share → Publish to web → Comma-separated values (.csv) → скопировать ссылку

**Как это экономит время:**

Вместо того чтобы вручную вбивать текст в каждую карточку Floating UI, вы заполняете одну Google-таблицу со всеми компонентами:

```
component_id | name           | type      | description              | power   | signal_type | datasheet_url
PLC_01       | Siemens S7-1200| ПЛК       | Основной контроллер...   | 24V DC  | Digital     | https://...
HMI_01       | KTP700 Basic   | ЧМИ       | Панель оператора...      | 24V DC  | Ethernet    | https://...
VFD_01       | Sinamics V20   | Частотник  | Управление двигателем... | 380V AC | Analog      | https://...
SENSOR_01    | SICK IME12     | Датчик     | Индуктивный датчик...    | 24V DC  | DI          | https://...
```

Потом в Floating UI Text элементах используете выражения вида:

```
${components.name}
${components.description}
${components.power}
```

**Где в документации:** `help.vectary.com/documentation/3d-configurator/variables-and-expressions/data-import-.csv`

**Экономия:** вместо ручного редактирования 30 карточек → заполнили 1 таблицу. Обновили таблицу → все карточки обновились автоматически.

---

## 2. TRIGGER() + OBJ() функции (одна интеракция на все компоненты)

**Что это:** Функция `TRIGGER()` возвращает имя объекта, на который кликнули. Функция `OBJ("name")` получает ссылку на объект по имени.

**Как это экономит время:**

Вместо создания 30 отдельных интеракций (по одной на каждый компонент), вы создаёте **одну универсальную интеракцию**:

```
Trigger:  Click → Selection (все компоненты)
Action:   Set Variable "selected_component" = TRIGGER()
Action:   Visibility → показать универсальный Floating UI
```

А в Floating UI текстовые поля подтягивают данные из CSV по имени выбранного компонента:

```
IF(selected_component == "PLC_01", components[0].name, 
IF(selected_component == "HMI_01", components[1].name, ""))
```

**Где в документации:** `help.vectary.com/documentation/3d-configurator/variables-and-expressions/functions`

**Экономия:** 1 интеракция вместо 30. Добавляете новый компонент → просто добавляете строку в CSV и объект в Selection.

---

## 3. Selections (группировка триггеров)

**Что это:** Selections позволяют объединить несколько объектов из разных групп в одну логическую выборку, которую можно использовать как единый триггер.

**Как использовать:**
- Выберите нужные объекты через Ctrl + клик на левой панели
- Нажмите «+» для создания Selection
- Используйте эту Selection как триггер в Interaction

**Как это экономит время:**

Вместо: 30 интеракций × «Click на PLC», «Click на HMI», «Click на VFD»...

Делаете: 1 Selection «all_components» → 1 интеракция «Click на all_components» → переменная `TRIGGER()` определяет, что именно кликнули.

**Где в документации:** `help.vectary.com/documentation/3d-configurator/interactions/triggers`

---

## 4. Interaction Grouping (порядок в хаосе)

**Что это:** Интеракции можно группировать в папки через правый клик → Group.

**Как использовать:**
- Правый клик на интеракции → Create group
- Перетащите связанные интеракции в группу
- Можно разгруппировать или удалить вместе с содержимым

**Зачем нужно:**

Когда у вас 4 режима × N интеракций, без группировки вы утонете в списке. Группируйте по режимам:
- `[Exploration] Click → Show card`
- `[Theory] Menu → Show section`
- `[Demo] Step → Next animation`
- `[Practice] Click terminal → Check answer`

**Где в документации:** `help.vectary.com/documentation/3d-configurator/interactions`

---

## 5. Variants (переключение сцен без перезагрузки)

**Что это:** Variants группируют объекты, показывая только один из них в момент времени. Переключение мгновенное.

**Как использовать для вашего проекта:**

Создайте Variant-группу для состояний проводов:
- `wire_1_off` (невидимый провод)
- `wire_1_on` (видимый зелёный провод)
- `wire_1_error` (красный провод)

Переключение через Interaction action «Variants → Object: wire_1_on»

**Также для режимов UI:**
- Variant с разными Floating UI layout для каждого режима
- Вместо скрытия/показа десятков элементов → переключение одного Variant

**Где в документации:** `help.vectary.com/documentation/3d-configurator/variants`

**Экономия:** одно действие переключает целое состояние сцены, вместо 10 отдельных Visibility actions.

---

## 6. Adaptive UI через Breakpoints (desktop + mobile за один раз)

**Что это:** Vectary позволяет создать два варианта UI (desktop и mobile) и автоматически переключать их по размеру экрана.

**Как настроить:**
- Создайте два Floating UI: один для десктопа, один для мобильного
- Interaction: Trigger = On update, Condition = Breakpoint, Action = Show desktop UI / Hide mobile UI (и наоборот)

**Зачем нужно:** не придётся делать два отдельных проекта для разных устройств.

**Где в документации:** `help.vectary.com/documentation/3d-configurator/floating-ui/tips`

---

## 7. Docked UI (боковая панель без перекрытия модели)

**Что это:** Floating UI может быть двух типов — Floating (поверх сцены) и Docked (рядом со сценой, не перекрывает 3D).

**Почему критично для вас:** ТЗ прямо говорит — «боковая панель, не popup поверх центра, не перекрывать модель».

**Как настроить:**
- Выберите Floating UI → в настройках справа → Type: Docked
- Позиция: Right или Left
- Ширина: фиксированная или Fill

**Где в документации:** `help.vectary.com/documentation/3d-configurator/floating-ui/floating-ui-settings`

---

## 8. Object Dropper (быстрый выбор объектов в интеракциях)

**Что это:** Инструмент для выбора объектов прямо на сцене, вместо поиска по списку.

**Как использовать:**
- В режиме Interactions, при назначении триггера или action
- Нажмите иконку Object Dropper
- Кликните на нужный объект прямо на 3D-сцене

**Экономия:** когда у вас 100+ объектов в дереве, искать нужный по имени — мучение. Object Dropper: клик — готово.

**Где в документации:** `help.vectary.com/documentation/3d-configurator/interactions`

---

## 9. Highlight action с Condition (цвета по состоянию)

**Что это:** Highlight позволяет визуально выделить объект outline или color fill. В сочетании с Conditions и Variables — автоматическая подсветка по состоянию.

**Как сделать цветовую индикацию из ТЗ одной системой:**

```
Variable: component_state (text)

Interaction 1:
  Trigger: On update
  Condition: component_state == "inactive"
  Action: Highlight → серый outline на объекте

Interaction 2:
  Trigger: On update
  Condition: component_state == "next_step"
  Action: Highlight → жёлтый outline

Interaction 3:
  Trigger: On update
  Condition: component_state == "done"
  Action: Highlight → зелёный outline

Interaction 4:
  Trigger: On update
  Condition: component_state == "error"
  Action: Highlight → красный outline
```

**Экономия:** вместо вручную менять цвета → меняете одну переменную, и все подсветки обновляются сами.

---

## 10. Cameras (пресеты камер для авто-навигации)

**Что это:** В Vectary можно создать несколько камер с разными позициями и переключаться между ними через Interactions.

**Как использовать:**
- Создайте камеру для каждой зоны стенда: `cam_PLC`, `cam_HMI`, `cam_VFD`, `cam_overview`
- В Interaction: Action → Cameras → переключить на `cam_PLC`
- При клике на компонент → камера автоматически приближается

**Экономия:** не нужно вручную анимировать каждое перемещение камеры. Одно действие = мгновенный переход.

---

## 11. Project Cloning (шаблоны для будущих стендов)

**Что это:** Готовый проект можно клонировать в другой workspace. Другие пользователи могут скопировать ваш проект к себе.

**Зачем:**
- Сделали один стенд → клонировали → заменили модель и CSV → получили второй стенд
- Вся логика интеракций, Floating UI, Variables сохраняется
- Студенты могут клонировать проект для самостоятельной работы

**Где в документации:** `help.vectary.com/documentation/sharing-exporting-embedding/project-cloning`

---

## 12. Version History (откат ошибок)

**Что это:** Vectary автоматически сохраняет версии проекта. Можно вернуться к предыдущей версии.

**Зачем:** когда настраиваете 50 интеракций и что-то сломалось — откатились на рабочую версию, а не начинаете заново.

---

## 13. Performance Analyzer (чтобы не гадать, почему тормозит)

**Что это:** Встроенный анализатор в Preview режиме, который показывает: количество объектов, полигонов, текстур, время загрузки, FPS, какие эффекты тормозят.

**Как использовать:** Preview → правая панель → Objects / Textures / Loading time / Performance

**Зачем:** если стенд тормозит, анализатор точно скажет что убрать (слишком тяжёлая HDRI, лишние Soft Shadows, избыточные текстуры).

---

## Сводная таблица: что изучить в первую очередь

| Приоритет | Функция | Что даёт | Сколько экономит |
|-----------|---------|---------|-----------------|
| 1 | CSV Data Import + Google Sheets | Все тексты карточек из одной таблицы | 70% ручного ввода текстов |
| 2 | TRIGGER() + Selections | Одна интеракция вместо 30 | 90% настройки кликов |
| 3 | Variables + Expressions | Динамический контент, состояния | 60% логики Practice Mode |
| 4 | Docked UI | Боковая панель как в ТЗ | Сразу правильный layout |
| 5 | Variants | Мгновенное переключение состояний | 50% анимационной работы |
| 6 | Camera presets | Авто-навигация по клику | 80% работы с камерой |
| 7 | Interaction Grouping | Порядок в 100+ интеракциях | Сохраняет рассудок |
| 8 | Object Dropper | Быстрый выбор объектов | 30% кликов по UI |
| 9 | Performance Analyzer | Понимание что тормозит | Часы дебага |
| 10 | Project Cloning | Шаблон для будущих стендов | Весь проект для следующего раза |

---

## Рекомендуемый порядок изучения

**Неделя 1: Фундамент**
- Изучите Variables & Expressions (синтаксис, функции IF, TRIGGER, OBJ)
- Подготовьте Google-таблицу со всеми компонентами
- Подключите CSV Import к проекту

**Неделя 2: Интеракции**
- Создайте Selections для групп компонентов
- Настройте одну универсальную интеракцию с TRIGGER()
- Разберитесь с Docked UI и Floating UI settings

**Неделя 3: Контент**
- Заполните все карточки через выражения (данные из CSV)
- Настройте камерные пресеты для каждой зоны
- Сделайте Variants для состояний компонентов

**Неделя 4: Режимы**
- Соберите переключение режимов через верхнее меню
- Группируйте интеракции по режимам
- Тестируйте через Performance Analyzer

---

## Ключевые ссылки на документацию

| Раздел | URL |
|--------|-----|
| Variables & Expressions | help.vectary.com/documentation/3d-configurator/variables-and-expressions |
| Функции (IF, TRIGGER, OBJ) | help.vectary.com/documentation/3d-configurator/variables-and-expressions/functions |
| Синтаксис выражений | help.vectary.com/documentation/3d-configurator/variables-and-expressions/syntax |
| CSV Data Import | help.vectary.com/documentation/3d-configurator/variables-and-expressions/data-import-.csv |
| Interactions (обзор) | help.vectary.com/documentation/3d-configurator/interactions |
| Triggers (включая Selections) | help.vectary.com/documentation/3d-configurator/interactions/triggers |
| Actions (все типы) | help.vectary.com/documentation/3d-configurator/interactions/actions |
| Conditions | help.vectary.com/documentation/3d-configurator/interactions/conditions |
| Floating UI | help.vectary.com/documentation/3d-configurator/floating-ui |
| Floating UI настройки (Docked) | help.vectary.com/documentation/3d-configurator/floating-ui/floating-ui-settings |
| Floating UI Tips (Adaptive) | help.vectary.com/documentation/3d-configurator/floating-ui/tips |
| Hotspots | help.vectary.com/documentation/3d-configurator/hotspots |
| Variants | help.vectary.com/documentation/3d-configurator/variants |
| Animations | help.vectary.com/documentation/3d-configurator/animations |
| Оптимизация (Performance) | help.vectary.com/3d-modeling-blog/optimizing-3d-models-for-the-web |
| Project Cloning | help.vectary.com/documentation/sharing-exporting-embedding/project-cloning |
| Model API (для Practice Mode) | help.vectary.com/api/model-api |
| API Events & Listeners | help.vectary.com/events-listeners |
