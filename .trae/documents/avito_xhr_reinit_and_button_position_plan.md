# План исправления 2: Переинициализация после XHR-запроса /web/1/js/items и перенос кнопок блокировки вниз-вправо

## Задача 1. Наши UI-элементы пропадают после AJAX-запроса `/web/1/js/items` (подгрузка объявлений / переключение страниц / смена фильтров)

### Диагноз

По логам пользователя: ответ `/web/1/js/items` приходит с `{count: 0, catalog: {items: [...]}}` и далее:

* В консоли остаётся только 1 родной свитчер «Сначала из Тюмени»

* Наши свитчеры «Только из Тюмени» / «Скрывать резервы» пропадают

* Кнопки блокировки на новых карточках отсутствуют

* Карточки товаров в списке обновлены (старые удалены, новые вставлены)

**Почему текущий MutationObserver не ловит:**

1. Обрабатывает **только добавленные** ноды (`addedNodes`), а при XHR-рендере Avito может **заменять внутренности** (удаляет старых детей topPanel/catalog-serp и добавляет новых). Если addedNode — это не сам `topPanel-*`, а глубокая нода внутри, мы её пропускаем.
2. Есть 4 условия запуска переинициализации search-страницы:

   * нода с `elementtiming="bx.catalog.container"` (только при SSR/первой загрузке)

   * нода с классом `styles-singlePageWrapper` (редко срабатывает при XHR)

   * нода с классом `topPanel-` / `index-topPanel-` (только если сам контейнер topPanel добавлен заново, а не его внутренности)

   * нода `<script>` с `abCentral` — при fetch/XHR `/web/1/js/items` скрипт не вставляется, ответ приходит как JSON
3. `catalogData` при XHR-запросе остаётся старым / пустым, т.к. новый парсинг запускается только через script-тег abCentral. Из-за этого `processSearchPageNow` в строке `if (!catalogData) return;` делает **early-exit**, и кнопки не вешаются даже если MutationObserver дошёл до него.
4. Даже если `processSearchPage()` сработает — если Avito полностью ре-рендерит внутренности `topPanel`, наши клонированные свитчеры `[data-marker="filters/cityOnly"]` и `filters/hideReserved` **физически удалены из DOM**, но мы не проверяем, существуют ли они до сих пор (только при `existingToggle` при вставке — то есть если вставка уже происходит, а не когда они удалены).

### Список фиксов по Задаче 1

#### Шаг 1.1. Проверка/восстановление наших toggle'ов после ЛЮБОЙ childList-мутации

Добавить debounced-функцию `ensureSearchTogglesPresent()`:

```
function ensureSearchTogglesPresent() {
  const city = document.querySelector(`[data-marker="${CITY_FILTER_MARKER}"]`);
  const hid  = document.querySelector(`[data-marker="${HIDE_RESERVED_FILTER_MARKER}"]`);
  if (!city || !hid) {
    insertCityFilterToggle();
    insertHideReservedToggle();
  }
}
```

Вызвать её в **самом конце** цикла по `mutation.addedNodes` в ветке search-page **без привязки к условиям**. Или ещё надёжнее: обернуть в собственный debounce (300мс) и вызывать изнутри общего обработчика MutationObserver на каждую childList-мутацию на странице поиска.

Почему это работает: если Avito удалил наш свитчер из DOM при ре-рендере topPanel — эта функция увидит, что его нет, и вставит заново.

#### Шаг 1.2. Обрабатывать появление новых карточек `[data-marker="item"]` при подгрузке / сортировке / фильтрации

В ветке search-page MutationObserver'а (внутри addedNodes.forEach) **добавить новое условие**:

```
// Вариант А: сама добавленная нода — карточка товара
const isNewItem = node instanceof Element && node.getAttribute && node.getAttribute('data-marker') === 'item';
// Вариант Б: добавленная нода — контейнер, внутри которого новые карточки (например, serp-list inner)
const hasNewItems = node instanceof Element && node.querySelector && node.querySelector('[data-marker="item"]');
if (isNewItem || hasNewItems) {
  // Актуализируем catalogData через DOM-парсер (на случай если XHR не дал нам abCentral)
  if (!catalogData || catalogData.length === 0) {
    setCatalogData(getCatalogDataAlternative());
  }
  // Гарантируем существование свитчеров
  scheduleEnsureToggles();
  processSearchPage();
}
```

Это покрывает сценарий «заменили детей catalog-serp внутри существующего offersRoot».

#### Шаг 1.3. Убрать early-exit в `processSearchPageNow` при пустом `catalogData`, но полной DOM'е

Сейчас в `processSearchPageNow()` первая строка: `if (!catalogData) return;`. Если карточки уже есть в DOM (например, после XHR), а catalogData не подтянулся из-за отсутствия скрипта abCentral — мы ничего не обрабатываем.

Исправление: в начале processSearchPageNow пробовать «добрать» catalogData из DOM:

```
function processSearchPageNow() {
  if (!catalogData || catalogData.length === 0) {
    const fromDom = getCatalogDataAlternative();
    if (fromDom && fromDom.length > 0) setCatalogData(fromDom);
  }
  if (!catalogData || catalogData.length === 0) {
    // Всё равно ничего нет — попытка последнего шанса: парсим DOM прямо сейчас
    // извлекая offerId и userId из атрибутов data-item-id и ссылок /user/
    // и применяем кнопки только по тому что есть в blacklistUsers/blacklistOffers
    // — будем делать в applyOfferState fallback без catalog entry
  }
  dedupeAllHiddenClones();
  pruneReservedCaches();
  const offerElements = document.querySelectorAll(offersSelector);
  for (const offerElement of offerElements) {
    if (isHiddenOriginal(offerElement)) continue;
    processOfferElement(offerElement);
  }
}
```

#### Шаг 1.4. Улучшить `processOfferElement` — работать без записи в `catalogData` (fallback по DOM)

Сейчас `processOfferElement` вызывает `getCatalogItemByOfferId`, потом `extractUserIdFromCatalogItem`, и только при неудаче — `extractUserIdFromOfferElement`. Это уже хорошо, но нужно добавить гарантию: если `userId` определился из DOM, а currentOfferData пустой — всё равно вызывать `applyOfferState(offerElement, { offerId, userId })`.

Также проверить `extractUserIdFromOfferElement` — если он опирается на старые классы, добавить `[data-marker="item"] a[href*="/user/"]`, `[data-marker="item"] a[href*="/brands/"]` — использовать href из title-ссылки или искать любую ссылку внутри карточки с `/user/`.

#### Шаг 1.5. (Опционально) Monkey-patch fetch/XHR, чтобы ловить ответ `/web/1/js/items` напрямую

Если MutationObserver не покрывает 100% (например, Avito делает replaceChild одного serp-элемента на другой, и mutationObserver доходит, но с большой задержкой), можно добавить безопасный monkey-patch `window.fetch` и `XMLHttpRequest.prototype` на стороне contentScript'а через injected-script. Но это рискованно и сложнее — **реализуем только если пункты 1.1–1.4 не сработают**. В текущем плане — вторично, как запасной вариант.

***

## Задача 2. Кнопки блокировки должны быть **снизу справа** (а не сверху)

### Диагноз

По скриншоту пользователя 1: сейчас кнопки (глаз и человек с крестиком) расположены **над зоной рейтинга/имени продавца** — перекрывают полезную визуальную зону. Требуется переместить их в **правый нижний угол карточки**, где находится свободное место (под блоком «Доставка от 1 дня» / «7 дней назад»).

### Список фиксов по Задаче 2

#### Шаг 2.1. `userCSS.css` — заменить `top: 8px` → `bottom: 8px` во всех hover-правилах

Текущие правила:

```
[class*='listItem-']:hover .button-container  { top: 8px; right: 8px; }
[class*='items-listItem']:hover .button-container { top: 8px; right: 8px; }
[class*='galleryItem-']:hover .button-container { top: 8px; right: 8px; }
[class*='items-galleryItem']:hover .button-container { top: 8px; right: 8px; }
[data-marker="item"]:hover .button-container { top: 8px; right: 8px; }
```

Заменить **во всех 5 правилах** `top: 8px` → `bottom: 8px`.

Также **сделать универсальное правило главным** (самое специфичное/последнее):

```
[data-marker="item"]:hover .button-container {
  position: absolute;
  display: flex;
  z-index: 10000;
  bottom: 8px;
  right: 8px;
}
```

Старые правила для `listItem-` и `galleryItem-` оставить тоже с `bottom: 8px`, чтобы при future-сменах классов не сломалось (CSS-специфичность позволит переопределить).

#### Шаг 2.2. `insertButtonContainer()` — точку вставки перенести в нижнюю часть карточки

Сейчас логика: найти `favoriteCartIcons-*` / `favorites-add` → вставить в `listTopBlock-*` / `body-*`.

Нужная логика (по приоритету):

1. Ищем **нижнюю зону карточки**:

   * `offerElement.querySelector('[data-marker="item-location"]')` — местоположение/доставка (там «Доставка от 1 дня»)

   * или `offerElement.querySelector('[class*="listBottomBlock-"]')` — нижний блок

   * или последний ребёнок у `offerElement.querySelector('[class*="body-"]')` (body-контейнер карточки)
2. Если нашли — берём **ближайший родительский контейнер** (обычно это `body-*` карточки), вставляем `.button-container` в **его конец**. Так как у `.button-container` position:absolute, он не влияет на flow, а DOM-вставка в нижнюю часть гарантирует, что stacking context / overflow не вырежут кнопку (кнопка «позже» в DOM, выше по z-index дефолту).
3. Гарантируем `position: relative` у offerElement (уже сделано прошлым патчем и CSS'ом).
4. Fallback: если ничего не нашли — вставить в `offerElement.appendChild` (тоже будет работать, т.к. position: absolute + bottom:0 прижмёт).

#### Шаг 2.3. Отступы — возможно `bottom: 12px` вместо 8px, чтобы кнопки не прижимались вплотную к скруглённым углам карточки

По скриншоту Avito карточки имеют скругление \~12px. Значение `8px` можно оставить как есть (минимальный gap), но проверить визуально — если вплотную, поднять до `10–12px`. В плане — реализуем `bottom: 8px; right: 8px`, как сейчас у top, но меняем только top→bottom. Пользователь может скорректировать после теста.

***

## Сводка по файлам для изменения

1. **`userCSS.css`** — 5 hover-правил: `top` → `bottom`; главное правило `.button-container` внутри `[data-marker="item"]:hover` тоже перевести на `bottom: 8px`
2. **`contentScript.js`**:

   * Функция `ensureSearchTogglesPresent()` + debounce-обёртка `scheduleEnsureToggles()`

   * Вызов `scheduleEnsureToggles()` в конце MutationObserver (ветка search-page) при любой childList-мутации

   * Новое условие в MutationObserver-ветке search-page: ловим новые `[data-marker="item"]` (и контейнеры с ними) → вызываем `processSearchPage()` и гарантируем `catalogData` fallback через `getCatalogDataAlternative()`

   * `processSearchPageNow()`: убираем жёсткий early-exit, добавляем fallback-заполнение catalogData из DOM перед возвратом

   * `insertButtonContainer()`: меняем стратегию поиска точки вставки (нижняя часть карточки: listBottomBlock / item-location / последний child body-)

   * `extractUserIdFromOfferElement()`: проверка/добавление универсальных селекторов ссылок на продавца (любой a\[href\*="/user/"] внутри карточки, href из itemprop или title-link)

***

## Риски и обработка

| Риск                                                                                       | Вероятность | Обработка                                                                                                                                               |
| ------------------------------------------------------------------------------------------ | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ensureSearchTogglesPresent()` вызывается слишком часто (лаг)                              | Средняя     | Оборачиваем в debounce 250–300мс. Плюс внутри функции быстрый getElementById-like query по data-marker                                                  |
| Debounce слишком медленный (FOUC: мелькают карточки без кнопок)                            | Средняя     | Первый вызов делаем немедленно через `setTimeout(0)` внутри scheduleEnsureToggles — если это первое событие, а последующие в течение 300мс схлопываются |
| Avito использует CSS `overflow: hidden` на внутреннем контейнере — кнопки снизу обрезаются | Низкая      | DOM-вставка как можно ближе к внешнему контейнеру offerElement (position:relative — гарантия из CSS)                                                    |
| Monkey-patch fetch ломает сторонние скрипты                                                | Высокая     | Не используем в основном плане, только как резерв                                                                                                       |
| Пустой catalogData при некоторых XHR                                                       | Средняя     | processSearchPageNow теперь пытается достать данные из DOM перед early-exit; extractUserIdFromOfferElement fallback'ом                                  |

***

## Порядок выполнения

1. ✅ **userCSS.css**: перевести все `top:8px` → `bottom:8px` во всех hover-правилах и главном `[data-marker="item"]:hover .button-container` — мгновенно даст нужный позиционирование
2. ✅ **contentScript.js insertButtonContainer()**: точка вставки перенесена в нижнюю часть карточки
3. ✅ **contentScript.js processSearchPageNow()**: убрать жёсткий `if (!catalogData) return;`, добавить fallback из getCatalogDataAlternative
4. ✅ **contentScript.js MutationObserver (search-page)**:

   * Добавить условие «появилась нода \[data-marker="item"] или контейнер с ними» → processSearchPage()

   * В конце каждой мутации на search-странице вызывать debounced `ensureSearchTogglesPresent()`
5. ✅ **contentScript.js ensureSearchTogglesPresent()**: новая функция + scheduleEnsureToggles() helper
6. ✅ **contentScript.js extractUserIdFromOfferElement()**: проверить/дополнить селекторы для извлечения sellerId из новой разметки (data-marker / href)
7. ✅ Синтаксический контроль + диагностика

