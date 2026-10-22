---
layout: post
title: "AutoResetUniTaskCompletionSource: как превратить событие в UniTask"
short_title: "AutoResetUniTaskCompletionSource"
slug: auto-reset-unitask-completion-source
last_modified_at: 2026-10-23
redirect_from: /unity/unitask/011-AutoResetUniTaskCompletionSource/
date: 2026-09-11
order: 11
description: "Как превратить разовое событие или колбэк в UniTask и дождаться его через await."
topic: unitask
---

# AutoResetUniTaskCompletionSource

Не всё в Unity возвращает `UniTask`. Нажатие кнопки, срабатывание триггера, ответ от SDK — обо всём этом сообщают события и колбэки. Чтобы дождаться их через `await`, их нужно превратить в `UniTask`.

`AutoResetUniTaskCompletionSource` делает именно это. Он создаёт `UniTask`, который завершится тогда, когда вы скажете. А после того как результат забрали, источник возвращается в пул и используется снова.

> Это вторая из трёх заметок о том, как превращать события и колбэки в `UniTask`. Если нужен результат, который сохранится для всех, кто придёт позже, — это [UniTaskCompletionSource]({% link _unitask/010-UniTaskCompletionSource.md %}).
{: .aside}


## Как это работает

Источник выдаёт `UniTask` через свойство `.Task`. Этот `UniTask` не завершится сам: его завершают вручную одним из трёх методов:

- `TrySetResult()` — операция завершилась успешно;
- `TrySetCanceled()` — операция отменена;
- `TrySetException()` — операция завершилась с ошибкой.

Главная особенность: **после того как `await` получил результат, источник сбрасывается и возвращается в пул**. Вместо того чтобы на каждое ожидание создавать новый объект и потом отдавать его сборщику мусора, `UniTask` берёт объект из пула, а после использования кладёт обратно.

Из этого следует простое правило: **выданный `UniTask` нужно дождаться, и ровно один раз**.

- Если его никто не ждёт, источник не вернётся в пул. Ошибки не будет, но появится лишняя аллокация — то, от чего `UniTask` как раз старается избавиться.
- Если ждать его второй раз, будет ошибка: источник к этому моменту уже сброшен и, возможно, выдан кому-то другому.


## Как использовать

Источник создаётся не через `new`, а через метод `Create()`: так он берётся из пула.

```csharp
var tcs = AutoResetUniTaskCompletionSource.Create();

// Где-то ждём завершения
await tcs.Task;
Debug.Log("Завершено");

// Где-то в другом месте сообщаем, что операция закончилась
tcs.TrySetResult();
```

Когда вызывается `TrySetResult()`, код после `await tcs.Task` продолжает выполнение. Как только `await` забрал результат, источник сбрасывается и возвращается в пул.

Если операция должна вернуть значение, используется `AutoResetUniTaskCompletionSource<T>`, а значение передаётся в `TrySetResult()`.


## Пример: дождаться нажатия кнопки

Простой сценарий: в туториале нужно дождаться, пока игрок нажмёт кнопку.

```csharp
await WaitForClick(_button);
Debug.Log("Кнопка нажата");
```

Напишем метод `WaitForClick`, который принимает кнопку:

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

`WaitForClick` подписывается на `onClick` и сразу возвращает `UniTask`. Пока кнопку не нажали, этот `UniTask` не завершён.

Когда игрок нажимает кнопку, вызывается `OnClick()`. Он отписывается от события и вызывает `TrySetResult()`. `UniTask` завершается, и код после `await` продолжает работу.

> Для кнопок в `UniTask` уже есть готовый `button.OnClickAsync()`. Здесь кнопка выбрана просто потому, что это самый наглядный пример. Тем же способом можно превратить в `UniTask` любое событие или колбэк.
{: .aside}


## Отмена

Ожидание часто нужно уметь прервать. Например, если окно закрыли раньше, чем игрок нажал кнопку.

Для этого в C# есть `CancellationToken`. Метод принимает токен и через `Register()` подписывается на его отмену:

```csharp
public UniTask WaitForClick(Button button, CancellationToken token)
{
    var tcs = AutoResetUniTaskCompletionSource.Create();
    CancellationTokenRegistration registration = default;

    button.onClick.AddListener(OnClick);
    registration = token.Register(OnCancel);

    return tcs.Task;

    void OnClick()
    {
        button.onClick.RemoveListener(OnClick);
        registration.Dispose();
        tcs.TrySetResult();
    }

    void OnCancel()
    {
        button.onClick.RemoveListener(OnClick);
        registration.Dispose();
        tcs.TrySetCanceled();
    }
}
```

Теперь у ожидания два возможных конца: игрок нажал кнопку или ожидание отменили. В обоих случаях метод снимает обе подписки — и на кнопку, и на отмену.

Переменная `registration` объявлена заранее, потому что к ней обращаются локальные методы. Если написать сразу `var registration = token.Register(OnCancel);`, компилятор не даст использовать её внутри `OnCancel()`.

Когда токен отменяют, срабатывает `OnCancel()`, и `UniTask` переходит в состояние отмены. Код, который его ждёт, получит исключение `OperationCanceledException`. Его ловят обычным `try/catch`:

```csharp
try
{
    await WaitForClick(_button, token);
}
catch (OperationCanceledException)
{
    Debug.Log("Ожидание нажатия отменено");
}
```

При отмене источник точно так же сбрасывается и возвращается в пул, как только `await` получит результат.

> Без отмены ожидание может зависнуть навсегда: кнопку уже уничтожили, а метод всё ещё ждёт её нажатия. Код после await так и не выполнится, а источник не вернётся в пул.
{: .aside}


## Когда использовать

`AutoResetUniTaskCompletionSource` подходит для разовых событий, результат которых не нужно хранить: нажатие кнопки, столкновение, срабатывание триггера, ответ на запрос.

Если результат нужно сохранить для всех, кто придёт позже, лучше взять `UniTaskCompletionSource`:

| | `UniTaskCompletionSource` | `AutoResetUniTaskCompletionSource` |
|---|---|---|
| Результат после завершения | сохраняется | сбрасывается |
| Сколько раз можно ждать | сколько угодно | ровно один раз |
| Объект на каждое ожидание | новый | из пула |
| Типичные случаи | загрузка, инициализация | клики, триггеры, сигналы |

Если ожиданий очень много, например каждый кадр, можно пойти ещё дальше и написать свой источник без лишних аллокаций. Об этом — в заметке про [IUniTaskSource]({% link _unitask/012-IUniTaskSource.md %}).
