# Docker_setup.md — диагностика медленного WordPress + Docker + WSL 2

## 1. Цель

Эта шпаргалка описывает практический сценарий диагностики локального WordPress-проекта под Docker Desktop + WSL 2.

Проект:

```text
C:\DockerArch\Game-Store
```

Стек:

- Docker Desktop
- WSL 2 / Ubuntu
- WordPress
- MySQL
- phpMyAdmin
- Docker Compose

### Проблема

WordPress работал, но медленно.

В результате диагностики выяснилось, что причина была не в неисправности Docker или WSL 2, а в огромном количестве файлов внутри `node_modules`, который был примонтирован в WordPress-контейнер.

---

# 2. Проверка WSL 2

В PowerShell:

```powershell
wsl --status
```

Проверить дистрибутивы:

```powershell
wsl -l -v
```

Ожидаемый результат:

```text
NAME              STATE           VERSION
Ubuntu            Running         2
docker-desktop    Running         2
```

Главное:

```text
VERSION = 2
```

Это означает, что используется WSL 2.

---

# 3. Проверка Docker

Версия клиента и сервера:

```powershell
docker version
```

Общая информация:

```powershell
docker info
```

Проверить работающие контейнеры:

```powershell
docker ps
```

Для Game-Store:

```text
game-store-wordpress-1
game-store-mysql-1
game-store-phpmyadmin-1
```

Если контейнеры работают, это ещё не означает, что внутри проекта нет проблемы с производительностью.

---

# 4. Проверка входа в WordPress-контейнер

```powershell
docker exec -it game-store-wordpress-1 bash
```

После появления:

```text
root@...:/var/www/html#
```

мы находимся внутри контейнера.

Выйти:

```bash
exit
```

Важно: появление shell после `docker exec -it ... bash` — не зависание. Это интерактивная командная строка контейнера.

---

# 5. Первый тест файловой системы

Внутри контейнера:

```bash
time ls -R /var/www/html > /dev/null
```

Результат:

```text
real    0m45.050s
```

Посчитать все файлы:

```bash
time find /var/www/html -type f | wc -l
```

Результат:

```text
93757

real    0m26.385s
```

Это показало необычно медленный рекурсивный обход дерева.

Но пока неизвестно, где именно находится проблема.

---

# 6. Проверка отдельных каталогов

Проверяем:

```powershell
docker exec game-store-wordpress-1 bash -c "time ls -la /var/www/html"
```

```powershell
docker exec game-store-wordpress-1 bash -c "time ls -la /var/www/html/wp-content"
```

```powershell
docker exec game-store-wordpress-1 bash -c "time ls -la /var/www/html/wp-content/plugins"
```

Обычный `ls` оказался быстрым.

Следовательно, проблема была не в простом доступе к каталогам, а в глубоком рекурсивном обходе большого дерева.

---

# 7. Найти каталог с большим количеством файлов

Проверяем plugins:

```powershell
docker exec game-store-wordpress-1 bash -c "time find /var/www/html/wp-content/plugins -type f | wc -l"
```

Результат:

```text
89988

real    0m27.337s
```

Для сравнения:

```powershell
docker exec game-store-wordpress-1 bash -c "time find /var/www/html/wp-content/themes -type f | wc -l"
```

```text
419

real    0m0.150s
```

```powershell
docker exec game-store-wordpress-1 bash -c "time find /var/www/html/wp-content/uploads -type f | wc -l"
```

```text
7

real    0m0.014s
```

```powershell
docker exec game-store-wordpress-1 bash -c "time find /var/www/html/wp-admin -type f | wc -l"
```

```text
593

real    0m0.055s
```

```powershell
docker exec game-store-wordpress-1 bash -c "time find /var/www/html/wp-includes -type f | wc -l"
```

```text
2729

real    0m0.946s
```

Вывод:

**почти все файлы находятся в `wp-content/plugins`.**

---

# 8. Найти проблемный плагин

Проверяем размеры:

```powershell
docker exec game-store-wordpress-1 bash -c 'du -sh /var/www/html/wp-content/plugins/*'
```

Результат:

```text
784M    /var/www/html/wp-content/plugins/blocks-gamestore
0       /var/www/html/wp-content/plugins/core-gamestore
0       /var/www/html/wp-content/plugins/index.php
```

Проблемный плагин:

```text
blocks-gamestore
```

---

# 9. Найти причину внутри плагина

```powershell
docker exec game-store-wordpress-1 bash -c 'du -sh /var/www/html/wp-content/plugins/blocks-gamestore/*'
```

Результат:

```text
4.0K    blocks-gamestore.php
160K    build
783M    node_modules
776K    package-lock.json
4.0K    package.json
4.0K    readme.txt
48K     src
```

Главный виновник:

```text
node_modules = 783 MB
```

---

# 10. Подтверждение экспериментом

Количество файлов в `node_modules`:

```powershell
docker exec game-store-wordpress-1 bash -c 'time find /var/www/html/wp-content/plugins/blocks-gamestore/node_modules -type f | wc -l'
```

Результат:

```text
89919

real    0m25.360s
```

Для сравнения:

```powershell
docker exec game-store-wordpress-1 bash -c 'time find /var/www/html/wp-content/plugins/blocks-gamestore/build -type f | wc -l'
```

Результат:

```text
40

real    0m0.017s
```

Итого:

```text
node_modules    89919 файлов    ~25 секунд
build           40 файлов       ~0.017 секунды
```

Это подтвердило источник проблемы.

---

# 11. Почему `node_modules` не нужен WordPress

`node_modules` нужен для разработки Gutenberg-плагина.

Например:

```bash
npm install
npm run build
```

После сборки готовые файлы находятся в:

```text
build/
```

Типичная структура:

```text
blocks-gamestore/
├── blocks-gamestore.php
├── build/
├── src/
├── package.json
├── package-lock.json
└── node_modules/
```

Для выполнения WordPress:

```text
build/               нужен
blocks-gamestore.php нужен

src/                 не нужен для выполнения
package.json         не нужен для выполнения
package-lock.json    не нужен для выполнения
node_modules/        не нужен для выполнения
```

Поэтому `node_modules` можно оставить на Windows для разработки, но не показывать его WordPress-контейнеру.

---

# 12. Почему `.dockerignore` здесь не помогает

В Compose используется bind mount:

```yaml
- ./plugins/:/var/www/html/wp-content/plugins/
```

Это означает:

```text
Windows plugins/
        ↓
Docker bind mount
        ↓
/var/www/html/wp-content/plugins/
```

`.dockerignore` применяется к Docker build context.

Он **не исключает файлы из bind mount**.

Поэтому здесь нужен отдельный Docker volume.

---

# 13. Решение: отдельный Docker volume

В `docker-compose.yml` у сервиса `wordpress` добавить:

```yaml
wordpress:
  image: wordpress:latest
  restart: always
  environment:
    WORDPRESS_DB_HOST: mysql:3306
    WORDPRESS_DB_USER: wordpress
    WORDPRESS_DB_PASSWORD: wordpress
    WORDPRESS_DB_NAME: gamestore
  ports:
    - "8000:80"
  volumes:
    - ./.srv/wordpress:/var/www/html
    - ./themes/:/var/www/html/wp-content/themes/
    - ./plugins/:/var/www/html/wp-content/plugins/
    - ./mu-plugins/:/var/www/html/wp-content/mu-plugins/
    - ./.srv/custom.ini:/usr/local/etc/php/conf.d/custom.ini

    # Скрываем Windows node_modules от WordPress
    - node_modules:/var/www/html/wp-content/plugins/blocks-gamestore/node_modules
```

В нижней части Compose добавить:

```yaml
volumes:
  wordpress-core:
  node_modules:
```

---

# 14. Что делает этот volume

Windows:

```text
C:\DockerArch\Game-Store\plugins\blocks-gamestore\node_modules
```

остаётся на месте.

Но внутри WordPress-контейнера этот путь перекрывается отдельным Docker volume:

```text
/var/www/html/wp-content/plugins/blocks-gamestore/node_modules
```

Получается:

```text
WINDOWS                         DOCKER / WORDPRESS

node_modules  ─────────X─────>  не видит Windows node_modules

src           ───────────────>  видит
build         ───────────────>  видит
*.php         ───────────────>  видит
package.json  ───────────────>  видит
```

Ничего с Windows удалять не требуется.

---

# 15. Применить изменение

Из каталога проекта:

```powershell
cd C:\DockerArch\Game-Store
```

Запустить:

```powershell
docker compose up -d
```

В нашем случае Docker создал:

```text
game-store_node_modules
```

и перезапустил WordPress-контейнер.

---

# 16. Проверить результат

```powershell
docker exec game-store-wordpress-1 bash -c 'time find /var/www/html/wp-content/plugins/blocks-gamestore/node_modules -type f | wc -l'
```

До исправления:

```text
89919

real    0m25.360s
```

После:

```text
0

real    0m0.004s
```

То есть:

```text
25.360 сек → 0.004 сек
```

Это практически полное устранение проблемного файлового обхода.

---

# 17. Повторить исходный тест

Теперь полезно снова проверить всё дерево:

```powershell
docker exec game-store-wordpress-1 bash -c 'time find /var/www/html -type f | wc -l'
```

Сравниваем результат с исходным:

```text
До:
93757 файлов
real ≈ 26 секунд
```

После:

```text
значительно меньше файлов доступно для обхода
и время должно существенно уменьшиться
```

---

# 18. Полезные команды Docker

### Работающие контейнеры

```powershell
docker ps
```

### Все контейнеры

```powershell
docker ps -a
```

### Использование ресурсов

```powershell
docker stats --no-stream
```

### Docker volumes

```powershell
docker volume ls
```

### Информация о контейнере

```powershell
docker inspect game-store-wordpress-1
```

### Только mounts

```powershell
docker inspect game-store-wordpress-1 --format "{{json .Mounts}}"
```

### Размер каталогов внутри контейнера

```powershell
docker exec game-store-wordpress-1 bash -c 'du -sh /var/www/html/wp-content/*'
```

### Количество файлов

```powershell
docker exec game-store-wordpress-1 bash -c 'find /var/www/html -type f | wc -l'
```

### Измерение времени команды

```powershell
docker exec game-store-wordpress-1 bash -c 'time <команда>'
```

---

# 19. Общий алгоритм диагностики медленного Docker WordPress

Если WordPress под Docker работает медленно:

1. Проверить WSL 2.
2. Проверить Docker Desktop.
3. Проверить контейнеры через `docker ps`.
4. Проверить файловую систему через `time`.
5. Посчитать количество файлов через `find`.
6. Определить каталог с большинством файлов.
7. Найти конкретный источник:

   - `node_modules`
   - cache
   - `.git`
   - временные файлы
   - другие большие деревья

8. Не удалять файлы вслепую.
9. Для development-only каталогов рассмотреть отдельный Docker volume.
10. Повторить тот же тест после изменения.

Главный принцип:

> **Сначала измеряем → находим узкое место → меняем одну вещь → измеряем снова.**

Не стоит сразу:

- увеличивать память Docker;
- менять настройки MySQL;
- переносить весь проект в WSL;
- переустанавливать Docker;
- отключать случайные плагины.

Сначала нужно найти фактическое узкое место.

---

# 20. Итог для Game-Store

Исходная проблема:

```text
WordPress + Docker работал медленно
```

Найдено:

```text
wp-content/plugins/
└── blocks-gamestore/
    └── node_modules/
        ├── ~89 919 файлов
        └── ~783 MB
```

Причина:

```text
node_modules был частью Windows bind mount,
который целиком отображался в WordPress-контейнере.
```

Решение:

```yaml
- node_modules:/var/www/html/wp-content/plugins/blocks-gamestore/node_modules
```

и:

```yaml
volumes:
  wordpress-core:
  node_modules:
```

Контрольный тест:

```text
До:
89919 файлов
25.360 секунд

После:
0 файлов
0.004 секунды
```

### Главный вывод

Docker Desktop и WSL 2 в данном случае **не требовали перенастройки**.

Проблема была в структуре файлов проекта и в том, что огромный `node_modules` оказался внутри bind mount WordPress-контейнера.

Правильная диагностика позволила устранить проблему точечным изменением Docker Compose, не удаляя `node_modules` из проекта и не меняя остальную конфигурацию.
