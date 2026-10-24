---
layout: post
title: "Async/await в Unity: основы для начинающих"
short_title: "Async/await в Unity"
slug: async-await-basic
last_modified_at: 2026-10-23
redirect_from: /unity/unitask/001-async-await-basic/
date: 2026-09-21
order: 01
description: "Как работает async/await, зачем он нужен и как его использовать в Unity."
topic: unitask
---

# Async/await

Игровой логике постоянно нужно чего-то ждать: нажатия кнопки, конца анимации, ответа сервера. `async/await` позволяет **дождаться такой операции и продолжить со следующей строки** — без колбэков и обработчиков, разбросанных по классу.

> Это небольшая практическая заметка для тех, кто только знакомится с `async/await` и `UniTask`. Без углубления во внутреннее устройство — только то, что нужно, чтобы начать пользоваться.
{: .aside}


## Пример

Возьмём простой сценарий: нужно дождаться нажатия двух кнопок в любом порядке.

Обычно такое делают через события. И приходится вручную запоминать, какие кнопки уже нажаты:

```csharp
bool leftPressed;
bool rightPressed;

void Start()
{
    leftButton.onClick.AddListener(OnLeftButtonClicked);
    rightButton.onClick.AddListener(OnRightButtonClicked);
}

void OnLeftButtonClicked()
{
    leftPressed = true;
    CheckComplete();
}

void OnRightButtonClicked()
{
    rightPressed = true;
    CheckComplete();
}

void CheckComplete()
{
    if (leftPressed && rightPressed)
    {
        leftButton.onClick.RemoveListener(OnLeftButtonClicked);
        rightButton.onClick.RemoveListener(OnRightButtonClicked);

        Continue();
    }
}
```

Здесь нет ничего принципиально плохого: код работает. Но ради одной задачи «дождаться двух кнопок» понадобились два флага, три метода и отписка, о которой нельзя забыть.

Теперь то же самое с `UniTask`:

```csharp
async UniTask WaitForBothButtons()
{
    await UniTask.WhenAll(
        leftButton.OnClickAsync(),
        rightButton.OnClickAsync()
    );
    Continue();
}
```

Код читается как описание задачи: дождаться нажатия обеих кнопок, затем продолжить. Флаги не нужны. Отписываться тоже не нужно: `OnClickAsync` делает это сам.

Главное отличие — **вся логика ожидания находится в одном методе**, а не разнесена по обработчикам.

> С `async/await` сценарий остаётся тем же, меняется только способ его записи.


## Два ключевых слова

`await` ставится перед операцией, которую нужно дождаться. Буквально: «дождись завершения и продолжай». Когда операция завершится, метод продолжит работу со следующей строки.

`async` ставится перед методом, в котором есть `await`. Без него компилятор не разрешит использовать `await`.

Такой метод должен возвращать тип, который можно дождаться. Чаще всего это стандартный `Task` из .NET, `Awaitable` из Unity и `UniTask` из одноимённой библиотеки.

> Эти заметки посвящены `UniTask`, поэтому все примеры написаны на нём. Почему именно он — в заметке [UniTask, Task, Awaitable или корутины]({% link _unitask/002-unitask-vs-alternatives.md %}).
{: .aside}


## Чего async/await не делает

- не ускоряет код;
- не создаёт новые потоки;
- не делает код параллельным.

`async/await` — это механизм ожидания. По сути, это тот же колбэк, только записанный удобнее.

Код после `await` выполнится, когда операция завершится. Ровно так же, как выполнился бы обработчик события. И выполнится он там же: как правило, в главном потоке Unity.

Поэтому `async/await` — это не многопоточность. Если после `await` посчитать что-то тяжёлое, игра зависнет так же, как в обычном обработчике.

<details markdown="1">
<summary>А если работа выполняется в другом потоке?</summary>

Тяжёлую работу можно отправить в другой поток, например через `UniTask.RunOnThreadPool`. Пока она выполняется, главный поток не ждёт: игра продолжает работать.

Когда работа закончится, ожидающий метод продолжит выполнение уже в главном потоке.
</details>


## А корутины?

Корутины тоже умеют ждать. Для простой задержки их вполне хватает.

Сложности начинаются, когда от ожидания нужен результат. Корутина не умеет возвращать значение, поэтому его передают через колбэк:

```csharp
// Корутина: результат приходится передавать в колбэк
StartCoroutine(LoadLevel(level => Show(level)));

// async/await: результат возвращается, как у обычного метода
var level = await LoadLevelAsync();
Show(level);
```

Ещё одно отличие: корутина останавливается вместе с объектом, а асинхронный метод продолжит работу, даже если объект уже уничтожен. Чтобы этого не происходило, ожидание нужно отменять через токен отмены — об этом в заметке про [AutoResetUniTaskCompletionSource]({% link _unitask/011-AutoResetUniTaskCompletionSource.md %}).


## Что дальше

Как самому превращать события и колбэки в `UniTask`, от простого к сложному:

- [UniTaskCompletionSource]({% link _unitask/010-UniTaskCompletionSource.md %}) — ожидание с сохранённым результатом;
- [AutoResetUniTaskCompletionSource]({% link _unitask/011-AutoResetUniTaskCompletionSource.md %}) — одноразовые ожидания и отмена;
- [IUniTaskSource]({% link _unitask/012-IUniTaskSource.md %}) — свой источник без лишних аллокаций.

Как проектировать API с `UniTask`:

- [Не прячьте UniTask внутри результата]({% link _unitask/100-unitask-dont-hide.md %});
- [Последовательные сценарии в UniTask]({% link _unitask/110-unitask-sequential-workflow.md %}).
