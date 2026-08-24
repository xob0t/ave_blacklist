# План исправления: Адаптация расширения Avito Blacklist к новой разметке Avito

## Диагностика проблемы

По логам пользователя и образцу новой разметки Avito выявлены следующие несовместимости:

### 1. Селектор верхней панели (устарел)

* **Текущий селектор (L1192, L2696):** `[class*="index-topPanel-"]`

* **Новая разметка:** `class="topPanel-_3dc380055c8f21b0 topPanelStatic-_3dc380055c8f21b0"` — префикс изменился на `topPanel-` (без `index-`)

* **Лог подтверждает:** `Верхняя панель не найдена для вставки переключателя filters/cityOnly` и `filters/hideReserved`

### 2. Селектор текста переключателя (устарел CSS-хеш)

* **Текущий (L1128, L1224, L1264):** `.filters-switcherLabel-vbkFI`

* **Новая разметка:** `class="switcherLabel-dba2ea368dca1406"` — префикс изменился, хеш другой

### 3. Функция поиска родительского контейнера для переключателей

* **Текущие селекторы (L1204-1210):** `[class*="styles-module-theme-"]` и `[class*="filters-subscription-additions-"]`

* **Новая структура:** `label` → `div.switcherWrapper-*` → `div.additions-*` → `div.theme-provider-*`

* Ни один из текущих селекторов не находит нужный контейнер

### 4. Класс активного состояния переключателя (устарел CSS-хеш)

* **Текущий (L1082):** `styles-module-controlledInput_checked-fJhQQ`

* Хеш `fJhQQ` изменится в новой разметке

### 5. Позиционирование кнопок блокировки (пропали / смещены)

**Причина 5а — CSS-селекторы позиционирования (userCSS.css L12-18):**

* **Текущие:** `[class*='items-galleryItem']` (top: 0) и `[class*='items-listItem']` (bottom: 0)

* **Новые классы:** `listItem-_9e71b67936344c85`, `list-_7501b6e172403284` — старые селекторы не срабатывают, поэтому для `position: absolute` не указаны `top`/`bottom`

**Причина 5б — Отсутствует** **`position: relative`** **у карточки:**

* Для `position: absolute; right: 0` у `.button-container` нужен предок с `position != static`

* В новой разметке у `[data-marker="item"]` может не быть `position: relative`, поэтому кнопки позиционируются относительно body и улетают со страницы

**Причина 5в — Точка вставки кнопок (L1654-1658):**

* `offerElement.appendChild()` — кнопки вставляются в самый конец сложной flex-структуры карточки

* Более надежная точка — рядом с иконкой избранного (`.favoriteCartIcons-*` / `[data-marker="favorites-add"]`)

### 6. Контейнер скрытых объявлений (createHiddenContainer, L1560-1561)

* Вставляется в `offersRoot` (`elementtiming="bx.catalog.container"`)

* Внутри него есть `#bx_serp-item-list` с `data-marker="catalog-serp"` — скрытые объявления лучше вставлять после этого списка, а не в общий корень

***

## План изменений по файлам

### Файл 1: `contentScript.js`

#### Шаг 1.1 — Улучшить селектор верхней панели

* **Место L1192:** Заменить единичный селектор на функцию-поиск:

  ```
  function findTopPanel() {
    return document.querySelector('[class*="topPanel-"]') ||
           document.querySelector('[class*="index-topPanel-"]') ||
           document.querySelector('[data-marker="view-change"]')?.closest('[class*="topPanel"]');
  }
  ```

* Использовать `findTopPanel()` вместо `querySelector('[class*="index-topPanel-"]')`

#### Шаг 1.2 — Обновить MutationObserver на появление панели

* **Место L2696:** Добавить проверку `"topPanel-"` помимо `"index-topPanel-"`

#### Шаг 1.3 — Унифицировать селектор текста переключателя

* **Места L1128, L1224, L1264:** Заменить `.filters-switcherLabel-vbkFI` на универсальный `[class*="switcherLabel-"]`

* Добавить fallback: поиск текстового узла внутри контейнера `.wrapper_mode_switcher-*` рядом с `input[type="checkbox"]`

#### Шаг 1.4 — Обновить `getToggleHostContainer()` (L1204-1210)

Добавить новые селекторы (по приоритету):

1. `label.closest('[class*="switcherWrapper-"]')` — прямой родитель свитчера
2. `label.closest('[class*="additions-"]')` — контейнер доп.элементов
3. `label.closest('[class*="theme-provider-"]')` — провайдер темы
4. Оставить существующие + `parentElement` как fallback

#### Шаг 1.5 — Убрать зависимость от хеша `TOGGLE_CHECKED_CLASS` (L1082)

* **Стратегия:** Управлять визуалом через `aria-checked` и прямой установкой `.checked`, не полагаясь на CSS-класс с хешем от Avito

* В `setToggleVisualState()` (L1158-1166): Установить checkbox.checked = true/false, а `aria-checked` на label — этого достаточно, т.к. Avito использует нативный input + стилизует через соседние селекторы

* Переменную `TOGGLE_CHECKED_CLASS` оставить как fallback, но добавить логику: если не нашли класс с таким суффиксом, поискать вариант по `_checked-`

#### Шаг 1.6 — Улучшить `insertButtonContainer()` (L1654-1658)

Вместо `offerElement.appendChild()` делать:

1. Найти в карточке зону избранного: `offerElement.querySelector('[class*="favoriteCartIcons-"]')` или `offerElement.querySelector('[data-marker="favorites-add"]')`
2. Взять её родительский элемент (там где `body-*`)
3. Если нашли — вставить контейнер в конец этого body-блока (всегда внутри `position: relative` области)
4. Гарантировать `position: relative` у контекста (установить inline-стиль если `getComputedStyle` показывает `static`)
5. Если ничего не нашли — fallback как сейчас (`offerElement.appendChild()`), но сразу выставить `offerElement.style.position = 'relative'`

***

### Файл 2: `userCSS.css`

#### Шаг 2.1 — Обновить CSS-селекторы для позиционирования кнопок

Заменить блоки:

```css
[class*='items-galleryItem']:hover .button-container { top: 0; }
[class*='items-listItem']:hover .button-container { bottom: 0; }
```

На:

```css
/* Новые селекторы (суффиксы классов Avito) */
[class*='listItem-']:hover .button-container,
[class*='items-listItem']:hover .button-container {
  top: 8px;
  right: 8px;
}
[class*='galleryItem-']:hover .button-container,
[class*='items-galleryItem']:hover .button-container {
  top: 8px;
  right: 8px;
}
/* Универсальное правило: позиция top (работает при любой раскладке) */
[data-marker="item"]:hover .button-container {
  top: 8px;
  right: 8px;
}
```

#### Шаг 2.2 — Гарантировать position: relative у карточки товара

Добавить:

```css
/* Гарантия контекста позиционирования для кнопок */
[data-marker="item"] {
  position: relative !important;
}
```

#### Шаг 2.3 — Скрытый контейнер: исправить возможные layout-сдвиги

У `.hidden-container` уже есть `margin-top: 20px`, это нормально. Проверить, чтобы `display: flex; flex-wrap: wrap` не ломал layout — оставить как есть.

***

### Шаг 3 (общий для обоих файлов) — Проверка связанных функций

#### 3.1 createHiddenContainer (L1534-1563)

* Если в `offersRoot` (elementtiming="bx.catalog.container") есть `[data-marker="catalog-serp"]` или `#bx_serp-item-list`, вставлять `<hr>` и `<details>` **после** этого списка, а не в самый конец.

* Это гарантирует, что скрытые объявления визуально будут после реальных, а не где-то в середине разметки

#### 3.2 Детекция города из переключателя (getCityDisplayName L1127-1140)

* После замены селектора на `[class*="switcherLabel-"]` должно заработать.

* Добавить fallback: брать label из `label[data-marker="filters/localPriority/localPriority"] .switcherLabel-*` напрямую

#### 3.3 Вставка переключателей (insertSearchFilterToggle)

* После исправления topPanel, getToggleHostContainer и label-селектора — должно заработать.

* **Важно:** функция клонирует существующий DOM-узел (L1218: `parentContainer.cloneNode(true)`). В клоне будут старые классы-хеши склонированного узла, а не новые из родителя. После клонирования нужно:

  * Найти label в клоне и перезаписать `data-marker` (уже делается L1221)

  * Найти label-text заново универсальным селектором `[class*="switcherLabel-"]` (уже будет обновлено L1224)

  * Найти input-checkbox (уже делается L1229)

  * Найти switcher-кружок/тоггл — останутся старыми, но это ок, т.к. клонированный элемент копирует стили родного Avito-свитчера

#### 3.4 TOGGLE\_CHECKED\_CLASS fallback (L1082 + использование в setToggleVisualState L1161)

* Идея: вместо жёстко зашитого класса при инициализации **один раз просканировать DOM** в поисках существующего переключателя `filters/localPriority` и определить его класс "checked":

  ```
  function detectToggleCheckedClass() {
    const checkedLabel = document.querySelector('label[data-marker="filters/localPriority/localPriority"][aria-checked="true"]');
    if (checkedLabel) {
      const input = checkedLabel.querySelector('input[type="checkbox"]');
      if (input) {
        const checkedClass = [...input.classList].find(c => c.includes('_checked-') || c.includes('checked'));
        if (checkedClass) return checkedClass;
      }
    }
    return null;
  }
  ```

* Если динамически определился — использовать его; иначе не добавлять класс вовсе (полагаться на `checked` и `aria-checked`).

***

## Риски и их обработка

| Риск                                                            | Вероятность | Обработка                                                                                                                            |
| --------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Avito снова поменяет хеши CSS-классов                           | Высокая     | Везде, где возможно, использовать `[class*="prefix-"]` вместо полного имени; использовать data-marker атрибуты как основной источник |
| Клонирование переключателя сломает стили                        | Средняя     | После cloneNode проставить атрибуты вручную; сохранить aria-patternы                                                                 |
| position: relative!important сломает внутреннюю раскладку Avito | Низкая      | Карточка товара обычно сама имеет position: relative; это страховка                                                                  |
| MutationObserver ловит слишком много нод и дублирует UI         | Средняя     | Проверка `existingToggle` (L1186) уже есть; гарантировать, что она работает                                                          |

***

## Список файлов для изменения

1. **`contentScript.js`** — основные селекторы (topPanel, switcherLabel, hostContainer), позиционирование кнопок, детекция checked-class, hidden-container insert point
2. **`userCSS.css`** — CSS-селекторы позиционирования `.button-container`, гарантия position:relative на карточке

***

## Порядок выполнения

1. ✅ Поправить `userCSS.css` (селекторы + position:relative) — даст мгновенный эффект для кнопок
2. ✅ Поправить `findTopPanel()` и MutationObserver — переключатели начнут вставляться
3. ✅ Поправить `[class*="switcherLabel-"]` и `getToggleHostContainer()` — текст и место вставки переключателей
4. ✅ Детекция `TOGGLE_CHECKED_CLASS` динамически
5. ✅ Улучшить `insertButtonContainer()` (вставка рядом с избранным + гарантия relative)
6. ✅ `createHiddenContainer()` — вставка после serp-list
7. ✅ Регрессионная проверка: страница товара, страница продавца, рекомендации (главная) не сломаны

