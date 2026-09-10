# Клавиатура в Telegram Mini App на iOS: причина рывков и рабочее решение

Демо, на котором всё проверено на iPhone:

- Бот: https://t.me/testswapklavabot (кнопка «Открыть» внизу слева)
- Страница: https://slobodianandrey3-arch.github.io/tg-keyboard-demo/
- Код: `index.html` в этом репозитории, один файл, без сборки и без бэкенда

## TL;DR

1. Высоту корневого контейнера брать из CSS-переменной `--tg-viewport-height`, которую выставляет `telegram-web-app.js`. Не из `window.innerHeight`, не из `100vh`, не из `visualViewport.height`, не из `viewportStableHeight`.
2. Фон под WebView и фон страницы сделать одного цвета: `html { background }` плюс `Telegram.WebApp.setBackgroundColor()`, `setHeaderColor()`, `setBottomBarColor()`.
3. На всех кнопках рядом с полем ввода (чипы, скрепка, эмодзи, отправка) вешать `pointerdown` с `preventDefault()`, чтобы тап не уводил фокус из поля.
4. Высоту не анимировать. Ничего не пересчитывать в JS по событиям viewport. Нижний блок держать в потоке flex-колонки, без `position: fixed` и без JS-отступов.

## Симптомы в проде

С видео из прод-бота:

- При закрытии клавиатуры контент остаётся сжатым, внизу примерно полсекунды висит чёрная полоса, потом контент рывком раскладывается.
- При открытии страница сжимается раньше, чем клавиатура доезжает. Между полем ввода и клавиатурой мелькает чёрный зазор.
- Иногда клавиатура «доезжает криво» после тапа по кнопкам над ней.

## Замеры на iOS

В демо вверху выводится строка с четырьмя высотами. Вот что приходит и когда (iPhone, экран 800 css px, с клавиатурой 455):

| Момент | `innerHeight` (рамка WebView) | `visualViewport.height` | `Telegram.WebApp.viewportHeight` |
|---|---|---|---|
| Клавиатура начала подниматься | 800 (старое) | 800 (старое) | **455 (новое)** |
| Клавиатура поднялась, +0.25 с | 800 (старое) | 455 | 455 |
| Ещё +0.25 с | 455 | 455 | 455 |
| Клавиатура начала уезжать | 800 | 455 (залипло) | **800 (новое)** |
| Клавиатура ушла | 800 | 455 (всё ещё залипло) | 800 |

Выводы:

- `viewportHeight` от Telegram приходит раньше всех и сразу с конечным значением, в обе стороны. Одно событие `viewportChanged` на переход, `isStateStable` сразу `true`.
- Рамку WebView Telegram меняет с опозданием около 0.5 с на открытии и сразу на закрытии.
- `visualViewport` на открытии отстаёт, а на закрытии вообще не обновляется, пока не случится что-то ещё.

## Причина прод-симптомов

Симптомы один в один совпадают с вёрсткой, где высота нижнего блока или всего экрана считается от `visualViewport.height` или `innerHeight`, а фон `html` не задан.

- Открытие: `visualViewport` сжимается раньше рамки, контейнер уезжает вверх, между полем и ещё не доехавшей клавиатурой видно фон WebView. Он чёрный, потому что `setBackgroundColor` не вызван и `html` без фона.
- Закрытие: `visualViewport` залипает на 455, контейнер остаётся сжатым, рамка уже 800, снизу 345 px чёрного фона WebView.
- Кривой доезд после кнопок: тап по кнопке над клавиатурой забирает фокус у поля, клавиатура начинает уезжать, код возвращает фокус, клавиатура едет обратно.

Что искать в прод-коде: `visualViewport`, `innerHeight`, `100vh`, `100dvh`, `viewportStableHeight`, `--tg-viewport-stable-height`, любые `resize`-хендлеры, которые выставляют `height` или `bottom` в px.

## Решение

### Каркас

```html
<div id="app">            <!-- фиксирован, высота от Telegram -->
  <div id="log">...</div>  <!-- flex:1, скроллится только он -->
  <div id="bottom">        <!-- лента кнопок + строка ввода, в потоке -->
    <div id="chips">...</div>
    <form id="bar">...</form>
  </div>
</div>
```

```css
html { height: 100%; background: #141416; }
body { margin: 0; height: 100%; background: #141416; overflow: hidden; overscroll-behavior: none; }

#app {
  position: fixed; top: 0; left: 0; right: 0;
  height: var(--tg-viewport-height, 100%);   /* ключевая строка */
  display: flex; flex-direction: column;
  background: #141416;
}
#log { flex: 1; min-height: 0; overflow-y: auto; -webkit-overflow-scrolling: touch; overscroll-behavior: contain; }
#bottom { flex: none; padding-bottom: env(safe-area-inset-bottom); }
#chips { display: flex; gap: 8px; overflow-x: auto; scrollbar-width: none; }
#inp { font-size: 16px; }   /* меньше 16px — iOS зумит страницу при фокусе */
```

### Инициализация

```js
const tg = window.Telegram?.WebApp;
const BG = '#141416';                 // тот же цвет, что в CSS
tg.ready();
tg.expand();
tg.setBackgroundColor(BG);            // фон под WebView
tg.setHeaderColor(BG);
tg.setBottomBarColor?.(BG);
tg.disableVerticalSwipes?.();
```

### Кнопки над клавиатурой

```js
function keepFocus(el) {
  el.addEventListener('pointerdown', e => e.preventDefault());
}
document.querySelectorAll('.chip, #attach, #emoji, #send').forEach(keepFocus);
```

`preventDefault` на `pointerdown` отменяет перенос фокуса, но не отменяет `click` и не ломает скролл ленты пальцем. Клавиатура остаётся открытой, если была открыта, и не открывается, если была закрыта.

### Страховка от скролла документа

```js
window.addEventListener('scroll', () => { if (window.scrollY) window.scrollTo(0, 0); }, { passive: true });
```

iOS иногда прокручивает сам документ, чтобы показать поле ввода. С фиксированным корнем это лишнее, возвращаем на место.

### Для React и подобных

Не надо подписываться на `viewportChanged` и класть высоту в state. CSS-переменная обновляется самим SDK, вёрстка перестраивается без ререндера. Ререндер на каждое событие только добавит задержку.

## Что пробовали и что не сработало

- **Высота от рамки WebView** (`position: fixed; inset: 0`, без переменной). Закрытие чистое, но на открытии поле ввода 0.5 с сидит под клавиатурой, потом один кадр с отскоком контента и только потом всё встаёт. Причина: рамка на открытии меняется с опозданием.
- **Анимация высоты** (`transition: height 250ms`). Стало хуже. Сигнал от Telegram приходит, когда клавиатура уже прошла треть пути. Анимация стартует оттуда и заканчивается после клавиатуры. На закрытии поле ещё едет вниз, когда клавиатуры уже нет, и висит посреди экрана.

## Чего сделать нельзя

Покадровая синхронизация с клавиатурой, как в нативном чате Telegram, из WebView недостижима. Нативное приложение получает от iOS длительность и кривую анимации до её старта и двигает композер тем же аниматором. В WebView эти данные не пробрасываются, есть только событие с конечной высотой, и оно приходит с небольшим опозданием. Лучший доступный результат: поле ввода мгновенно занимает конечную позицию в начале анимации клавиатуры, без чёрных полос, без рывков, без отскоков.

Обычные сайты (Apple, Amazon) эту проблему обходят: поле ввода лежит в потоке страницы, Safari сам прокручивает его в видимую область, клавиатура накрывает низ. Веб-чаты с нижним композером (Telegram Web, WhatsApp Web) в мобильном Safari имеют тот же прыжок.

## Чек-лист внедрения в прод

- [ ] Корневой контейнер: `position: fixed; top: 0; height: var(--tg-viewport-height, 100%)`.
- [ ] Убрать все расчёты высоты от `visualViewport`, `innerHeight`, `100vh`, `100dvh`, `viewportStableHeight`.
- [ ] `html` и `body` с фоном цвета приложения, `body { overflow: hidden }`, скроллится только список сообщений.
- [ ] `setBackgroundColor`, `setHeaderColor`, `setBottomBarColor` тем же цветом.
- [ ] Нижний блок (чипы + строка ввода) в потоке flex-колонки, без `position: fixed` и без отступов из JS.
- [ ] `pointerdown` + `preventDefault` на всех кнопках рядом с полем ввода.
- [ ] `font-size: 16px` у поля ввода.
- [ ] Никаких `transition` на высоту контейнера.
- [ ] Проверить на iPhone: открыть и закрыть клавиатуру, тапнуть чипы при открытой и при закрытой клавиатуре, проскроллить ленту чипов при открытой клавиатуре.

## Ссылки

- Открытые issue в Telegram-iOS по этой теме, без ответа разработчиков: [#1298](https://github.com/TelegramMessenger/Telegram-iOS/issues/1298), [#1410](https://github.com/TelegramMessenger/Telegram-iOS/issues/1410), [#1637](https://github.com/TelegramMessenger/Telegram-iOS/issues/1637), [#1296](https://github.com/TelegramMessenger/Telegram-iOS/issues/1296)
- Как `telegram-web-app.js` выставляет `--tg-viewport-height`: https://telegram.org/js/telegram-web-app.js (функция `setViewportHeight`)
- Почему `position: fixed` и клавиатура в Safari не дружат: https://www.bram.us/2021/09/13/prevent-items-from-being-hidden-underneath-the-virtual-keyboard-by-means-of-the-virtualkeyboard-api/
