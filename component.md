# QuestionCard

## Часть 1. Архитектура компонента

### Структура компонентов

```
QuestionCard (container)
├── QuestionStem
│   └── TipTapRenderer
│       └── KaTeXRenderer
├── AnswerOptions
│   └── AnswerOption (xN)
├── ActionBar
│   └── CheckAnswerButton
├── Explanation (conditional)
│   └── DemoOverlay (conditional)
└── ErrorBoundary (для stem / KaTeX)
```

### Хранение состояния

**Локальное состояние (QuestionCard):**

- `selectedAnswerId` — выбранный вариант ответа
- `isChecked` — был ли выполнен check answer
- `isChecking` — состояние запроса (loading / debounce)
- `localError` — ошибки рендера (например, KaTeX)

**Глобальное состояние (store / context):**

- `questionId`
- `questionData`
- `userMode` (demo / paid)
- `checkAnswerResult` (если проверка идёт через API)

### Ответы на вопросы

**Где хранится selectedAnswer**  
Локально в `QuestionCard`. Это одноразовое UI-состояние, не нужное вне компонента.

**Где хранится isChecked**  
Локально в `QuestionCard`. Управляет UI (дизейблы, показ explanation).

**Что сбрасывается при смене questionId**

- `selectedAnswerId`
- `isChecked`
- `isChecking`
- `localError`

Сброс происходит при реакции на изменение `questionId`.

**Что происходит при очень быстрых кликах**

- Выбор ответа синхронный, последний клик считается актуальным
- Кнопка `Check Answer` дизейблится при `isChecking === true`
- Повторные клики игнорируются
- Ответ API, пришедший после смены `questionId`, игнорируется

---

## Часть 2. Псевдокод логики

```
// state
state:
  selectedAnswerId = null
  isChecked = false
  isChecking = false
  localError = null

// derived
canCheck = selectedAnswerId != null && !isChecked && !isChecking

onAnswerSelect(answerId):
  if (isChecked) return
  selectedAnswerId = answerId

onCheckAnswer():
  if (!canCheck) return

  isChecking = true

  result = checkAnswerAPI(questionId, selectedAnswerId)

  if (questionId changed while waiting):
    return // ignore outdated response

  isChecked = true
  isChecking = false
  save result

onQuestionChange(newQuestionId):
  resetState()

resetState():
  selectedAnswerId = null
  isChecked = false
  isChecking = false
  localError = null

// render logic
render:
  QuestionStem
    if render error -> show fallback

  AnswerOptions
    disabled = isChecked || isChecking

  CheckAnswerButton
    disabled = !canCheck
    loading = isChecking

  if (isChecked):
    if (userMode === demo):
      show DemoOverlay
    else if (explanation exists):
      show Explanation
```

### Логические ограничения

- Explanation нельзя показывать до нажатия `Check Answer`
- После `isChecked === true`:
  - ответы недоступны для изменения
  - кнопка проверки недоступна
- При новом вопросе состояние всегда сбрасывается

---

## Часть 3. Edge cases и UX

### Explanation отсутствует

- После check показывается текст:  
  **«Объяснение недоступно для этого вопроса»**
- Layout карточки не прыгает

### В stem только формулы

- Контейнер центрируется
- KaTeX в block-режиме
- Нет лишних текстовых отступов

### Очень длинный текст в stem

- Ограничение ширины строки
- Скролл внутри карточки или фиксированная длина строки + `...`
- ActionBar всегда видим (sticky)

### KaTeX упал с ошибкой

- Ошибка ловится ErrorBoundary
- Показывается fallback:  
  **«Не удалось отобразить формулу»**
- Вопрос остаётся интерактивным

### Пользователь пытается изменить ответ после check

- Ответы задизейблены
- Tooltip:  
  **«Ответ уже зафиксирован»**

### Demo режим

- Explanation скрыт или заблюрен
- Поясняющий текст:  
  **«Подробное решение доступно в полной версии»**
- CTA кнопка:  
  **«Открыть полный доступ»**
- Переход на оплату не ломает состояние вопроса

---

## Итог

- Локальное состояние для быстрого и предсказуемого UI
- Guard по `questionId` для защиты от плохого API
- Disabled + loading для защиты от спама кликов
- ErrorBoundary для устойчивого рендера
- UX не блокируется из-за ошибок контента
