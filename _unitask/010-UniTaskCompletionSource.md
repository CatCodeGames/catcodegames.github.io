---
layout: post
title: "UniTaskCompletionSource"
date: 2026-09-10
order: 10
description: "Почему UniTask стоит возвращать напрямую, а не прятать его вместе с данными."
topic: unitask
---

# UniTaskCompletionSource

## Зачем нужен

`UniTaskCompletionSource` позволяет создать `UniTask`, завершением которого можно управлять. Применяется, когда операция не возвращает `UniTask`, но её нужно дождаться через `await`.

С помощью `UniTaskCompletionSource` **можно преобразовать событие или callback в `UniTask`**, который можно ожидать через `await`.

Через свойство `.Task` можно получить связанный с источником `UniTask`. А с помощью следующих методов управлять состоянием операции:

- `TrySetResult()` — успешно завершает операцию;
- `TrySetCanceled()` — отменяет операцию;
- `TrySetException()` — завершает операцию с указанной ошибкой.

Главное отличие от `UniTaskCompletionSource` — после получения результата `AutoResetUniTaskCompletionSource` **сбрасывает своё состояние и возвращается в пул**. Поэтому он подходит для одноразовых сигналов, когда завершённое состояние не нужно сохранять для последующих ожиданий.

`UniTaskCompletionSource`, в отличие от `AutoResetUniTaskCompletionSource` сохраняет состояние завершённой операции. Поэтому его можно использовать, когда результат должен оставаться доступным для последующих ожиданий.

## Как использовать

Работа с `UniTaskCompletionSource` сводится к следующим действиям:

- создаём источник:

``` csharp
var utcs = new UniTaskCompletionSource();
```

- Через свойство `.Task` получаем связанный с ним `UniTask`, который можно ожидать через `await`:
``` csharp
await utcs.Task;
Debug.Log("Завершено");
```

- Когда операция завершена, сообщаем об этом источнику:

``` csharp
utcs.TrySetResult();
```

После этого связанный `UniTask` считается завершённым, и код после `await utcs.Task` начинает выполняться дальше. А последующие ожидания `utcs.Task` сразу получат результат.

### Пример
Рассмотрим простой сценарий: нужно загрузить ресурс один раз, а затем использовать его в разных частях программы.

```csharp
var loader = new ResourceLoader("Prefabs/Player");

var player1 = await loader.LoadAsync();

// Загрузка уже завершена, результат сразу доступен.
var player2 = await loader.LoadAsync();
```

Создадим `ResourceLoader`, который запускает загрузку ресурса при создании:

```csharp
public sealed class ResourceLoader
{
    private UniTaskCompletionSource<GameObject> _utcs;

    public UniTask<GameObject> LoadAsync(string path)
    {
        if (_utcs != null)
            return _utcs.Task;

        _utcs = new UniTaskCompletionSource<GameObject>();

        var request = Resources.LoadAsync<GameObject>(path);

        request.completed += _ =>
        {
            _utcs.TrySetResult((GameObject)request.asset);
        };

        return _utcs.Task;
    }
}
```

При первом вызове `LoadAsync()` запускается загрузка ресурса. Пока она выполняется, последующие вызовы этого метода будут ждать ту же загрузку.

Когда ресурс загрузится, `TrySetResult()` завершит `UniTaskCompletionSource` и сохранит результат. Поэтому при следующих вызовах `LoadAsync()` ресурс уже не загружается заново — `await` сразу получает готовый результат.



## Когда и где использовать

`UniTaskCompletionSource` подходит, когда нужно вручную управлять завершением операции и сохранить её результат. Это особенно удобно, если одну операцию могут ожидать несколько потребителей, в том числе те, которые начали ожидание уже после её завершения.

Например, это может быть однократная загрузка ресурса, инициализация системы или получение данных, результат которых нужно использовать в разных частях программы.

В отличие от `AutoResetUniTaskCompletionSource`, после завершения `UniTaskCompletionSource` не сбрасывает своё состояние. Поэтому последующие вызовы могут получить уже готовый результат.