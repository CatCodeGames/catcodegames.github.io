---
layout: post
title: "AutoResetUniTaskCompletionSource"
date: 2026-09-11
order: 11
description: "Как превратить событие или callback в UniTask и дождаться его через await."
topic: unitask
---

# AutoResetUniTaskCompletionSource

## Зачем нужен

`AutoResetUniTaskCompletionSource` — как и `UniTaskCompletionSource`, позволяет создать `UniTask`, завершением которого можно управлять. Применяется, когда операция не возвращает `UniTask`, но её нужно дождаться через `await`.

С помощью `AutoResetUniTaskCompletionSource` **можно преобразовать событие или callback в `UniTask`**, который можно ожидать через `await`.

Через свойство `.Task` можно получить связанный с источником `UniTask`. А с помощью следующих методов управлять состоянием операции:

- `TrySetResult()` — успешно завершает операцию;
- `TrySetCanceled()` — отменяет операцию;
- `TrySetException()` — завершает операцию с указанной ошибкой.

Главное отличие от `UniTaskCompletionSource` — после получения результата `AutoResetUniTaskCompletionSource` **сбрасывает своё состояние и возвращается в пул**. Поэтому он подходит для одноразовых сигналов, когда завершённое состояние не нужно сохранять для последующих ожиданий.


## Как использовать

Работа с `AutoResetUniTaskCompletionSource` сводится к следующим действиям:

- создаём источник с помощью метода `Create()`:

``` csharp
var utcs = AutoResetUniTaskCompletionSource.Create();
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

После этого связанный `UniTask` считается завершённым, и код после `await utcs.Task` начинает выполняться дальше. Когда `await` получает результат и выходит из задачи, `AutoResetUniTaskCompletionSource` сбрасывает своё внутреннее состояние и возвращается в пул


### Пример
Рассмотрим простой сценарий: нужно дождаться нажатия на кнопку. Например, в туториале может потребоваться дождаться определённого действия игрока.

```csharp
await WaitForClick(_button);
Debug.Log("Кнопка была нажата");
```

Создадим метод `WaitForClick`, принимающий кнопку:

```csharp
public UniTask WaitForClick(Button button)
{
    var tcs = AutoResetUniTaskCompletionSource.Create();
    button.onClick.AddListener(OnClick);

    return tcs.Task;

    void OnClick()
    {
        button.onClick.RemoveListener(OnClick);        
        tcs.TrySetResult();
    }
}
```

Метод `WaitForClick` подписывается на `onClick` и возвращает `UniTask`, связанный с `AutoResetUniTaskCompletionSource`. Пока кнопка не нажата, этот `UniTask` остаётся незавершённым.

Когда пользователь нажимает кнопку, вызывается метод `OnClick()`. Обработчик отписывается от события и вызывает `TrySetResult()`, завершая ожидаемый `UniTask`. После того как `await` получит результат, `AutoResetUniTaskCompletionSource` сбрасывает своё состояние и возвращается в пул.

> Использование `AutoResetUniTaskCompletionSource` предполагает, что **выданную задачу нужно обязательно дождаться**. Если `UniTask` никто не ждёт, связанный источник не проходит этап получения результата и не возвращается в пул. Это не приводит к ошибке, но приводит к лишним аллокациям. Что немного противоречит самой идее `UniTask` как библиотеки с низкими аллокациями.


### Отмена 
Чтобы можно было отменить ожидание, метод должен принимать `CancellationToken` и зарегистрировать обработчик отмены через `Register()`.

```csharp
public UniTask WaitForClick(Button button, CancellationToken token)
{
    var tcs = AutoResetUniTaskCompletionSource.Create();
    CancellationTokenRegistration ctr = default;

    button.onClick.AddListener(OnClick);
    ctr = token.Register(OnCancel);

    return tcs.Task;

    void OnClick()
    {
        button.onClick.RemoveListener(OnClick);
        ctr.Dispose();
        tcs.TrySetResult();
    }
    
    void OnCancel()
    {
        button.onClick.RemoveListener(OnClick);
        ctr.Dispose();
        tcs.TrySetCanceled();
    }
}
```
> `CancellationTokenRegistration` объявляется заранее, потому что локальный метод `OnCancel()` использует переменную `ctr`. Если сразу написать `var ctr = token.Register(OnCancel)`, компилятор не позволит использовать `ctr` внутри `OnCancel()`.

Когда вызывается `Cancel()` у `CancellationTokenSource`, токен помечается как отменённый и вызывается обработчик `OnCancel()`. Он отписывает от события и переводит задачу в состояние отмены. Если в этот момент задача ожидается через `await`, будет выброшено исключение `OperationCanceledException`, которое нужно обработать в `try/catch`

``` csharp
try
{
    await WaitForClick(_button, token);
}
catch (OperationCanceledException)
{
    Debug.Log("Ожидание нажатия на кнопку отменено");
}
```

> При отмене, так же как и при успешном завершении, `AutoResetUniTaskCompletionSource` сбрасывает своё состояние и возвращается в пул после того, как `await` получает результат.



## Когда и где использовать

`AutoResetUniTaskCompletionSource` подходит для разовых событий, результат которых не нужно сохранять: например, для ожидания нажатия кнопки, столкновения, срабатывания триггера и других событий.

Также `AutoResetUniTaskCompletionSource` удобно использовать, когда нужно ждать следующее событие. Например, следующий тик таймера или для примера следующее нажатие кнопки:

```csharp
public sealed class ClickService
{
    private readonly List<AutoResetUniTaskCompletionSource> _waiters = new();
    private readonly Button _button;

    public ClickService(Button button)
    {
        _button = button;
        _button.onClick.AddListener(OnClick);
    }

    public UniTask WaitNextClick()
    {
        var tcs = AutoResetUniTaskCompletionSource.Create();
        _waiters.Add(tcs);
        return tcs.Task;
    }

    private void OnClick()
    {
        foreach (var tcs in _waiters)
            tcs.TrySetResult();

        _waiters.Clear();
    }
}
```

Теперь можно дождаться следующего клика:

``` csharp
await service.WaitNextClick();
```

После нажатия все текущие ожидания завершаются. После получения результата их источники сбрасываются и возвращаются в пул. Для ожидания следующего события можно снова вызвать `WaitNextClick()`.

Такой подход позволяет разово 

## Когда и где использовать

`AutoResetUniTaskCompletionSource` полезен, когда нужно адаптировать ожидание операции под `async/await`. Он подходит для разовых событий, результат которых не нужно сохранять: например, для ожидания нажатия кнопки, столкновения, срабатывания триггера и других событий.

Также `AutoResetUniTaskCompletionSource` удобно использовать, когда нужно ждать следующее событие. Например, следующий тик таймера или следующее нажатие кнопки:

``` csharp
public sealed class ClickService
{
    private readonly List<AutoResetUniTaskCompletionSource> _waiters = new();
    private readonly Button _button;

    public ClickService(Button button)
    {
        _button = button;
        _button.onClick.AddListener(OnClick);
    }

    public UniTask WaitNextClick()
    {
        var tcs = AutoResetUniTaskCompletionSource.Create();
        _waiters.Add(tcs);
        return tcs.Task;
    }

    private void OnClick()
    {
        foreach (var tcs in _waiters)
            tcs.TrySetResult();

        _waiters.Clear(); 
    }
}
``` 

Теперь разные классы могут в нужный момент дождаться его следующего клика через `await`:
``` csharp
await service.WaitNextClick();
```

После нажатия все текущие ожидания завершаются. После получения результата их источники сбрасываются и возвращаются в пул. Если потребуется ожидание следующего события, то снова вызвать `WaitNextClick()`.