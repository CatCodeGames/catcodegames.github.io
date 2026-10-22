# catcodegames.github.io

## Локальный запуск

Нужен Docker. Все файлы для запуска лежат в папке `.docker`.

```bash
docker compose -f .docker/compose.yaml up
```

Сайт откроется на http://localhost:4000 и будет пересобираться при сохранении файлов.

После изменения `.docker/Gemfile` образ нужно пересобрать:

```bash
docker compose -f .docker/compose.yaml up --build
```
