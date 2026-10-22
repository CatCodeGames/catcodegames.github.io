---
layout: post
title: "UniTaskCompletionSource: как превратить событие в UniTask"
short_title: "UniTaskCompletionSource"
slug: unitask-completion-source
last_modified_at: 2026-10-23
redirect_from: /unity/unitask/010-UniTaskCompletionSource/
date: 2026-09-10
order: 10
description: "Как вручную завершить UniTask и сохранить результат для последующих ожиданий."
topic: unitask
---

# UniTaskCompletionSource

Не всё в Unity возвращает `UniTask`. Многие операции сообщают о завершении через событие или колбэк. Чтобы дождаться такой операции через `await`, её нужно превратить в `UniTask`.

Для этого есть `UniTaskCompletionSource`. Он создаёт `UniTask`, который завершится тогда, когда вы скажете.

> Это первая из трёх заметок о том, как превращать события и колбэки в `UniTask`. Здесь — самый простой и универсальный способ.
{: .aside}


## Как это работает

Источник `UniTaskCompletionSource` выдаёт `UniTask` через свойство `.Task`. Этот `UniTask` не завершится сам: его завершают вручную одним из трёх методов:

- `TrySetResult()` — операция завершилась успешно;
- `TrySetCanceled()` — операция отменена;
- `TrySetException()` — операция завершилась с ошибкой.

Главная особенность `UniTaskCompletionSource`: **после завершения он запоминает результат**. Все, кто начнёт ждать его позже, сразу получат готовое значение.


## Как использовать

Создаём источник, ждём его `UniTask` и в нужный момент завершаем:

```csharp
var tcs = new UniTaskCompletionSource();

// Где-то ждём завершения
await tcs.Task;
Debug.Log("Завершено");

// Где-то в другом месте сообщаем, что операция закончилась
tcs.TrySetResult();
```

Когда вызывается `TrySetResult()`, код после `await tcs.Task` продолжает выполнение. Если кто-то вызовет `await tcs.Task` позже, он не будет ждать: результат уже есть.

Если операция должна вернуть значение, используется `UniTaskCompletionSource<T>`. Значение передаётся в `TrySetResult()`, а `await` его возвращает:

```csharp
var tcs = new UniTaskCompletionSource<int>();

tcs.TrySetResult(42);

int value = await tcs.Task; // 42
```

> Почему методы начинаются с `Try`? Завершить источник можно только один раз. Случайный или не случайный повторный вызов ничего не сломает: метод просто вернёт `false`.
{: .aside}


## Пример: загрузить ресурс один раз

Простой сценарий: ресурс нужно загрузить один раз, а потом использовать в разных частях игры.

Вот как это может выглядеть для вызывающего кода:

```csharp
var loader = new ResourceLoader("Prefabs/Player");

var player1 = await loader.LoadAsync();

// Загрузка уже завершена, результат доступен сразу.
var player2 = await loader.LoadAsync();
```

`Resources.LoadAsync` сообщает о завершении через событие `completed`, объекта `ResourceRequest`. Превратим эту операцию в `UniTask` с помощью `UniTaskCompletionSource`:

```csharp
public sealed class ResourceLoader
{
    private readonly string _path;
    private UniTaskCompletionSource<GameObject> _tcs;

    public ResourceLoader(string path)
    {
        _path = path;
    }

    public UniTask<GameObject> LoadAsync()
    {
        // Загрузка уже запущена или завершена: отдаём ту же задачу.
        if (_tcs != null)
            return _tcs.Task;

        _tcs = new UniTaskCompletionSource<GameObject>();

        var request = Resources.LoadAsync<GameObject>(_path);
        request.completed += _ =>
        {
            var asset = request.asset as GameObject;

            if (asset != null)
                _tcs.TrySetResult(asset);
            else
                _tcs.TrySetException(new Exception($"Ресурс не найден: {_path}"));
        };

        return _tcs.Task;
    }
}
```

При первом вызове `LoadAsync()` запускается загрузка. Пока она идёт, повторные вызовы получают тот же `UniTask` и ждут ту же загрузку.

Когда ресурс загрузится, `TrySetResult()` завершит источник и сохранит результат. Дальше `LoadAsync()` уже ничего не загружает: `await` сразу получает готовый объект.

Если ресурс не найден, `TrySetException()` завершит источник с ошибкой. Тогда `await` выбросит это исключение, и его можно поймать обычным `try/catch`.

> Ошибку источник тоже запоминает. Если загрузка однажды не удалась, все следующие вызовы `LoadAsync()` получат ту же ошибку. Для ресурса, которого нет в проекте, это правильно. А вот для запроса к серверу, скорее всего, понадобится возможность попробовать ещё раз.
{: .aside}


## Когда использовать

`UniTaskCompletionSource` подходит, когда **результат операции нужно сохранить**. Например, если одну операцию ждут несколько частей программы, и некоторые из них приходят уже после её завершения.

Типичные случаи:

- однократная загрузка ресурса;
- инициализация системы, которую ждут другие системы;
- получение данных, которые потом используются в разных местах.

Если же нужно дождаться разового события, например нажатия кнопки, и результат хранить не нужно, удобнее [AutoResetUniTaskCompletionSource]({% link _unitask/011-AutoResetUniTaskCompletionSource.md %}). Он не создаёт новый объект на каждое ожидание, а берёт его из пула.

> По сути, `UniTaskCompletionSource` — это обещание: «результат будет, просто не сейчас». А кто пришёл за ним позже, получит его сразу.
{: .aside}
