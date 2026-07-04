---
tags: [шаблон, геймдев, TDD, технический-документ, архитектура, ООП, тестирование, логирование]
date_created: {{date}}
type: project
---

# Technical Design Document: {{Название игры}}

**Команда:** {{Имя 1}}, {{Имя 2}}  
**Дата:** {{ГГГГ-ММ-ДД}}  
**Версия:** 0.1  
**Статус:** 🔴 В разработке / 🟡 На проверке / 🟢 Утверждён

> TDD пишется **до написания кода**. Если в процессе разработки архитектура меняется — обновляй документ, а не игнорируй расхождение. Каждое существенное изменение фиксируй строкой в [[#История изменений]].

---

## 1. Структура проекта

Плоская структура (маленький проект / прототип):

```
project/
├── main.py            # точка входа, игровой цикл
├── settings.py        # константы (размер экрана, FPS, цвета, баланс)
├── states/
│   ├── menu.py        # экран главного меню
│   ├── game.py        # игровое состояние
│   └── game_over.py   # экран завершения
├── entities/
│   ├── player.py
│   ├── enemy.py
│   └── projectile.py  # (если есть)
├── systems/
│   ├── spawner.py      # таймерный спавнер объектов
│   ├── parallax.py     # параллакс-фон (если есть)
│   └── hud.py          # отрисовка HUD
├── utils/
│   └── {{helpers.py}} # вспомогательные функции
├── assets/
│   ├── images/
│   ├── sounds/
│   └── fonts/
├── tests/
│   └── {{test_player.py}}  # unit-тесты на ключевую логику
└── docs/              # Concept Doc, GDD, TDD (этот файл)
```

**src-layout** *(опционально — для проектов с uv/pip-пакетом)*:

```
project/
├── main.py                 # тонкая точка входа: вызывает app.main()
├── pyproject.toml          # uv: python версия, зависимости, pytest, ruff
├── uv.lock                 # зафиксированные версии зависимостей
├── save.json               # рекорды и настройки (создаётся при первом запуске)
├── logs/{{game.log}}       # роллинг-лог событий (создаётся при первом запуске)
├── src/{{package_name}}/
│   ├── __init__.py
│   ├── app.py               # State Machine, главный игровой цикл
│   ├── settings.py
│   ├── levels/               # см. раздел 3 — слой уровней (если игра многоуровневая)
│   ├── entities/
│   ├── systems/
│   ├── states/
│   └── utils/
│       ├── save.py
│       └── logger.py         # см. раздел 5.3 — логирование
├── assets/
└── tests/
```

Выбирай плоскую структуру для небольших игр/учебных прототипов; src-layout — когда проект оформляется как устанавливаемый пакет (uv, публикация, полноценный CI). Запуск: `uv run main.py`, тесты — `uv run pytest`.

---

## 2. Диаграмма классов (UML)

### 2.1 Иерархия наследования

```
pygame.sprite.Sprite
└── GameObject                  # базовый класс: image, rect, update()
    ├── Player                  # управление, HP, прыжок
    ├── Enemy                   # ИИ, патрулирование
    │   ├── {{EnemyTypeA}}
    │   └── {{EnemyTypeB}}
    └── Projectile              # движение, коллизия, урон
```

### 2.2 Описание классов

#### `GameObject(pygame.sprite.Sprite)`
Базовый класс для всех игровых объектов.

| Атрибут / Метод | Тип | Описание |
|-----------------|-----|----------|
| `image` | `Surface` | Спрайт объекта |
| `rect` | `Rect` | Позиция и размер |
| `update(dt)` | `None` | Обновление состояния на каждом кадре |

---

#### `Player(GameObject)`

| Атрибут / Метод | Тип | Описание |
|-----------------|-----|----------|
| `hp` | `int` | Текущее здоровье |
| `speed` | `float` | Скорость движения (px/с) |
| `velocity` | `Vector2` | Вектор скорости |
| `is_alive` | `bool` | Жив ли игрок |
| `handle_input()` | `None` | Чтение клавиатуры/мыши |
| `move(dt)` | `None` | Перемещение с учётом dt |
| `take_damage(amount)` | `None` | Уменьшение HP, проверка смерти |
| `{{jump()}}` | `None` | {{Применение вертикального импульса}} |

---

#### `Enemy(GameObject)`

| Атрибут / Метод | Тип | Описание |
|-----------------|-----|----------|
| `hp` | `int` | Здоровье |
| `speed` | `float` | Скорость |
| `damage` | `int` | Урон при контакте |
| `update(dt)` | `None` | Обновление ИИ и позиции |
| `{{patrol()}}` | `None` | {{Патрулирование между точками}} |

---

#### `{{Другие классы}}`

*Добавь по аналогии.*

> **Заметка об архитектурном решении** *(добавляй такой блок для каждого неочевидного решения)*:
> `{{имя механизма}}` — {{в чём нестандартность (напр. мутабельная ссылка вместо копии, общее состояние, порядок инициализации)}}. Причина: {{почему обычный/наивный подход не подошёл}}.

---

### 2.3 Основные группы спрайтов

```python
all_sprites   = pygame.sprite.Group()   # все объекты — для update и draw
enemies       = pygame.sprite.Group()   # враги — для коллизий с игроком
projectiles   = pygame.sprite.Group()   # снаряды — для коллизий с врагами
pickups       = pygame.sprite.Group()   # бонусы
```

### 2.4 Управление состояниями (State Machine)

```python
# settings.py
class GameState:
    MENU     = "menu"
    PLAYING  = "playing"
    PAUSED   = "paused"
    GAME_OVER = "game_over"
```

```
MENU ──[Играть]──► PLAYING ──[Смерть]──► GAME_OVER ──[Снова]──► PLAYING
                      │                                └──[Меню]──► MENU
                   [Esc]
                      ▼
                   PAUSED ──[Esc / Продолжить]──► PLAYING
                      └──[Меню]──► MENU
```

---

## 3. Слой уровней / миров *(опционально — если в игре несколько уровней с разным балансом)*

Для игр с прогрессией удобно выносить всё, что зависит от уровня (пул спавна, скорость, фон), в отдельные классы вместо `if/elif` в спавнере.

### 3.1 Абстрактный базовый класс

```python
# levels/base.py
from abc import ABC, abstractmethod

class BaseLevel(ABC):
    name: str = ""
    unlock_score: int = 0          # минимальный рекорд для разблокировки

    @abstractmethod
    def parallax_layers(self) -> list[dict]: ...

    @abstractmethod
    def spawn_pool(self, elapsed: float) -> dict[str, int]: ...
    # → {"{{obstacle_a}}": 10, "{{obstacle_b}}": 8, ...} — веса, меняются со временем

    @abstractmethod
    def speed_at(self, elapsed: float) -> float: ...

    def music_track(self) -> str:
        return "{{common/music_game.ogg}}"
```

### 3.2 Конкретные уровни

| Класс | Файл | `unlock_score` | Старт скорости | Особенности |
|-------|------|---------------|---------------|-------------|
| `{{LevelOne}}` | `{{level_one.py}}` | 0 | {{250 px/с}} | {{…}} |
| `{{LevelTwo}}` | `{{level_two.py}}` | {{2000}} | {{300 px/с}} | {{…}} |

Добавить новый уровень = один новый файл в `levels/` + регистрация в `level_select.py`. Остальной код (спавнер, параллакс, игровой цикл) не трогается.

---

## 4. [[Игровой цикл|Игровой цикл]] и архитектура `main.py`

```python
# Псевдокод main.py
init pygame
state = GameState.MENU

while running:
    dt = clock.tick(FPS) / 1000

    events = pygame.event.get()
    for event in events:
        handle_quit(event)

    if state == GameState.MENU:
        state = menu.update(events)
        menu.draw(screen)

    elif state == GameState.PLAYING:
        state = game.update(dt, events)
        game.draw(screen)

    elif state == GameState.PAUSED:
        state = pause.update(events)
        pause.draw(screen)

    elif state == GameState.GAME_OVER:
        state = game_over.update(events)
        game_over.draw(screen)

    pygame.display.flip()
```

Каждый стейт реализует `update(dt, events) → str | None` и `draw() → None`; возвращает строку нового состояния или `None`, если переход не нужен.

---

## 5. Формат сохранений, конфигурации и логирования

### 5.1 `settings.py` — константы

```python
# Экран
WIDTH, HEIGHT = 800, 600
FPS = 60
TITLE = "{{Название игры}}"

# Цвета
BLACK  = (0, 0, 0)
WHITE  = (255, 255, 255)
{{COLOR_NAME}} = {{(R, G, B)}}

# Баланс
PLAYER_HP    = 3
PLAYER_SPEED = 200       # px/с
GRAVITY      = 800       # px/с² (если есть)
JUMP_FORCE   = -400      # px/с (если есть)
ENEMY_SPEED  = 100
```

### 5.2 `save.json` — рекорды и прогресс

```json
{
  "high_score": 0,
  "total_games": 0,
  "settings": {
    "music_volume": 0.7,
    "sfx_volume": 1.0,
    "fullscreen": false
  }
}
```

**Чтение / запись:**

```python
import json, os

SAVE_PATH = "save.json"

def load_save():
    if not os.path.exists(SAVE_PATH):
        return {"high_score": 0, "total_games": 0}
    with open(SAVE_PATH) as f:
        return json.load(f)

def write_save(data):
    with open(SAVE_PATH, "w") as f:
        json.dump(data, f, indent=2)
```

### 5.3 Логирование *(опционально, но рекомендуется для проектов с src-layout)*

Роллинг-файл через `logging.handlers.RotatingFileHandler` — ротация по размеру, с хранением N старых файлов. Директория логов создаётся автоматически при первом вызове.

```python
# utils/logger.py
import logging, os
from logging.handlers import RotatingFileHandler
from {{package_name}} import settings

_LOGGER_NAME = "{{package_name}}"

def get_logger() -> logging.Logger:
    """Настраивает роллинг-логгер при первом вызове, дальше переиспользует."""
    logger = logging.getLogger(_LOGGER_NAME)
    if any(isinstance(h, RotatingFileHandler) for h in logger.handlers):
        return logger

    os.makedirs(settings.LOG_DIR, exist_ok=True)
    handler = RotatingFileHandler(
        os.path.join(settings.LOG_DIR, settings.LOG_FILE),
        maxBytes=settings.LOG_MAX_BYTES,
        backupCount=settings.LOG_BACKUP_COUNT,
        encoding="utf-8",
    )
    handler.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s"))
    logger.addHandler(handler)
    logger.setLevel(getattr(logging, settings.LOG_LEVEL))
    logger.propagate = False
    return logger
```

Что логировать:

| Событие | Уровень | Место |
|---------|---------|-------|
| Запуск / закрытие игры | `INFO` | `main.py` / `app.py` |
| Переход между стейтами | `INFO` | `app.py` |
| Game Over (уровень, счёт) | `INFO` | `states/game.py` |
| Повреждённый или отсутствующий `save.json` | `WARNING` | `utils/save.py` |
| {{…}} | {{…}} | {{…}} |

---

## 6. Используемые алгоритмы

### 6.1 Обнаружение коллизий

| Пара | Метод | Описание |
|------|-------|----------|
| Игрок ↔ Враги | `spritecollide` | AABB по `rect` |
| Игрок ↔ Тайлы платформ | `colliderect` + проверка по оси Y | Односторонние платформы |
| Снаряды ↔ Враги | `groupcollide` | Удаляем и снаряд, и врага |
| Игрок ↔ Бонусы | `spritecollide` | Подбор предмета |

### 6.2 Простой ИИ врагов

**Патрулирование:**
```python
# Враг движется между двумя точками
if self.rect.x >= self.right_bound:
    self.direction = -1
elif self.rect.x <= self.left_bound:
    self.direction = 1
self.rect.x += self.speed * self.direction * dt
```

**Преследование игрока:**
```python
dx = player.rect.centerx - self.rect.centerx
if abs(dx) > 5:
    self.rect.x += self.speed * (1 if dx > 0 else -1) * dt
```

### 6.3 Гравитация и прыжок *(если есть)*

```python
# В Player.update(dt):
self.velocity.y += GRAVITY * dt          # ускорение вниз
self.rect.y    += self.velocity.y * dt   # применяем скорость

# Приземление: сброс velocity.y при коллизии с платформой снизу
```

### 6.4 {{Другой алгоритм}}

*Например: [[Алгоритм A*]] для поиска пути, процедурная генерация уровня, взвешенный случайный выбор из пула спавна, расчёт баллистики.*

{{Описание + псевдокод или ссылка на источник}}

---

## 7. Тестирование *(рекомендуется для src-layout проектов)*

| Файл теста | Что покрывает | Кол-во тестов |
|------------|---------------|---------------|
| `tests/{{test_player.py}}` | {{ключевая логика игрока: урон, прыжок, подборы}} | {{…}} |
| `tests/{{test_spawner.py}}` | {{интервалы спавна, пул, сброс}} | {{…}} |

Приоритет — покрывать не отрисовку, а логику без побочных эффектов: физику, урон, подсчёт очков, выбор из пула, переходы состояний.

---

## 8. Сетевой протокол *(только для сетевых игр)*

### 8.1 Архитектура

```
[Клиент 1] ──► [Сервер] ◄── [Клиент 2]
               (Python socket / websocket)
```

- **Протокол:** {{TCP / UDP}}
- **Формат:** JSON
- **Порт:** {{5555}}

### 8.2 Типы пакетов

```json
// Клиент → Сервер: действие игрока
{
  "type": "player_action",
  "player_id": 1,
  "action": "move",
  "data": { "x": 320, "y": 240, "vx": 200, "vy": 0 },
  "timestamp": 1234567890
}

// Сервер → Клиенты: снимок состояния мира
{
  "type": "game_state",
  "tick": 42,
  "players": [
    { "id": 1, "x": 320, "y": 240, "hp": 3 },
    { "id": 2, "x": 500, "y": 200, "hp": 2 }
  ],
  "enemies": [
    { "id": 10, "x": 400, "y": 300, "hp": 20 }
  ]
}
```

### 8.3 Обработка рассинхронизации

- **Авторитетный сервер:** только сервер считает физику и коллизии; клиент отображает
- **Интерполяция:** клиент сглаживает позиции между двумя последними снимками сервера
- **Таймаут:** если пакет не получен за {{500 мс}} — игрок считается отключённым

---

## История изменений

| Версия | Дата | Изменение |
|--------|------|-----------|
| 0.1 | {{ГГГГ-ММ-ДД}} | Первый черновик |
| {{…}} | {{…}} | {{напр.: рефакторинг на src-layout, добавлен слой levels/, добавлено логирование}} |

---

*Предыдущий шаг: [[Шаблон — GDD]]*  
*Связанные темы: [[Документация игрового проекта]], [[Игровой цикл]]*
