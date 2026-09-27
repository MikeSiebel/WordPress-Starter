Разработка статических блоков Gutenberg на базе React/JSX — один из ключевых элементов современного WordPress-проекта с архитектурой Full Site Editing (FSE) [cite: 29].

Давайте подробно разберем концепцию, внутреннюю структуру, жизненный цикл и практические примеры создания статических блоков (на примере блоков **Header** и **Hero** из Модуля 1) [cite: 29, 418, 419].

---

### 1. Концепция: Статический блок (Static) vs Динамический (SSR)

В WordPress Gutenberg существует два фундаментальных подхода к созданию блоков [cite: 29, 30]:

- **Статический блок (Static Block):**
  - `edit.js` генерирует UI-интерфейс и органы управления в админ-панели (редакторе) [cite: 29, 53].
  - `save.js` генерирует чистый статичный HTML-код, который при сохранении записи записывается прямо в базу данных (в поле `post_content`) [cite: 29, 53].
  - При просмотре страницы на фронтенде WordPress просто отдаёт готовый сохранённый HTML из базы без вызова серверного PHP [cite: 29, 53].
- **Динамический блок (Dynamic / SSR Block):**
  - `save.js` возвращает `null`, а итоговый HTML рендерится на сервере при каждом запросе через PHP-функцию `render_callback` (применяется для вывода каталогов товаров WooCommerce, постов или новостей) [cite: 30, 53].

---

### 2. Структура статического блока и роль `block.json`

Инициализация плагина блоков выполняется утилитой `@wordpress/create-block` [cite: 1, 4, 53]. Все исходные JSX/React-файлы хранятся в папке `src/`, а компилятор `@wordpress/scripts` собирает их в готовую папку `build/` [cite: 4, 40, 48].

Главный конфигурационный файл каждого блока — **`block.json`** [cite: 4, 53]. Он определяет имя, атрибуты и подключение скриптов:

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "blocks-gamestore/block-hero",
  "version": "1.0.0",
  "title": "Hero Section",
  "category": "gamestore",
  "icon": "superhero",
  "attributes": {
    "title": { "type": "string", "source": "html", "selector": ".hero-title" },
    "description": {
      "type": "string",
      "source": "html",
      "selector": ".hero-description"
    },
    "buttonUrl": { "type": "string", "default": "" },
    "buttonText": { "type": "string", "default": "Buy Game" },
    "videoUrl": { "type": "string", "default": "" },
    "isVideo": { "type": "boolean", "default": true },
    "slides": { "type": "array", "default": [] }
  },
  "editorScript": "file:./index.js",
  "editorStyle": "file:./index.css",
  "style": "file:./style-index.css",
  "viewScript": "file:./view.js"
}
```

#### Ключевые параметры `block.json`:

- **`attributes`**: Объявляет переменные состояния блока.
- **`source` и `selector`**: Указывают Gutenberg брать значение атрибута напрямую из сохраненного HTML-тега (например, из тега с классом `.hero-title`), чтобы не дублировать данные в мета-комментариях записи [cite: 29].
- **`viewScript`**: Подключает JS-скрипт (`view.js`), который будет исполняться исключительно на фронтенде и только тогда, когда данный блок есть на странице [cite: 29].

---

### 3. Разработка интерфейса редактора (`edit.js`)

Файл `edit.js` отвечает за отображение блока в админ-панели и предоставляет элементы управления [cite: 29, 53].

В нем используются компоненты пакетов `@wordpress/block-editor` и `@wordpress/components` [cite: 29]:

- **`<useBlockProps />`**: Поставляет системные классы и обертки Гутенберга.
- **`<RichText />`**: Позволяет редактировать текст в режиме WYSIWYG прямо в холсте редактора [cite: 29].
- **`<InspectorControls />`**: Помещает настройки и текстовые поля в правую боковую панель сайдбара [cite: 29].
- **`<MediaUpload />`**: Предоставляет встроенную модалку загрузки файлов из медиабиблиотеки WordPress [cite: 29].
- **`<InnerBlocks />`**: Позволяет вкладывать нативные блоки WordPress (например, `wp:site-logo` или `wp:navigation`) внутри нашего кастомного блока [cite: 29].

#### Пример реализации `edit.js` (промо-блок Hero) [cite: 29, 419]:

```jsx
import {
  useBlockProps,
  RichText,
  InspectorControls,
  MediaUpload,
} from "@wordpress/block-editor";
import {
  PanelBody,
  TextControl,
  ToggleControl,
  Button,
} from "@wordpress/components";

export default function Edit({ attributes, setAttributes }) {
  const { title, description, buttonUrl, buttonText, videoUrl, isVideo } =
    attributes;
  const blockProps = useBlockProps();

  return (
    <>
      {/* Настройки в сайдбаре редактора */}
      <InspectorControls>
        <PanelBody title="Медиа и кнопки">
          <ToggleControl
            label="Использовать видео фон"
            checked={isVideo}
            onChange={(val) => setAttributes({ isVideo: val })}
          />
          <MediaUpload
            onSelect={(media) => setAttributes({ videoUrl: media.url })}
            type="video"
            render={({ open }) => (
              <Button onClick={open} variant="secondary">
                Загрузить видео
              </Button>
            )}
          />
          <TextControl
            label="Текст кнопки"
            value={buttonText}
            onChange={(val) => setAttributes({ buttonText: val })}
          />
          <TextControl
            label="Ссылка кнопки"
            value={buttonUrl}
            onChange={(val) => setAttributes({ buttonUrl: val })}
          />
        </PanelBody>
      </InspectorControls>

      {/* Визуальная верстка в самом холсте */}
      <div {...blockProps}>
        <RichText
          tagName="h1"
          className="hero-title"
          value={title}
          onChange={(val) => setAttributes({ title: val })}
          placeholder="Введите заголовок..."
        />
        <RichText
          tagName="p"
          className="hero-description"
          value={description}
          onChange={(val) => setAttributes({ description: val })}
          placeholder="Введите описание..."
        />
      </div>
    </>
  );
}
```

---

### 4. Сериализация и сохранение (`save.js`)

Файл `save.js` возвращает структуру чистого HTML, которая сохраняется в базу данных [cite: 29, 53].

#### Важные правила `save.js`:

1. Функция `save` должна быть **чистой** (Pure Function). В ней нельзя использовать хуки состояния React (`useState`, `useEffect`) или вызывать асинхронные запросы [cite: 29].
2. Вместо инпут-компонента `<RichText />` в `save.js` используется его статический аналог `<RichText.Content />` [cite: 29].
3. Итоговый HTML-код должен строго соответствовать структуре, описанной в `edit.js` [cite: 29].

#### Пример реализации `save.js` [cite: 29, 419]:

```jsx
import { useBlockProps, RichText } from "@wordpress/block-editor";

export default function save({ attributes }) {
  const { title, description, buttonUrl, buttonText, videoUrl, isVideo } =
    attributes;
  const blockProps = useBlockProps.save();

  return (
    <section {...blockProps}>
      {isVideo && videoUrl && (
        <video className="video-background" autoPlay loop muted playsInline>
          <source src={videoUrl} type="video/mp4" />
        </video>
      )}
      <div className="hero-content">
        <RichText.Content tagName="h1" className="hero-title" value={title} />
        <RichText.Content
          tagName="p"
          className="hero-description"
          value={description}
        />
        <a href={buttonUrl} className="hero-button">
          {buttonText}
        </a>
      </div>
    </section>
  );
}
```

---

### 5. Интерактивность на фронтенде (`view.js`)

Если блоку требуется JS-логика на пользовательской части сайта (например, инициализация слайдера Swiper.js, открытие модалок или аккордеонов), она выносится в файл `view.js` [cite: 29]:

```javascript
import Swiper from "swiper";

document.addEventListener("DOMContentLoaded", () => {
  const sliderContainers = document.querySelectorAll(".hero-slider-container");

  sliderContainers.forEach((container) => {
    new Swiper(container, {
      loop: true,
      autoplay: { delay: 2000, disableOnInteraction: false },
      slidesPerView: "auto",
      grabCursor: true,
    });
  });
});
```

---

### 6. Ошибка валидации блоков (Block Validation & Recovery)

Одна из самых частых проблем при работе со статическими блоками — предупреждение редактора: **«Block contains unexpected or invalid content»** (Ошибки валидации блока) [cite: 29].

- **Причина:** Gutenberg сравнивает HTML-код, записанный в базе данных (`post_content`), с HTML-кодом, который генерирует текущая функция `save.js` [cite: 29]. Если вы изменили теги или классы в `save.js` после того, как блок уже был сохранен на странице, Gutenberg синтаксически зафиксирует расхождение [cite: 29].
- **Решение:**
  1. В редакторе нажать кнопку **«Attempt Block Recovery»** (Восстановить блок) [cite: 29].
  2. В разработке — очистить сохраненную страницу или пересохранить её с новым кодом `save.js` [cite: 29].

---

### 7. Пошаговый процесс разработки и запуска компилятора

При создании и тестировании статических блоков придерживайтесь следующего порядка команд [cite: 40, 48, 423, 426]:

1. В терминале перейдите строго в директорию плагина с блоками [cite: 40, 48, 428, 429]:
   ```bash
   cd plugins/blocks-gamestore
   ```
2. Убедитесь, что установлены `node_modules` (особенно если перешли на другой ПК) [cite: 48, 423, 426]:
   ```bash
   npm install
   ```
3. Запустите компилятор в режиме наблюдения (Watch Mode) [cite: 5, 40, 423, 426]:
   ```bash
   npm start
   ```

Хотите подробнее разбрать работу с `<InnerBlocks />` для вложенной навигации, репитеры для списков или стилизацию через CSS-переменные?
