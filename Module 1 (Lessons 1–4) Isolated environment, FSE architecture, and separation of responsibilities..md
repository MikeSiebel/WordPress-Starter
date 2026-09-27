# Подробный конспект Модуля 1 (Уроки 1–4)

**Курс:** Modern WordPress & WooCommerce Development: Full Site Editing Mastery [cite: 55, 60]  
**Автор курса:** Александр Сакирка («Быть Программистом») [cite: 57, 60]  
**Стек модуля:** WordPress FSE, Docker Compose (WSL 2), PHP 8.x, Git, Node.js (`@wordpress/create-block`), Figma [cite: 29, 58, 61, 62].

---

## 00:00:00 | Урок 1: Интро

## 00:01:32 | Необходимый софт для курса и обзор дизайн-макета

### 1. Практическая реализация (Practice)

- **Подготовка локального стека разработчика:**
  - **Docker Desktop**: Установка актуальной версии программы [cite: 61, 62]. Для операционных систем Windows в обязательном порядке активируется интеграция с подсистемой **WSL 2** (Windows Subsystem for Linux 2) [cite: 62].
  - **Visual Studio Code**: Настройка редактора кода с AI-ассистентами и встроенным терминалом [cite: 63].
  - **Git и GitHub**: Установка распределенной системы контроля версий Git и создание аккаунта на GitHub [cite: 63, 64].
  - **Figma**: Установка десктопного приложения Figma для работы с векторной графикой и макетами [cite: 64].
- **Анализ дизайн-макета GameStore в Figma:**
  - Исследование структуры e-commerce магазина видеоигр [cite: 66]: главная страница, каталог товаров с фильтрацией, сингл открытой игры с вариациями (Standard, Deluxe, Complete Edition), новостной раздел, корзина, checkout и личный кабинет [cite: 66, 67, 68, 69, 70].
  - Макет спроектирован в двух цветовых вариациях — светлой (**Light Mode**) и контрастной тёмной (**Dark Mode**) [cite: 66].
- **Входные требования к навыкам:** Уверенное владение версткой (HTML5/CSS3), понимание синтаксиса PHP и JavaScript [cite: 64, 65].

### 2. Теоретическое обоснование (Theory)

- **Зачем нужен модуль WSL 2 на Windows:** Файловые системы Windows (NTFS) и Linux (ext4) обрабатывают операции ввода-вывода (I/O) по-разному [cite: 62]. При запускe Docker без WSL 2 обращения к тысячам мелких файлов ядра WordPress происходят через слой трансляции, что вызывает критические задержки [cite: 62]. WSL 2 разворачивает полноценное ядро Linux внутри Windows, ускоряя интерпретацию файлов в разы [cite: 62].
- **Концептуальный сдвиг FSE vs Classic WordPress:**
  - _Классическая разработка_: Использование PHP-шаблонов (`header.php`, `footer.php`, `page.php`, `sidebar.php`), вывод контента через шорткоды и хаки переопределения WooCommerce (`Storefront`) [cite: 29, 65].
  - _Full Site Editing (FSE)_: Переход к компонентной архитектуре. Интерфейс строится из **блоков Gutenberg (React/JSX)**, структура страниц хранится в чисто HTML-шаблонах с Гутенберг-комментариями (`/templates/*.html`), а глобальные стили, палитра и типографика задаются декларативно через файл `theme.json` [cite: 15, 29, 65].

---

## 00:13:38 | Урок 2: Запускаем WordPress под Docker

### 1. Практическая реализация (Practice)

- **Организация структуры папок проекта:**
  В корневой директории проекта `GameStore` создается служебная папка `.srv/` для хранения изолированных данных базы и настроек сервера [cite: 73, 80]:

  ```text
  GameStore/
  └── .srv/
      ├── database/     <-- Точка монтирования данных MySQL
      └── custom.ini    <-- Кастомная конфигурация PHP
  ```

- **Создание файла `docker-compose.yaml` в корне проекта:**
  Файл конфигурации сгорает от ошибок синтаксиса, поэтому используется строго **2 пробела** для каждого уровня вложенности YAML [cite: 73, 74].

  ```yaml
  version: "3.9"

  services:
    mysql:
      image: mysql:latest
      restart: always
      ports:
        - "3310:3306"
      environment:
        MYSQL_ROOT_PASSWORD: wordpress
        MYSQL_DATABASE: gamestore
        MYSQL_USER: wordpress
        MYSQL_PASSWORD: wordpress
      volumes:
        - ./.srv/database:/var/lib/mysql

    wordpress:
      image: wordpress:latest
      restart: always
      ports:
        - "8000:80"
      environment:
        WORDPRESS_DB_HOST: mysql:3306
        WORDPRESS_DB_USER: wordpress
        WORDPRESS_DB_PASSWORD: wordpress
        WORDPRESS_DB_NAME: gamestore
      volumes:
        - ./.srv/wordpress:/var/www/html
        - ./.srv/custom.ini:/usr/local/etc/php/conf.d/custom.ini
      depends_on:
        - mysql

    phpmyadmin:
      image: phpmyadmin/phpmyadmin:latest
      restart: always
      ports:
        - "8181:80"
      environment:
        PMA_HOST: mysql
        MYSQL_USERNAME: wordpress
        MYSQL_ROOT_PASSWORD: wordpress
  ```

  _(Примечание: В `docker-compose.yaml` внутренний порт контейнера `mysql:3306` связывается с веб-сервисом WordPress через параметр `WORDPRESS_DB_HOST: mysql:3306`)_ [cite: 75, 77, 85].

- **Настройка файла `.srv/custom.ini` для предотвращения ошибок 500:**
  Для увеличения лимитов загрузки медиафайлов и выделяемой памяти создается кастомный файл [cite: 80]:

  ```ini
  file_uploads = On
  memory_limit = 1024M
  upload_max_filesize = 128M
  post_max_size = 128M
  max_execution_time = 1000
  max_input_time = 2000
  ```

- **Терминальные команды управления окружением:**

  ```bash
  # Запуск всех контейнеров в фоновом режиме (Detached mode)
  docker-compose up -d

  # Остановка работающих контейнеров без удаления данных
  docker-compose down

  # Полная очистка контейнеров и удаление сетевых томов (при сбросе)
  docker-compose down -v
  ```

- **Проверка работоспособности:**
  - Сайт WordPress: `http://localhost:8000` [cite: 78, 83]
  - СУБД phpMyAdmin: `http://localhost:8181` [cite: 78, 85]

### 2. Теоретическое обоснование (Theory)

- **Зачем нужны Docker Volumes (`volumes`):** Контейнеры по своей природе эемерны (не сохраняют состояние при перезапуске) [cite: 76]. Если остановить контейнер без монтирования внешних папок, все таблицы базы данных MySQL и загруженные файлы будут навсегда уничтожены [cite: 76, 85]. Проброс пути `./.srv/database:/var/lib/mysql` связывает внутреннюю папку базы данных Linux-контейнера с диском хост-машины, сохраняя все данные между перезапусками [cite: 75, 76, 85].
- **Зачем подключать `custom.ini`:** Стандартные сборки Docker PHP имеют жесткие ограничения (память 128MB, лимит загрузки файлов 2MB) [cite: 80]. При установке плагина WooCommerce или загрузке тяжелых медиафайлов сервер выбрасывает Fatal Error (Exhausted Memory) или HTTP 500 [cite: 81]. Проброс файлового тома в `/usr/local/etc/php/conf.d/custom.ini` переопределяет настройки PHP ядра без необходимости пересобирать сам Docker-образ [cite: 79, 81].

---

## 00:35:17 | Урок 3: Создаем стартовую FSE тему для проекта

### 1. Практическая реализация (Practice)

- **Генерация FSE темы:** Использование генератора шаблонов [fullsiteediting.com/block-theme-generator](https://fullsiteediting.com/block-theme-generator/) для создания базовой темы `GameStore` [cite: 1, 13, 90, 91].
- **Распаковка темы:** Сгенерированные файлы темы помещаются в локальную папку `./themes/gamestore/` [cite: 91].
- **Маппинг локальных каталогов в `docker-compose.yaml`:**
  Чтобы разработка велась в корне локального репозитория Git, а не внутри служебных папок `.srv/wordpress`, обновляется секция `volumes` контейнера `wordpress` [cite: 86, 89, 90]:

  ```yaml
  volumes:
    - ./.srv/wordpress:/var/www/html
    - ./themes/gamestore:/var/www/html/wp-content/themes/gamestore
    - ./plugins:/var/www/html/wp-content/plugins
    - ./mu-plugins:/var/www/html/wp-content/mu-plugins
    - ./.srv/custom.ini:/usr/local/etc/php/conf.d/custom.ini
  ```

- **Перезапуск контейнера для применения путей:**

  ```bash
  docker-compose up -d
  ```

- **Активация темы:** Переход в админ-панель WordPress (`http://localhost:8000/wp-admin`) -> Внешний вид (Appearance) -> Темы (Themes) -> Активация темы **GameStore** [cite: 92, 93].

- **Инициализация контроля версий Git и создание `.gitignore`:**
  В корне проекта инициализируется репозиторий [cite: 96]:

  ```bash
  git init
  ```

  В корне проекта создается файл `.gitignore` для исключения тяжелого мусора и бинарников базы [cite: 34, 96, 97]:

  ```gitignore
  .srv/database
  .srv/wordpress
  node_modules/
  .DS_Store

  # Исключаем дефолтные темы WordPress, оставляет только нашу тему
  themes/*
  !themes/gamestore/
  ```

### 2. Теоретическое обоснование (Theory)

- **Принцип Volume Mapping для WP-Content:** Маппинг пробрасывает мост между локальными папками (`./themes/gamestore`, `./plugins`, `./mu-plugins`) и внутренним каталогом контейнера `/var/www/html/wp-content/` [cite: 89, 90]. Разработчик пишет код в привычном окружении VS Code в корне своего репозитория, а виртуальная машина Docker мгновенно подхватывает и исполняет эти файлы [cite: 29, 90].
- **Чистота Git-репозитория:** База данных (`.srv/database`) и файлы ядра WordPress (`.srv/wordpress`) весят сотни мегабайт и меняются локально на каждом ПК [cite: 34, 95]. В Git должны попадать **исключительно исходный код темы, кастомных плагинов и конфигурационные файлы** (`docker-compose.yaml`, `custom.ini`) [cite: 34, 98, 99].

---

## 00:54:09 | Урок 4: Создаем MU / Core / Blocks плагины для проекта

### 1. Практическая реализация (Practice)

- **1. Must-Use (MU) плагин (`mu-plugins/gamestore-general.php`):**
  Создается непосредственно в папке `mu-plugins/` как единственный PHP-файл [cite: 87, 100, 101]:

  ```php
  <?php
  /*
  Plugin Name: GameStore General MU Plugin
  Description: Обязательный системный функционал сайта (нельзя отключить из админки).
  Version: 1.0.0
  Author: Senior Dev
  */

  // Отключение дефолтных виджетов консоли WordPress
  add_action('wp_dashboard_setup', 'gamestore_remove_dashboard_widgets');
  function gamestore_remove_dashboard_widgets() {
      global $wp_meta_boxes;
      unset($wp_meta_boxes['dashboard']['normal']['core']['dashboard_activity']);
      unset($wp_meta_boxes['dashboard']['normal']['core']['dashboard_primary']);
      unset($wp_meta_boxes['dashboard']['side']['core']['dashboard_quick_press']);
  }
  ```

  _Проверка:_ В панели `wp-admin` -> Плагины появляется отдельная вкладка **Must-Use (Обязательные)**, где этот плагин отображается без кнопки «Отключить» [cite: 102].

- **2. Core-плагин бизнес-логики (`plugins/core-gamestore/core-gamestore.php`):**
  Создается в отдельной подпапке `plugins/core-gamestore/` [cite: 106]:

  ```php
  <?php
  /*
  Plugin Name: Core GameStore
  Description: Основная бизнес-логика (CPT, хуки, фильтры WooCommerce).
  Version: 1.0.0
  Text Domain: core-gamestore
  */

  define('GAMESTORE_PLUGIN_URL', plugin_dir_url(__FILE__));
  define('GAMESTORE_PLUGIN_DIR', plugin_dir_path(__FILE__));
  ```

  _Проверка:_ Активируется вручную через админ-панель WordPress в разделе «Плагины» [cite: 108].

- **3. Плагин кастомных блоков Gutenberg (`plugins/blocks-gamestore`):**
  Инициализируется через официальную CLI-утилиту WordPress `@wordpress/create-block` [cite: 1, 3, 110, 111]:

  ```bash
  # 1. Переходим в папку плагинов
  cd plugins

  # 2. Генерируем блочный плагин
  npx @wordpress/create-block@latest blocks-gamestore

  # 3. Переходим внутрь созданного плагина
  cd blocks-gamestore

  # 4. Запускаем компилятор JSX/React компонентов
  npm start
  ```

- **Реструктуризация под мульти-блочную архитектуру (`plugins/blocks-gamestore/src/`):**
  По умолчанию CLI создает только один блок [cite: 112, 113]. Исходники переносятся в подпапки `src/block-hero/`, `src/block-header/`, а главный PHP-файл плагина обновляется для динамической регистрации нескольких блоков [cite: 113, 114, 115, 116]:

  ```php
  // plugins/blocks-gamestore/blocks-gamestore.php
  function gamestore_register_blocks() {
      register_block_type(__DIR__ . '/build/block-header');
      register_block_type(__DIR__ . '/build/block-hero');
  }
  add_action('init', 'gamestore_register_blocks');
  ```

- **Навигация по директориям и решение ошибки `ENOENT: package.json` при работе на двух ПК:**
  При переходе на другой ПК с репозитория Git (где папка `node_modules` отсутствует из-за `.gitignore`) попытка выполнить `npm start` из корня проекта `GameStore` приводит к ошибке `ENOENT` [cite: 34, 40, 48].

  _Правильный алгоритм запуска сборщика React:_

  ```bash
  # Из корня проекта переходим СТРОГО в папку плагина с блоками:
  cd plugins/blocks-gamestore

  # Устанавливаем отсутствующие npm-зависимости (@wordpress/scripts):
  npm install

  # Запускаем компилятор React-компонентов:
  npm start
  ```

### 2. Теоретическое обоснование (Theory)

- **Архитектурный принцип Separation of Concerns (Разделение ответственности):**
  - **Тема (`themes/gamestore`)**: Отвечает **исключительно за визуальное представление** (CSS/SCSS, раскладка блоков, стилизация) [cite: 87, 108, 109].
  - **Плагины (`plugins/` и `mu-plugins/`)**: Отвечают за **функционал и бизнес-логику** (Custom Post Types, AJAX-обработчики, кастомные React-блоки Gutenberg) [cite: 87, 108, 109].
  - _Почему это важно:_ В классической разработке регистрация CPT и хуков часто помещалась в `functions.php` темы [cite: 108]. При смене темы пользователь моментально «терял» свой контент и функционал [cite: 108, 109]. В FSE-подходе функционал строго отделен, поэтому смена темы не затрагивает данные сайта [cite: 109].
- **Зачем нужны Must-Use (MU) плагины:**
  - Файлы из `mu-plugins/` подгружаются ядром автоматически в алфавитном порядке **раньше всех обычных плагинов и темы** [cite: 87, 88, 100].
  - Их невозможно случайно отключить в админ-панели WordPress [cite: 88, 100, 102]. Это идеальное место для критической системной логики (безопасность, регистрация CPT, отключение служебных виджетов) [cite: 100, 102].
- **Принцип работы компилятора `@wordpress/scripts`:**
  - Сборщик на базе Webpack/Babel берет исходный JSX/React-код и модульные CSS-файлы из папки `src/` и компилирует их в оптимизированные бандлы в папку `build/` [cite: 113, 114, 116].
  - Запуск `npm start` отслеживает изменения в реальном времени (Watch mode) [cite: 5, 114]. Терминал должен находиться строго в директории `plugins/blocks-gamestore/`, где расположен `package.json` с конфигурацией сборки [cite: 40, 48].
