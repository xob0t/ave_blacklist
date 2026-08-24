# План исправления 3: Кнопки в углу всей карточки + фикс toggle-визуала + отступы + автопагинция

## Список проблем и диагноз по каждой

На основе HTML-дампа пользователя, скриншотов и исследования кода:

***

### **Проблема 1. Кнопки блокировки (user/item) сдвинуты в центр — нужно сдвинуть ПРАВЕЕ под «информацию о компании» (aside-блок справа)**

#### Диагноз

По дампу HTML структуры `[data-marker="item"]`:

```
[data-marker="item"]  <-- position: relative (из userCSS.css !!!)
├── .content-*
│   ├── .slider-*    <-- фото (левая колонка)
│   ├── .body-*      <-- title/price/badges/description/date (центр)
│   │   └── …
│   │   └── .button-container   <-- ВСТАВЛЕНО В КОНЕЦ body-* (сейчас)
│   └── .aside-*                <-- seller info + rating + Написать/Телефон (ПРАВАЯ КОЛОНКА)
└── …
```

Наш `insertButtonContainer()` вставлял кнопки в конец `.body-*`, а `position: absolute; bottom:8px; right:8px` относительно ближайшего position:relative. Если у `.body-*` есть `position: relative` (мы его ставим через `ensurePositionRelative`), то **right: 8px отсчитывается от правого края body-блока (центральной колонки), а не всей карточки \[data-marker="item"]**. То есть кнопки накладываются на текст описания и не доходят до правого края (где aside).

#### Фикс (шаги 1.1–1.3)

**Шаг 1.1.** **`insertButtonContainer()`** **— вставлять строго в конец offerElement** (самой внешней `[data-marker="item"]`):

* Т.к. у offerElement гарантировано `position: relative !important` из userCSS.css, то `bottom:8px; right:8px` будет относительно **правого нижнего угла ВСЕЙ карточки** (то есть под aside-блоком «Написать / Показать телефон» — как раз куда пользователь хочет «под информацию о компании»).

**Шаг 1.2. Убрать** **`ensurePositionRelative(bodyBlock)`** **в цепочке** — больше не нужно, т.к. контекст всегда offerElement. Убираем промежуточные вставки в listTopBlock/item-location/body-\*.

**Шаг 1.3. Добавить** **`gap`** **между двумя SVG-кнопками** в CSS: у `.button-container` уже есть `display: flex`, надо добавить `gap: 8px` или больше (чтобы случайно не промахнуться между «скрыть продавца» и «скрыть объявление»).

***

### **Проблема 2. BUG: нажимаю «Скрывать резервы» / «Только из Тюмени» — логика работает (в настройках сохраняется, фильтрует), но визуально toggle остаётся СЕРЫМ (выключенным). После reload — ОК.**

#### Диагноз (3 причины, работающие вместе)

**Причина А —** **`detectToggleCheckedClass()`** **в 99% случаев возвращает несуществующий класс:**

* Функция ищет `label[data-marker="filters/localPriority/localPriority"][aria-checked="true"]`.

* У большинства пользователей default «Сначала из Тюмени» = выключен, то есть aria-checked="false".

* Следовательно, **не найдя label с** **`[aria-checked="true"]`**, детекция возвращает hardcoded-fallback `styles-module-controlledInput_checked-fJhQQ` — которого нет в новой разметке (у пользователя input имеет классы: `input-_1711db9ec52be64f input-a304188e465cde6c`).

**Причина Б — стилизация Avito теперь через** **`input:checked`** **(не через класс \_checked-\*):**

* Новый switch стилизуется CSS-правилом вида: `input[type="checkbox"]:checked ~ .toggle-a30418... > .switcherCircle-a30418... { transform: translateX(...) }`

* У нас `checkbox.checked = isEnabled` стоит, но есть баг: при `cloneNode(true)` у input-состояние `checked` (native property) не копируется 100% предсказуемо. Также `:checked` CSS иногда не применяется немедленно, если в CSS есть более высокий приоритет у другого класса.

**Причина В — нет fallback-анимации на inline-style transform circle:**

* Если ни класс, ни :checked не сработали немедленно, — у нас нет другого способа визуально включить кружок.

#### Фикс (шаги 2.1–2.4)

**Шаг 2.1. Переделать** **`detectToggleCheckedClass()`** **так, чтобы детектить класс ИЗ АТРИБУТОВ input независимо от текущего checked-состояния:**

* Берём существующий localPriority `input[type="checkbox"]` (он всегда есть).

* Смотрим на его classList, ищем по маске `_checked-`, `checked`, `toggle_checked`.

* Если класс не найден — ИЩЕМ соседний `.toggle-*` или `.wrapper_mode_switcher-*`, ищем у НЕГО класс с `checked`/`mode`.

* Если всё равно нет — выставляем флаг `_noToggleCheckedClassAvailable = true`, и далее управляем ТОЛЬКО через нативный :checked + inline-style на circle.

**Шаг 2.2. В** **`setToggleVisualState`** **гарантировать нативный state 2 способами:**

```js
if (checkbox) {
  // (а) Нативное свойство — для логики и :checked CSS-селектора
  checkbox.checked = isEnabled;
  // (б) HTML-атрибут — для :checked при клонировании, SSR, восстановления и т.д. (вместе работают надежнее)
  if (isEnabled) checkbox.setAttribute('checked', '');
  else checkbox.removeAttribute('checked');
  // (в) Форсируем change-событие? Нет — чтобы не зациклить listener.
}
```

**Шаг 2.3. В** **`setToggleVisualState`** **добавить inline-анимацию на** **`.switcherCircle-*`:**

* Ищем sibling элемент `checkbox.parentElement.querySelector('[class*="switcherCircle-"]')` или `.toggle > span`.

* Если `isEnabled` → `transform: translateX(18px)` (типичный сдвиг кружка, подбираем ширину тоггла).

* Если disabled → `transform: translateX(0)`.

* Это 100% визуальный страховой механизм, не зависящий от CSS-хешей Avito.

**Шаг 2.4. В** **`setToggleVisualState`** **— выставить label.classList toggle с классом Avito label'а aria-checked=true:**

* Иногда Avito на label вешает класс вида `root_preset_default_checked-*` или `root_checked-*`. Найти в существующем localPriority label и дублировать.

***

### **Проблема 3. Слишком близко стоят переключатели / label. Пользователь жалуется, что «первый toggle относится к "Сначала из Тюмени", а он думал что к "Только из Тюмени"» — путает соседний toggle.**

#### Диагноз

На скриншоте topPanel:
`Сначала из Тюмени ●   Только из Тюмени ●   Скрывать резервы ●`
Каждый toggle состоит из структуры: `<label>[switcherLabel-* ТЕКСТ] [wrapper+input+toggle-circle]</label>`. Avito создаёт их как flex-row в topPanel.

Проблема: когда мы делаем `insertAfterContainer.after(newContainer)` (вставка нового контейнера-переключателя СРАЗУ после существующего `switcherWrapper-*`), между ними **нет margin-left gap**. У исходного localPriority-wrapper могут быть gap от родительского flex (авито `additions-*` имеет `justify-content: space-between`). Но когда мы клонируем `switcherWrapper` и вставляем рядом, — расстояние между соседями клон/оригинал может быть нулевым или минимальным. Пользователь путается.

#### Фикс (шаг 3.1)

**Шаг 3.1.** В `insertSearchFilterToggle`: после создания `newContainer` ДО вставки применяем inline-style `marginLeft = '24px'` (или 20px — достаточно, чтобы визуально отделить). Либо в CSS отдельное правило `[data-marker="filters/cityOnly"], [data-marker="filters/hideReserved"]` (на label) → их родительский wrapper (который cloneNode) получает margin-left. Так более устойчиво к будущим правкам:

```css
/* userCSS.css */
[data-marker="filters/cityOnly"] { margin-left: 24px; }
[data-marker="filters/hideReserved"] { margin-left: 24px; }
```

(Можно также через inline-style в момент clone, чтобы не плодить CSS правила — выберем вариант inline-style в insertSearchFilterToggle.)

***

### **Проблема 4. Автопагинция НЕ работает. Приходится переключать вручную. Запрос уходит на** **`/web/1/js/items?...updateListOnly=true&spaFlow=true`.**

#### Диагноз

Селекторы пагинации в коде полностью устарели (Avito поменял префиксы + pagination теперь SPA-режим).

**Селектор PAGINATOR (L808, L916, L1012) — УСТАРЕЛ:**

* Ищем `[class*="js-pages pagination-pagination-"]`

* По пользовательскому HTML: `class="js-pages pagination-_8ed0b2da60e26425"` — префикс `pagination-` (не `pagination-pagination-`). Wildcard **не находит** → paginator «не виден» для `isPaginatorVisible()` (которая используется scroll-хендлером для триггера fetchNextPage). Соответственно автопагинка никогда не срабатывает.

**getCurrentPage() L823 — УСТАРЕЛ:**

* Ищет `[class*="styles-module-item_current-"]` span.

* В новой разметке li имеет class `listItem-*`, а активная страница — это `<a class="item-... ... item_current-..." aria-current="page">`. Current-класс теперь на `<a>` и/или есть атрибут `aria-current="page"`. Надо использовать `a[aria-current="page"]` внутри `[data-marker="pagination-button"]`.

**getNextPageUrl() L836 — УСТАРЕЛ:**

* Ищет `[data-value="${currentPage+1}"]` (цифра).

* В новой разметке и по скриншоту пользователя часто видна **ТОЛЬКО СТРЕЛКА ВПРАВО** (остальные цифры скрыты breakpoint'ами: `breakpoint_s / breakpoint_m / breakpoint_l` и т.д. — в HTML они есть, но display:none). У стрелки нет `data-value`, у неё `data-marker="pagination-button/nextPage"` и она имеет атрибут `href` (если не последняя страница).

* Если стрелка `disabled` (нет href / aria-disabled="true") — только тогда искать цифровые кнопки по data-value.

**fetchNextPage L935:**

* Делает `await fetch(nextPageUrl)`, без `Accept:text/html` заголовка. В режиме `updateListOnly=true` Avito может отвечать JSON через SPA API (content-type: application/json) → `response.text()` вернёт JSON, `DOMParser` найдёт 0 элементов → «No containers found in new page» → стоп. Надо добавить заголовок `Accept: text/html,application/xhtml+xml` чтобы Avito отдал SSR HTML, либо детектировать JSON-ответ и корректно его обрабатывать / логировать.

#### Фикс (шаги 4.1–4.5)

**Шаг 4.1. Унифицировать** **`paginatorSelector`:**

* Создать константу/функцию `getPaginator()`, которая ищет `document.querySelector('[class*="js-pages"]')` (без префикса pagination-pagination-), с fallback на `document.querySelector('[data-marker="pagination-button"]')?.closest('[class*="js-pages"]')`.

* Заменить **все 3 вхождения** селектора paginator (L808, L916, L1012) на эту функцию.

**Шаг 4.2. Переделать** **`getCurrentPage()`:**

* Приоритет 1: ссылка с `[data-marker="pagination-button"] a[aria-current="page"]` — её текстовый контент.

* Приоритет 2: искать `<a>` внутри pagination-button, у которой класс с `_current-`, и брать её текст.

* Приоритет 3 (старый): `[class*="styles-module-item_current-"] span` — fallback.

**Шаг 4.3. Переделать** **`getNextPageUrl()`:**

* Приоритет 1: стрелка `a[data-marker="pagination-button/nextPage"]` — если существует И у неё есть `href` (не disabled). Берём `href`.

* Приоритет 2: если стрелки нет или без href — ищем цифровую кнопку `a[data-value="${currentPage+1}"]` href.

* Логировать оба варианта.

**Шаг 4.4. fetch с Accept:text/html:**

* В `fetch(nextPageUrl, { headers: { 'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8' } })`.

* После `response.text()` проверить `<!DOCTYPE html` или `<html` в начале. Если нет (ответ JSON) — логировать предупреждение «получен не HTML, возможно updateListOnly» и возвращать.

**Шаг 4.5. Прокачать scroll/triггер автопагинции:**

* Текущая логика: scroll → проверка isPaginatorVisible() → fetch. Но `isPaginatorVisible()` сейчас падает без paginatorSelector. Также если Avito использует sticky filters и pagination не догружается, — IntersectionObserver на сам pagination или последнюю карточку был бы надёжнее, но это step2. В первую очередь достаточно починить селекторы (шаги 4.1–4.4) — 99% кейсов покрывается.

***

## Список файлов и сводка изменений

1. **`userCSS.css`**:

   * `.button-container { display: flex; }` → добавить `gap: 8px` (пространство между 2 кнопками)

   * Опционально margin-left для toggle'ов (через data-marker label'ов) — но лучше inline-style из JS

2. **`contentScript.js`**:

   * **insertButtonContainer():** убрать body/itemLocation/favoriteCartInsertion, всегда `offerElement.appendChild(container)` + `ensurePositionRelative(offerElement)` (гарантия контекста ВСЕЙ карточки)

   * **detectToggleCheckedClass():** переработать: брать classList input localPriority и искать \_checked-\* независимо от aria-checked state; если нет — искать у соседнего toggle/wrapper; если нет — выставить флаг \_noClassAvailable

   * **setToggleVisualState():** добавить checkbox.setAttribute('checked') / removeAttribute; добавить inline transform на \[class\*="switcherCircle-\*"] как гарантию визуала

   * **insertSearchFilterToggle():** newContainer.style.marginLeft = '24px' перед вставкой (отделяет наши 2 свитчера от соседних)

   * **getPaginator helper:** вместо `[class*="js-pages pagination-pagination-"]` использовать querySelector('\[class\*="js-pages"]') + fallback на data-marker pagination-button

   * **getCurrentPage():** искать a\[aria-current="page"] внутри pagination (новый селектор), с fallback на старые

   * **getNextPageUrl():** приоритет `a[data-marker="pagination-button/nextPage"]` со своим href; fallback на старый `[data-value="${current+1}"]`

   * **fetchNextPage():** fetch с header Accept:text/html; проверка что ответ HTML; логирование если ответ JSON

***

## Риски и обработка

| Риск                                                                                     | Вероятность | Обработка                                                                                                                                                           |
| ---------------------------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| switcherCircle не найден (новые хеши) → визуал через inline не сработает                 | Средняя     | Ищем несколькими селекторами: \[class\*="switcherCircle-"], \[class\*="toggle-"] > span, \[class\*="circle-"]; fallback — всегда checkbox.checked + атрибут checked |
| margin-left 24px слишком большой / маленький                                             | Низкая      | Выбираем 22–24px; если мало/много пользователь попросит подкорректировать                                                                                           |
| fetch с Accept:text/html меняет поведение Avito SPA                                      | Низкая      | В ответе на paginated page = ссылку с ?p=2 всегда html; не трогаем JS API                                                                                           |
| offerElement.appendChild вызовет overflow и кнопки будут упираться в скругления карточки | Низкая      | right:8px bottom:8px достаточно; у нас \[data-marker="item"] имеет position:relative, всё ок                                                                        |
| gap: 8px между кнопками блокировки недостаточно                                          | Низкая      | Можно всегда увеличить в CSS без JS-правок                                                                                                                          |

***

## Порядок выполнения

1. ✅ **Шаг 1 (кнопки правее):** переработать `insertButtonContainer()` → всегда offerElement.appendChild + gap в CSS
2. ✅ **Шаг 2 (toggle визуал):** `detectToggleCheckedClass`, `setToggleVisualState` + setAttribute('checked') + inline transform на switcherCircle
3. ✅ **Шаг 3 (отступы toggle):** `insertSearchFilterToggle newContainer.style.marginLeft = '24px'`
4. ✅ **Шаг 4 (автопагинция):** getPaginator/getCurrentPage/getNextPageUrl селекторы + Accept header
5. ✅ Валидация синтаксиса, диагностика

