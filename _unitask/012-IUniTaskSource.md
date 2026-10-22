---
layout: post
title: "IUniTaskSource: свой источник UniTask без аллокаций"
short_title: "IUniTaskSource"
slug: iunitasksource
last_modified_at: 2026-10-23
redirect_from: /unity/unitask/012-IUniTaskSource/
date: 2026-10-05
order: 12
description: "Как написать свой источник UniTask без лишних аллокаций."
topic: unitask
---

# IUniTaskSource

Чтобы дождаться события или колбэка через `await`, их превращают в `UniTask`. Обычно для этого хватает готовых источников: [UniTaskCompletionSource]({% link _unitask/010-UniTaskCompletionSource.md %}) и [AutoResetUniTaskCompletionSource]({% link _unitask/011-AutoResetUniTaskCompletionSource.md %}). Создали источник, подписались на событие, в обработчике вызвали `TrySetResult()`.

Но полностью избавиться от аллокаций они не помогают. На каждое ожидание всё равно создаются делегаты для подписки и объекты для захваченных переменных. Готовый `OnInvokeAsync()` из `UniTask` тоже создаёт новый объект на каждый вызов.

Если ожидание создаётся редко, это не важно. Но если очень часто, например каждый кадр, мелкие аллокации складываются в заметную нагрузку на сборщик мусора. Тогда можно написать собственный источник, который реализует `IUniTaskSource`, и взять всю логику ожидания под свой контроль.

> Это третья заметка о том, как превращать события и колбэки в `UniTask`, и самая сложная. Для большинства задач первых двух способов вполне хватает. Свой источник нужен, когда каждая аллокация на счету.
{: .aside}

Для примера сделаем ожидание `UnityEvent`. В итоге вызывающий код будет выглядеть так:

```csharp
await button.onClick.WaitAsync(cancellationToken);
```

А внутри объект будет проходить такой цикл:

```
WaitAsync → взять объект из пула → подписаться на событие и отмену
          → событие или отмена → завершить ожидание
          → await забрал результат → отписаться → вернуть объект в пул
```

Дальше соберём этот цикл по шагам.

> Кода будет много, но пишется такой класс один раз. Дальше его можно копировать как шаблон для других событий.
{: .aside}


## IUniTaskSource и UniTaskCompletionSourceCore

`IUniTaskSource` требует реализовать четыре метода:

```csharp
public interface IUniTaskSource
{
    UniTaskStatus GetStatus(short token);
    void OnCompleted(Action<object> continuation, object state, short token);
    void GetResult(short token);
    UniTaskStatus UnsafeGetStatus();
}
```

Подробно разбирать их не будем. Через них `UniTask` узнаёт состояние операции, регистрирует продолжение (код после `await`) и забирает результат.

Писать эту логику самому не нужно. В `UniTask` уже есть `UniTaskCompletionSourceCore<T>` — структура, которая хранит состояние операции и умеет всё необходимое. Готовые источники из `UniTask` устроены так же.

Создадим класс `UnityEventPromise`, добавим в него `UniTaskCompletionSourceCore` и передадим ему вызовы методов интерфейса:

```csharp
public sealed class UnityEventPromise : IUniTaskSource
{
    private UniTaskCompletionSourceCore<AsyncUnit> _core;

    public void GetResult(short token)
        => _core.GetResult(token);

    public void OnCompleted(Action<object> continuation, object state, short token)
        => _core.OnCompleted(continuation, state, token);

    public UniTaskStatus GetStatus(short token)
        => _core.GetStatus(token);

    public UniTaskStatus UnsafeGetStatus()
        => _core.UnsafeGetStatus();
}
```

`UniTaskCompletionSourceCore<T>` требует указать тип результата. `UnityEvent` ничего не возвращает, поэтому используем `AsyncUnit` — специальный тип для операций без результата.

Важная деталь: `_core` — изменяемая структура, поэтому **поле нельзя помечать `readonly`**. Иначе каждый вызов будет работать с копией, и состояние не сохранится. Компилятор об этом не предупредит.

Источник уже умеет хранить состояние и работать с `UniTask`, но пока его ничто не завершает.


## Ожидание UnityEvent

Чтобы завершить ожидание, источник должен знать, какое событие он ждёт. Передадим ему `UnityEvent`.

Сразу учтём главное: новый объект на каждое ожидание создаваться не будет. Цель — минимум аллокаций, поэтому **`UnityEventPromise` будет переиспользоваться через пул**.

Из-за этого событие нельзя передать через конструктор. Объект из пула нужно подготовить к работе, а перед возвратом в пул — очистить. Для этого добавим методы `Initialize()` и `Release()`:

```csharp
private UnityEvent _unityEvent;

private void Initialize(UnityEvent unityEvent)
{
    _unityEvent = unityEvent;
}

private void Release()
{
    _unityEvent = null;
}
```

Источник должен реагировать на событие. Добавим обработчик и подпишемся на `UnityEvent`:

```csharp
private void Initialize(UnityEvent unityEvent)
{
    _unityEvent = unityEvent;
    _unityEvent.AddListener(OnInvoked);
}

private void OnInvoked()
{
    _unityEvent.RemoveListener(OnInvoked);
    _core.TrySetResult(AsyncUnit.Default);
}
```

Когда событие сработает, обработчик отпишется от него и вызовет `TrySetResult()`. Ожидающий `UniTask` завершится, и код после `await` продолжит выполнение.


## Отмена ожидания

Теперь добавим отмену. Передадим в `Initialize()` токен отмены и зарегистрируем обработчик, который сработает при отмене:

```csharp
private CancellationToken _cancellationToken;
private CancellationTokenRegistration _cancellationTokenRegistration;

private void Initialize(UnityEvent unityEvent, CancellationToken cancellationToken)
{
    // ...
    _cancellationToken = cancellationToken;
    _cancellationTokenRegistration = _cancellationToken.Register(OnCanceled);
}
```

`Register()` возвращает `CancellationTokenRegistration`. Сохраняем его в поле, чтобы потом отписаться от отмены через `Dispose()`.

Если токен отменят, отписываемся от события и от отмены, а затем переводим источник в состояние отмены:

```csharp
private void OnCanceled()
{
    _unityEvent.RemoveListener(OnInvoked);
    _cancellationTokenRegistration.Dispose();

    _core.TrySetCanceled(_cancellationToken);
}
```

При обычном завершении тоже не забываем отписаться от отмены:

```csharp
private void OnInvoked()
{
    _unityEvent.RemoveListener(OnInvoked);
    _cancellationTokenRegistration.Dispose();

    _core.TrySetResult(AsyncUnit.Default);
}
```

Объект будет использоваться повторно, поэтому при возврате в пул сбрасываем и поля, связанные с отменой:

```csharp
private void Release()
{
    // ...
    _cancellationToken = default;
    _cancellationTokenRegistration = default;
}
```

Чем бы ни закончилось ожидание — событием или отменой, — все подписки будут сняты, а объект будет готов к следующему использованию.


## Без лишних делегатов

Источник уже работает, но при каждой подписке на событие и регистрации отмены создаются новые делегаты:

```csharp
_unityEvent.AddListener(OnInvoked);
_cancellationToken.Register(OnCanceled);
```

Каждый раз, когда метод передаётся как делегат, выделяется память. Чтобы этого избежать, создадим делегаты один раз в конструкторе и будем переиспользовать их всё время жизни объекта:

```csharp
private readonly UnityAction _onInvoked;
private readonly Action _onCanceled;

private UnityEventPromise()
{
    _onInvoked = OnInvoked;
    _onCanceled = OnCanceled;
}

private void Initialize(UnityEvent unityEvent, CancellationToken cancellationToken)
{
    // ...
    _unityEvent.AddListener(_onInvoked);
    _cancellationTokenRegistration = _cancellationToken.Register(_onCanceled);
}
```

Теперь каждое новое ожидание использует уже готовые делегаты. Отписываться тоже стоит через `_onInvoked`: `RemoveListener(OnInvoked)` создал бы ещё один делегат ради одной отписки. В полном коде ниже так и сделано.

> Экономить на одном делегате выглядит немного смешно. Но если такие ожидания создаются каждый кадр, мелочи быстро складываются.
{: .aside}


## Пул

Как и планировали, `UnityEventPromise` будет браться из пула. Создадим статический `ObjectPool` из `UnityEngine.Pool` и статический метод `Create()`, который достаёт объект из пула и готовит его к работе:

```csharp
private static readonly ObjectPool<UnityEventPromise> s_pool = new(() => new UnityEventPromise());

public static IUniTaskSource Create(
    UnityEvent unityEvent,
    CancellationToken cancellationToken,
    out short token)
{
    var promise = s_pool.Get();
    promise.Initialize(unityEvent, cancellationToken);
    token = promise._core.Version;
    return promise;
}
```

Вместе с источником `UniTask` получает `token` — номер текущей версии источника. При каждом обращении к источнику этот номер проверяется. Если объект уже вернулся в пул и был выдан заново, версии не совпадут, и `UniTask` выбросит ошибку. Именно поэтому большинство `UniTask` нельзя ждать дважды.

После завершения ожидания объект нужно сбросить и вернуть в пул. Для этого подходит метод `GetResult()`: он вызывается, когда ожидающий код забирает результат, и после него объект больше не нужен.

```csharp
public void GetResult(short token)
{
    try
    {
        _core.GetResult(token);
    }
    finally
    {
        Release();
    }
}

private void Release()
{
    // ...
    _core.Reset();
    s_pool.Release(this);
}
```

`_core.Reset()` сбрасывает состояние `UniTaskCompletionSourceCore` и увеличивает его версию. После этого объект можно выдавать снова.

> `GetResult()` вызывается и при успехе, и при отмене, и при ошибке. Поэтому `Release()` стоит в `finally`: объект вернётся в пул, даже если `_core.GetResult()` выбросит исключение.
{: .aside}


## TaskTracker

Источник готов. Осталось добавить его в `TaskTracker`, чтобы видеть активные ожидания в окне UniTask Tracker при отладке. Регистрируем источник при создании и убираем из трекера при возврате в пул:

```csharp
public static IUniTaskSource Create(
    UnityEvent unityEvent,
    CancellationToken cancellationToken,
    out short token)
{
    // ...
    TaskTracker.TrackActiveTask(promise, 3);
    return promise;
}

private void Release()
{
    TaskTracker.RemoveTracking(this);
    // ...
}
```

Трекер запоминает стек вызовов, чтобы в окне UniTask Tracker было видно, из какого места кода запущено ожидание. Число `3` говорит, сколько верхних вызовов в этом стеке пропустить: сам `TrackActiveTask()`, `Create()` и `WaitAsync()`. Тогда первой строкой в трекере будет ваш код, который вызвал `WaitAsync()`.

Трекер работает только в редакторе, а запись стека включается в самом окне UniTask Tracker.

Если ожидание висит в трекере слишком долго, значит, его так и не дождались или забыли отменить.


## Метод расширения

Осталось связать источник с `UniTask`. Для этого сделаем метод расширения для `UnityEvent`:

```csharp
public static class UnityEventExtensions
{
    public static UniTask WaitAsync(this UnityEvent unityEvent, CancellationToken cancellationToken = default)
    {
        var source = UnityEventPromise.Create(unityEvent, cancellationToken, out var token);
        return new UniTask(source, token);
    }
}
```

`UniTask` — это структура, поэтому `new UniTask(source, token)` ничего не выделяет в куче. Вызывающий код работает только с `UniTask` и ничего не знает о внутреннем источнике:

```csharp
await button.onClick.WaitAsync(cancellationToken);
```


## Полный код

Готовый класс, который можно взять как шаблон для своих источников. Здесь есть две проверки, которых не было в разборе выше: если токен уже отменён, объект из пула не берётся вовсе, а если токен нельзя отменить, на него не подписываемся.

Класс рассчитан на главный поток Unity: `UnityEvent` и `ObjectPool` не потокобезопасны, поэтому токен отмены тоже стоит отменять из главного потока.

```csharp
using System;
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine.Events;
using UnityEngine.Pool;

public sealed class UnityEventPromise : IUniTaskSource
{
    private static readonly ObjectPool<UnityEventPromise> s_pool = new(() => new UnityEventPromise());

    public static IUniTaskSource Create(UnityEvent unityEvent, CancellationToken cancellationToken, out short token)
    {
        // Токен уже отменён: сразу возвращаем отменённую задачу и ничего не берём из пула.
        if (cancellationToken.IsCancellationRequested)
            return AutoResetUniTaskCompletionSource.CreateFromCanceled(cancellationToken, out token);

        var promise = s_pool.Get();
        promise.Initialize(unityEvent, cancellationToken);
        token = promise._core.Version;

        TaskTracker.TrackActiveTask(promise, 3);

        return promise;
    }

    private readonly UnityAction _onInvoked;
    private readonly Action _onCanceled;

    private UniTaskCompletionSourceCore<AsyncUnit> _core;
    private UnityEvent _unityEvent;
    private CancellationToken _cancellationToken;
    private CancellationTokenRegistration _cancellationTokenRegistration;

    private UnityEventPromise()
    {
        _onInvoked = OnInvoked;
        _onCanceled = OnCanceled;
    }

    private void Initialize(UnityEvent unityEvent, CancellationToken cancellationToken)
    {
        _unityEvent = unityEvent;
        _cancellationToken = cancellationToken;

        _unityEvent.AddListener(_onInvoked);

        // Токен без источника отмены (например, CancellationToken.None) отменить нельзя.
        if (_cancellationToken.CanBeCanceled)
            _cancellationTokenRegistration = _cancellationToken.Register(_onCanceled);
    }

    private void OnInvoked()
    {
        _unityEvent.RemoveListener(_onInvoked);
        _cancellationTokenRegistration.Dispose();

        _core.TrySetResult(AsyncUnit.Default);
    }

    private void OnCanceled()
    {
        _unityEvent.RemoveListener(_onInvoked);
        _cancellationTokenRegistration.Dispose();

        _core.TrySetCanceled(_cancellationToken);
    }

    private void Release()
    {
        TaskTracker.RemoveTracking(this);

        _unityEvent = null;
        _cancellationToken = default;
        _cancellationTokenRegistration = default;

        _core.Reset();

        s_pool.Release(this);
    }

    // Методы IUniTaskSource

    public void GetResult(short token)
    {
        try
        {
            _core.GetResult(token);
        }
        finally
        {
            Release();
        }
    }

    public void OnCompleted(Action<object> continuation, object state, short token)
        => _core.OnCompleted(continuation, state, token);

    public UniTaskStatus GetStatus(short token)
        => _core.GetStatus(token);

    public UniTaskStatus UnsafeGetStatus()
        => _core.UnsafeGetStatus();
}
```

```csharp
public static class UnityEventExtensions
{
    public static UniTask WaitAsync(this UnityEvent unityEvent, CancellationToken cancellationToken = default)
    {
        var source = UnityEventPromise.Create(unityEvent, cancellationToken, out var token);
        return new UniTask(source, token);
    }
}
```


## Когда это нужно

Свой источник — это больше кода и больше мест, где можно ошибиться. Поэтому начинать стоит с готовых вариантов:

- нужно сохранить результат для всех, кто придёт позже, — [UniTaskCompletionSource]({% link _unitask/010-UniTaskCompletionSource.md %});
- нужно дождаться разового события — [AutoResetUniTaskCompletionSource]({% link _unitask/011-AutoResetUniTaskCompletionSource.md %}) или готовые методы из `UniTask`, например `OnClickAsync()`.

К своему `IUniTaskSource` стоит переходить, когда профайлер показывает, что аллокации от ожиданий действительно мешают. Обычно это ожидания, которые создаются каждый кадр или для множества объектов сразу.

> Хорошая новость: такой класс пишется один раз. Для другого события достаточно скопировать его и поменять подписку.
{: .aside}
