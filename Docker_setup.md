Для твоего учебного проекта лучше сделать так, чтобы node_modules вообще не попадал в WordPress-контейнер.

1. Сейчас у тебя

В docker-compose.yml есть:

wordpress:
image: wordpress:latest
restart: always
environment:
WORDPRESS_DB_HOST: mysql:3306
WORDPRESS_DB_USER: wordpress
WORDPRESS_DB_PASSWORD: wordpress
WORDPRESS_DB_NAME: gamestore
ports: - "8000:80"
volumes: - ./.srv/wordpress:/var/www/html - ./themes/:/var/www/html/wp-content/themes/ - ./plugins/:/var/www/html/wp-content/plugins/ - ./mu-plugins/:/var/www/html/wp-content/mu-plugins/ - ./.srv/custom.ini:/usr/local/etc/php/conf.d/custom.ini

Проблема именно в этой строке:

- ./plugins/:/var/www/html/wp-content/plugins/

Она говорит Docker:

«Возьми всё, что находится в C:\DockerArch\Game-Store\plugins, и покажи это WordPress-контейнеру».

А внутри blocks-gamestore находится:

node_modules/
└── 89 919 файлов 2. Что мы хотим получить

Нам нужно, чтобы Docker видел:

blocks-gamestore/
├── blocks-gamestore.php
├── build/
├── src/
├── package.json
├── package-lock.json
└── ❌ node_modules — не показывать Docker

При этом на Windows ничего удалять не надо.

То есть ты по-прежнему сможешь находиться в:

C:\DockerArch\Game-Store\plugins\blocks-gamestore

и выполнять:

npm install
npm run build

А WordPress будет работать только с нужными ему файлами.

3. Как это сделать

Docker Compose позволяет сделать дополнительный mount поверх конкретной папки.

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

Но есть один очень важный момент

Этот вариант создаёт отдельный Docker volume с именем node_modules.

То есть внутри контейнера путь:

/var/www/html/wp-content/plugins/blocks-gamestore/node_modules

будет существовать, но Docker будет показывать там содержимое отдельного volume, а не твой Windows-каталог:

C:\DockerArch\Game-Store\plugins\blocks-gamestore\node_modules

Это именно то, что нам нужно для производительности.

Но поскольку WordPress вообще не использует node_modules, нам даже не важно, что там находится.

4. Почему это безопасно

Главное:

мы ничего не удаляем с твоего компьютера.

Твой:

C:\DockerArch\Game-Store\plugins\blocks-gamestore\node_modules

останется нетронутым.

Он нужен тебе для разработки.

Docker просто перестанет использовать его как часть bind mount.

Получается разделение:

WINDOWS DOCKER / WORDPRESS

node_modules ─────────X─────> не видит

src ───────────────> видит
build ───────────────> видит
\*.php ───────────────> видит
package.json ───────────────> видит 5. После изменения

Сохраняем docker-compose.yml.

Затем из:

C:\DockerArch\Game-Store

выполняем:

docker compose up -d

Docker пересоздаст WordPress-контейнер с новым mount.

После этого сразу ничего больше не меняем.

Проверяем:

docker exec game-store-wordpress-1 bash -c 'time find /var/www/html/wp-content/plugins/blocks-gamestore/node_modules -type f | wc -l'

Мы ожидаем увидеть:

0

или небольшое количество файлов, если Docker volume будет содержать что-то созданное внутри.

А главное — команда должна выполняться практически мгновенно.

И затем повторяем наш первоначальный тест:

docker exec game-store-wordpress-1 bash -c 'time find /var/www/html -type f | wc -l'

Если всё сделано правильно, вместо прежних 26 секунд мы должны получить уже доли секунды или значительно меньшее время.

Я бы пока не менял ничего другого в Docker. Сначала сделаем только это одно изменение и посмотрим, насколько изменится скорость.

PS C:\DockerArch\Game-Store> docker compose up -d
[+] up 4/4
✔ Container game-store-phpmyadmin-1 Running 0.0s
✔ Container game-store-mysql-1 Running 0.0s
✔ Volume game-store_node_modules Created 0.0s
✔ Container game-store-wordpress-1 Started

PS C:\DockerArch\Game-Store> docker exec game-store-wordpress-1 bash -c 'time find /var/www/html/wp-content/plugins/blocks-gamestore/node_modules -type f | wc -l'
0

real 0m0.004s
user 0m0.000s
sys 0m0.005s, похоже получилось. С
👍
