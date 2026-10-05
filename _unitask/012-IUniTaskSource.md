---
layout: post
title: "IUniTaskSource"
date: 2026-10-05
order: 12
description: "Самый эффективный способ создать UniTask"
topic: unitask
---

# IUniTaskSource

`UniTaskCompletionSource` и `AutoResetUniTaskCompletionSource` позволяют преобразовать события и callback в `UniTask`. 
Однако они не позволяет полностью избежать аллокаций, возникающих из-за локальных методов и захвата переменных.

В таких случаях можно создать собственный источник, реализующий `IUniTaskSource`, и взять логику ожидания под свой контроль.

В качестве примера реализуем ожидание `UnityEvent`.

## IUniTaskSource и UniTaskCompletionSourceCore

Создадим класс `UnityEventPromise` реализующего `IUniTaskSource`.

``` csharp
public sealed class UnityEventPromise : IUniTaskSource
{
    public void GetResult(short token) {}
    public void OnCompleted(Action<object> continuation, object state, short token) {}
    public UniTaskStatus GetStatus(short token) {} 
    public UniTaskStatus UnsafeGetStatus() {}
}
```

Подробно разбирать назначение методов не будем. Достаточно знать, что через них `UniTask` проверяет состояние операции, регистрирует продолжение и получает результат после завершения ожидания.

Реализовывать эту логику самостоятельно не требуется. В `UniTask` уже есть `UniTaskCompletionSourceCore<T>`, который хранит состояние операции и содержит всю необходимую механику работы.

Добавим его в класс и используем внутри методов интерфейса.

``` csharp

private UniTaskCompletionSourceCore<AsyncUnit> _core = new();

public sealed class UnityEventPromise : IUniTaskSource
{
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

`UniTaskCompletionSourceCore<T>` требует указать тип результата. Поскольку `UnityEvent` не возвращает никаких  данных, используем `AsyncUnit` — специальный тип для операций без результата.

На этом этапе источник уже умеет хранить своё состояние и работать с `UniTask`, но пока его ничто не завершает.

## Ожидание UnityEvent

Чтобы завершить ожидание, источнику нужно знать, какое событие он ожидает. Для этого передадим ему `UnityEvent`.

Сразу учтём, что мы не будем создавать новый объект для каждого ожидания. Наша цель - минимизировать аллокации, поэтому **`UnityEventPromise` будет переиспользоваться через пул**.

Из-за этого передавать `UnityEvent` через конструктор нельзя. После получения `UnityEventPromise` из пула ему нужно передать событие, а перед возвратом очистить состояние.

Для этого добавим методы `Initialize()` и `Release()`.

``` csharp
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

Источник должен реагировать на событие. Для этого добавим обработчик и подпишемся на событие `UnityEvent`.

``` csharp
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

Когда событие срабаотает, источник перейдёт в завершённое состояние через `TrySetResult()`. После этого ожидающий `UniTask` завершится, а выполнение продолжится после `await`.


## Отмена ожидания

Теперь добавим поддержку отмены. 
Передадим в `Initialize()` объект `CancellationToken` и зарегистрируем обработчик, который будет вызван при отмене ожидания.

``` csharp
private CancellationToken _cancelationToken;
private CancellationTokenRegistration _cancellationTokenRegistration;

private void Initialize(UnityEvent unityEvent, CancellationToken cancellationToken)
{
    ...
    _cancellationToken = cancellationToken
    _cancellationTokenRegistration = _cancellationToken.Register(OnCanceled);
}

```

> `Register()` возвращает `CancellationTokenRegistration`. Сохраняем его в соотвествующее поле, чтобы позже можно было отписаться от отмены через `Dispose()`.

Если токен будет отменён, отписываемся от события и отмены, после чего переводим источник в состояние отмены.

``` csharp
private void OnCanceled()
{
    _unityEvent.RemoveListener(OnInvoked);
    _cancellationTokenRegistration.Dispose();
         
    _core.TrySetCanceled(_cancellationToken);
}
```

При успешном завершении ожидания также не забываем отписаться от отмены.

``` csharp
private void OnInvoked()
{
    _unityEvent.RemoveListener(OnInvoked);
    _cancellationTokenRegistration.Dispose();
 
    _core.TrySetResult(AsyncUnit.Default);
}
```

Поскольку объект будет использоваться повторно, при возврате в пул сбросим поля, связанные с отменой.

``` csharp 
private void Release()
{
    ...
    _cancellationToken = default;
    _cancellationTokenRegistration = default;
    ... 
}
```

Независимо от того, завершилось ожидание через событие или через отмену, все подписки будут корректно сняты, а объект останется готовым к следующему использованию.

## Оптимизация

На этом этапе источник уже работает, но при каждой подписке на событие и регистрации отмены будут создаваться новые экземпляры делегатов.

``` csharp
_unityEvent.AddListener(OnInvoked);
_cancellationToken.Register(OnCanceled);
```

Чтобы избежать этих аллокаций, создадим делегаты один раз и будем переиспользовать их на протяжении всей жизни объекта.

``` csharp
private readonly UnityAction _onInvoked;
private readonly Action _onCanceled;

private UnityEventPromise()
{
    _onInvoked = OnInvoked;
    _onCanceled = OnCanceled;
}

private void Initialize(UnityEvent unityEvent, CancellationToken cancellationToken)
{
    ...
    _unityEvent.AddListener(_onInvoked);    
    _cancellationTokenRegistration = _cancellationToken.Register(_onCanceled);
}
```

После этого при каждом новом ожидании будут использоваться уже существующие экземпляры делегатов.

## Пул

Как говорилось ранее, экземпляры `UnityEventPromise` будут переиспользоваться через пул. Для этого создадим статический `ObjectPool` и статический метод `Create()`, который будет получать объект из пула и подготавливать его к работе.

``` csharp
private static readonly ObjectPool<UnityEventPromise> s_pool = new(() => new());

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

Вместе с источником `UniTask` получает `token` — значение его текущей версии. При обращении к источнику это значение проверяется. Если объект уже вернулся в пул и был переиспользован, версии не совпадут, и `UniTask` выдаст ошибку.
Именно поэтому, в большинстве случаев, `UniTask` и нельзя ожидать повторно.

После завершения ожидания объект нужно сбросить и вернуть в пул. 
Для этого воспользуемся метод `GetResult()` интерфейса `IUniTaskSource`. Он вызывается после того, как ожидающий код получил результат и объект будет больше не нужен.

``` csharp
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
    ...
    _core.Reset();
    s_pool.Release(this);
}
```

Перед возвратом в пул также нужно сбросить состояние `UniTaskCompletionSourceCore` ввключая его версию. Это позволит использовать его повторно.

## TaskTracker

Источник полностью готов, осталось добавить его отслеживание при отладке. Для этого зарегистрируем его в `TaskTracker` при создании и удалим из трекера после завершения ожидания.

```csharp

public static IUniTaskSource Create(
    UnityEvent unityEvent,
    CancellationToken cancellationToken,
    out short token)
{
    ... 
    TaskTracker.TrackActiveTask(promise, 3);
    return promise;
}

private void Release()
{
    TaskTracker.RemoveTracking(this);
    ...
}
```
3 — тип задачи, который используется `TaskTracker` при отображении источника.

## Создание UniTask

Теперь осталось связать готовый источник с `UniTask`. Для этого можно создать метод расширения для `UnityEvent`, который вернёт `UniTask`.

``` csharp
public static WaitAsync(this UnityEvent unityEvent, CancellationToken cancalletionToken)
    var source = UnityEventPromise.Create(
        unityEvent,
        cancellationToken,
        out var token);
 
    return new UniTask(source, token);
```

В результате внешний код работает только с UniTask, не зная о внутреннем источнике:

``` csharp
await button.onClick.WaitAsync(cancellationToken);
```

## UnityEventPromise

Ниже полностью готовый и рабочий код, который можно использовать как шаблон для создания своих классов.

```
public sealed class UnityEventPromise : IUniTaskSource
{
    private readonly static ObjectPool<UnityEventPromise> s_pool = new(() => new());

    public static IUniTaskSource Create(UnityEvent unityEvent, CancellationToken cancellationToken, out short token)
    {
        if (cancellationToken.IsCancellationRequested)
            return AutoResetUniTaskCompletionSource.CreateFromCanceled(cancellationToken, out token);

        var promise = s_pool.Get();
        promise.Init(unityEvent, cancellationToken);
        token = promise._core.Version;

        TaskTracker.TrackActiveTask(promise, 3);

        return promise;
    }

    private readonly UnityAction _onInvoked;
    private readonly Action _onCanceled;

    private UniTaskCompletionSourceCore<AsyncUnit> _core = new();
    private UnityEvent _unityEvent;
    private CancellationToken _cancellationToken;
    private CancellationTokenRegistration _cancellationTokenRegistration;

    private UnityEventPromise()
    {
        _onInvoked = OnInvoked;
        _onCanceled = OnCanceled;
    }

    private void Init(UnityEvent unityEvent, CancellationToken cancellationToken)
    {
        _unityEvent = unityEvent;
        _cancellationToken = cancellationToken;

        _unityEvent.AddListener(_onInvoked);
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

    // методы IUniTaskSource

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