# Внедрение в Hermes Agent

Проверено на Hermes 0.23.0. Нужен 0.22.5 или новее: раньше не было подключаемых провайдеров
делегирования.

## 1. Плагин

```bash
hermes plugins install advanced-delegation
hermes plugins enable advanced-delegation     # установка сама НЕ включает
hermes plugins list | grep advanced           # должен быть enabled
```

Плагин официальный (`NousResearch/hermes-official-plugins`, каталог `advanced-delegation`). Даёт два
режима вместо встроенного: `model_map` (маршруты моделей для субагентов) и `profile_routes` (субагент
как отдельный профиль Hermes). Модели, провайдеры и ключи остаются в конфиге; агенту видны только
имена маршрутов и их описания, выдумать маршрут он не может.

## 2. Конфиг маршрутов

```yaml
delegation:
  backend: model_map
  backends:
    model_map:
      default: workhorse
      options:
        - name: workhorse
          description: "Основной агент: файлы, терминал, веб, длинные сессии. По умолчанию."
          provider: <provider>
          model: <model-id>
        - name: fast
          description: "Дешёвый массовый труд: извлечение, парсинг, классификация, черновики."
          provider: <provider>
          model: <model-id>
        - name: deep
          description: "Тяжёлые многошаговые агентские цепочки. Дорого."
          provider: <provider>
          model: <model-id>
          reasoning_effort: max
```

Ключи маршрута: `name` и `description` обязательны, `model` обязателен, `provider` и/или `base_url`;
опционально `reasoning_effort`, `service_tier`, `fallback_providers`, `api_mode`, `api_key`
(лучше `provider:` – ключ останется в `.env`).

Правка: `hermes delegation config` (интерактивно) либо YAML напрямую с бэкапом:

```bash
cp config.yaml "config.yaml.bak-$(date +%Y%m%d-%H%M%S)-prerouting"
python - <<'EOF'
import io
from ruamel.yaml import YAML
p = "config.yaml"
y = YAML(typ="rt"); y.preserve_quotes = True; y.width = 4096
cfg = y.load(io.open(p, encoding="utf-8"))
cfg.setdefault("delegation", {})["backend"] = "model_map"
y.dump(cfg, io.open(p, "w", encoding="utf-8"))
EOF
```

Ссылку на шаблон всех ролей см. `../templates/routes.yaml`.

## 3. Служебные вызовы на дешёвую модель

```yaml
auxiliary:
  approval:          {provider: <provider>, model: <cheap-model>}
  title_generation:  {provider: <provider>, model: <cheap-model>}
  background_review: {provider: <provider>, model: <cheap-model>}
  curator:           {provider: <provider>, model: <cheap-model>}
  monitor:           {provider: <provider>, model: <cheap-model>}
```

Эти блоки мержатся с существующими (не заменять целиком: у `title_generation` есть свои таймауты).
Проверка: `hermes doctor` печатает строку `auxiliary task routing resolves: ...` по каждой задаче.

## 4. Резерв провайдера

```yaml
fallback_providers:
  - provider: <second-provider>
    model: <model-id>
```

Пока список пуст, любой 403 или таймаут единственного провайдера роняет ход. Резерв подхватывает
беседу без потери контекста.

## 5. Проверка, что маршруты работают

Схема инструмента фиксируется на старте сессии, поэтому проверять надо в новой сессии:

```bash
hermes delegation config --show
hermes doctor | grep -i auxiliary

hermes -z "Вызови delegate_task с двумя задачами в одном вызове.
Первая: goal='Верни ровно слово ALPHA', option='code'.
Вторая: goal='Верни ровно слово BETA' без option.
Затем ответь JSON-массивом [{\"label\":...,\"model\":...,\"summary\":...}]"
```

Ожидаемый результат: у первой задачи `label` равен запрошенному маршруту и `model` – модель этого
маршрута; у второй – маршрут `default`. Если оба ребёнка на одной модели, значит режим не применился
(не включён плагин либо сессия старая).

## 6. Откат

```bash
hermes plugins disable advanced-delegation
cp config.yaml.bak-<метка>-prerouting config.yaml
```

Встроенный режим `default` продолжает работать и без плагина: дети получают `delegation.model`, иначе
модель родителя.

## Ловушки

- `install` не включает плагин. Без `enable` режима в конфиге просто нет.
- Старый `delegation.reasoning_effort` (например `ultra`) протекает в маршрутных детей и попадает в
  отчёт `route` как есть, даже если модель такого уровня не принимает. Очистить ключ пустой строкой,
  усилие задавать в маршруте.
- Смена режима и маршрутов применяется к новым сессиям. В текущей сессии список параметров
  инструмента уже собран; `/delegation --now` пересоберёт его ценой разрыва кэша промпта.
- Маршрут без `provider` и без `base_url` отвергается: модель не найдётся у провайдера по умолчанию.
- Дублирующиеся имена маршрутов молча отбрасываются, смотреть `hermes logs`.
