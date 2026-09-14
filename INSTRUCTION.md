# INSTRUCTION

Запуск Django-Todolist разом із базою MySQL через Docker Compose. Обидва сервіси описані в `docker-compose.yml`: `db` — MySQL з постійним томом, `app` — Django-застосунок.

## Вимоги

- Docker з плагіном Compose — перевірити командою `docker compose version`
- Вільний порт 8080 на хості

## Запуск

З кореня репозиторію:

```bash
docker compose up --build
```

Перший запуск триває кілька хвилин: завантажуються базові образи, встановлюються залежності.

Що відбувається по черзі:

1. Збирається образ бази з `Dockerfile.mysql`, створюється том `mysql_data`
2. MySQL ініціалізує базу `app_db` і користувача `app_user`
3. Compose чекає, поки healthcheck бази почне відповідати — застосунок не стартує раніше
4. Збирається образ застосунку, виконуються міграції, піднімається сервер

Готовність видно за рядком у логах:


Запуск у фоні, щоб звільнити термінал:

```bash
docker compose up --build -d
```

## Доступ до застосунку

| Адреса | Призначення |
|---|---|
| http://localhost:8080/ | головна сторінка |
| http://localhost:8080/api/ | браузерний інтерфейс REST API |
| http://localhost:8080/admin/ | панель адміністратора Django |

Створити користувача для входу:

```bash
docker compose exec app python manage.py createsuperuser
```

## Перевірка стану

```bash
docker compose ps
docker compose logs -f app
docker compose logs -f db
```

У `docker compose ps` сервіс `db` має бути `healthy`, `app` — `running`.

## Зупинка

Зупинити, зберігши контейнери:

```bash
docker compose stop
```

Продовжити потім — `docker compose start`.

Зупинити й видалити контейнери та мережу:

```bash
docker compose down
```

Том `mysql_data` лишається, усі завдання збережені.

Видалити все разом з даними:

```bash
docker compose down -v
```

Прапорець `-v` видаляє том — дані зникають безповоротно.

## Перевірка збереження даних

1. Відкрий http://localhost:8080/ і створи кілька завдань
2. `docker compose down`
3. `docker compose up -d`
4. Онови сторінку — завдання на місці

Подивитися записи прямо в базі:

```bash
docker compose exec db mysql -uapp_user -p1234 app_db -e "SELECT id, description FROM lists_todo;"
```

## Як влаштована конфігурація

**Том.** `mysql_data` змонтовано в `/var/lib/mysql` — каталог даних MySQL. Завдяки цьому база переживає видалення контейнерів.

**Порядок запуску.** У сервіса `db` є healthcheck, а `app` залежить від нього через `condition: service_healthy`. Без цього застосунок стартував би раніше за базу і міграції падали б з `Can't connect to MySQL server`.

**Міграції.** Перенесені зі збірки в `ENTRYPOINT`: на етапі `docker build` бази ще не існує, тому `RUN python manage.py migrate` з Dockerfile прибрано.

**Адреса бази.** У `settings.py` хост читається зі змінної `DB_HOST`, яку Compose передає як `db` — ім'я сервіса. Всередині мережі Compose воно резолвиться в IP контейнера автоматично.

## Якщо щось не запускається

- **Порт 8080 зайнятий** — зміни ліву частину в `docker-compose.yml` на `"8081:8080"`
- **База не стає `healthy`** — `docker compose logs db`. Якщо том лишився від запуску з іншим паролем: `docker compose down -v` і заново
- **Зміни в коді не видно** — образ старий, перезібрати: `docker compose up --build`