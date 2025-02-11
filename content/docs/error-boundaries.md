---
id: error-boundaries
title: Предохранители
permalink: docs/error-boundaries.html
---

Ранее ошибки JavaScript внутри компонентов портили внутреннее состояние React и заставляли его [выдавать](https://github.com/facebook/react/issues/4026) [таинственные](https://github.com/facebook/react/issues/6895) [сообщения об ошибках](https://github.com/facebook/react/issues/8579) во время следующего рендера. Эти сообщения всегда вызывались ошибками, расположенными где-то выше в коде приложения, но React не предоставлял способа адекватно обрабатывать их в компонентах и не мог обработать их самостоятельно.

## Представляем предохранители (компоненты Error Boundary) {#introducing-error-boundaries}

Ошибка JavaScript где-то в коде UI не должна прерывать работу всего приложения. Чтобы исправить эту проблему для React-пользователей, React 16 вводит концепцию «предохранителя» (error boundary).

Предохранители — это компоненты React, которые **отлавливают ошибки JavaScript в любом месте деревьев их дочерних компонентов, сохраняют их в журнале ошибок и выводят запасной UI** вместо рухнувшего дерева компонентов. Предохранители отлавливают ошибки при рендеринге, в методах жизненного цикла и конструкторах деревьев компонентов, расположенных под ними.

> Примечание
>
> Предохранители **не** поймают ошибки в:
>
> * обработчиках событий ([подробнее](#how-about-event-handlers));
> * асинхронном коде (например колбэках из `setTimeout` или `requestAnimationFrame`);
> * серверном рендеринге (Server-side rendering);
> * самом предохранителе (а не в его дочерних компонентах).

Классовый компонент является предохранителем, если он включает хотя бы один из следующих методов жизненного цикла: [`static getDerivedStateFromError()`](/docs/react-component.html#static-getderivedstatefromerror) или [`componentDidCatch()`](/docs/react-component.html#componentdidcatch). Используйте `static getDerivedStateFromError()` при рендеринге запасного UI в случае отлова ошибки. Используйте `componentDidCatch()` при написании кода для журналирования информации об отловленной ошибке.

```js{7-10,12-15,18-21}
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // Обновить состояние с тем, чтобы следующий рендер показал запасной UI.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Можно также сохранить информацию об ошибке в соответствующую службу журнала ошибок
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // Можно отрендерить запасной UI произвольного вида
      return <h1>Что-то пошло не так.</h1>;
    }

    return this.props.children; 
  }
}
```

И можно дальше им пользоваться, как обыкновенным компонентом:

```js
<ErrorBoundary>
  <MyWidget />
</ErrorBoundary>
```

Предохранители работают как JavaScript-блоки `catch {}`, но только для компонентов. Только классовые компоненты могут выступать в роли предохранителей. На практике чаще всего целесообразным будет один раз описать предохранитель и дальше использовать его по всему приложению.

Обратите внимание, что **предохранители отлавливают ошибки исключительно в своих дочерних компонентах**. Предохранитель не сможет отловить ошибку внутри самого себя. Если предохранителю не удаётся отрендерить сообщение об ошибке, то ошибка всплывает до ближайшего предохранителя, расположенного над ним в дереве компонентов. Этот аспект их поведения тоже напоминает работу блоков `catch {}` в JavaScript.

## Живой пример {#live-demo}

Посмотрите [пример объявления и использования предохранителя](https://codepen.io/gaearon/pen/wqvxGa?editors=0010).

## Где размещать предохранители {#where-to-place-error-boundaries}

Степень охвата кода предохранителями остаётся на ваше усмотрение. Например, вы можете защитить им навигационные (route) компоненты верхнего уровня, чтобы выводить пользователю сообщение «Что-то пошло не так», как это часто делают при обработке ошибок серверные фреймворки. Или вы можете охватить индивидуальными предохранителями отдельные виджеты, чтобы помешать им уронить всё приложение.

## Новое поведение при обработке неотловленных ошибок {#new-behavior-for-uncaught-errors}

Это изменение влечёт за собой существенное последствие. **Начиная с React 16, ошибки, не отловленные ни одним из предохранителей, будут приводить к размонтированию всего дерева компонентов React.**

Хотя принятие этого решения и вызвало споры, судя по нашему опыту, бо́льшим злом будет вывести некорректный UI, чем удалить его целиком. К примеру, в приложении типа Messenger, вывод поломанного UI может привести к тому, что пользователь отправит сообщение не тому адресату. Аналогично, будет хуже, если приложение для проведения платежей выведет пользователю неправильную сумму платежа, чем если оно не выведет вообще ничего.

Это изменение означает, что при миграции на React 16 вы с большой вероятностью натолкнётесь на незамеченные ранее ошибки в вашем приложении. Добавляя в ваше приложение предохранители, вы обеспечиваете лучший опыт взаимодействия с приложением при возникновении ошибок.

Например, Facebook Messenger охватывает содержимое боковой и информационной панелей, журнала и поля ввода сообщений отдельными предохранителями. Если один из этих компонентов UI упадёт, то остальные сохранят интерактивность.

Также мы призываем пользоваться сервисами обработки ошибок JavaScript (или написать собственный аналогичный сервис), чтобы вы знали и могли устранять необработанные исключения в продакшен-режиме.

## Стек вызовов компонентов {#component-stack-traces}

В режиме разработки React 16 выводит на консоль сообщения обо всех ошибках, возникших при рендеринге, даже если они никак не сказались на работе приложения. Помимо сообщения об ошибке и стека JavaScript, React 16 также выводит и стек вызовов компонентов. Теперь вы можете увидеть, где именно в дереве компонентов произошел сбой:

<img src="../images/docs/error-boundaries-stack-trace.png" style="max-width:100%" alt="Ошибка, отловленная предохранителем">

Кроме этого, в стеке вызовов компонентов выводятся имена файлов и номера строк. Такое поведение по умолчанию настроено в проектах, созданных при помощи [Create React App](https://github.com/facebookincubator/create-react-app):

<img src="../images/docs/error-boundaries-stack-trace-line-numbers.png" style="max-width:100%" alt="Ошибка, отловленная предохранителем c номерами строк">

Если вы не пользуетесь Create React App, вы можете вручную добавить к вашей конфигурации Babel [вот этот плагин](https://www.npmjs.com/package/@babel/plugin-transform-react-jsx-source). Обратите внимание, что он предназначен исключительно для режима разработки и **должен быть отключён в продакшене**.

> Примечание
>
> Имена компонентов, выводимые в их стеке вызовов, определяются свойством [`Function.name`](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Function/name). Если ваше приложение поддерживает более старые браузеры и устройства, которые могут ещё не предоставлять его нативно (например, IE 11), рассмотрите возможность включения полифилла `Function.name` в бандл вашего приложения, например [`function.name-polyfill`](https://github.com/JamesMGreene/Function.name). В качестве альтернативы, вы можете явным образом задать проп [`displayName`](/docs/react-component.html#displayname) в каждом из ваших компонентов.

## А как насчёт try/catch? {#how-about-trycatch}

`try` / `catch` — отличная конструкция, но она работает исключительно в императивном коде:

```js
try {
  showButton();
} catch (error) {
  // ...
}
```

В то время, как компоненты React являются декларативными, указывая *что* должно быть отрендерено:

```js
<Button />
```

Предохранители сохраняют декларативную природу React и ведут себя так, как вы уже привыкли ожидать от компонентов React. Например, если ошибка, произошедшая в методе `componentDidUpdate`, будет вызвана `setState` где-то в глубине дерева компонентов, она всё равно корректно всплывёт к ближайшему предохранителю.

## А что насчёт обработчиков событий? {#how-about-event-handlers}

Предохранители **не** отлавливают ошибки, произошедшие в обработчиках событий.

React не нуждается в предохранителях, чтобы корректно обработать ошибки в обработчиках событий. В отличие от метода `render` и методов жизненного цикла, обработчики событий не выполняются во время рендеринга. Таким образом, даже если они сгенерируют ошибку, React всё равно знает, что нужно выводить на экран.

Чтобы отловить ошибку в обработчике событий, пользуйтесь обычной JavaScript-конструкцией `try` / `catch`:

```js{9-13,17-20}
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = { error: null };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    try {
      // Делаем что-то, что сгенерирует ошибку
    } catch (error) {
      this.setState({ error });
    }
  }

  render() {
    if (this.state.error) {
      return <h1>Отловил ошибку.</h1>
    }
    return <button onClick={this.handleClick}>Нажми на меня</button>
  }
}
```

Обратите внимание, что приведённый выше пример демонстрирует стандартное поведение JavaScript и не использует предохранителей.

## Изменение названия метода по сравнению с React 15 {#naming-changes-from-react-15}

React 15 включал очень ограниченную поддержку предохранителей с другим названием метода: `unstable_handleError`. Этот метод больше не работает и вам нужно будет заменить его на `componentDidCatch` в своем коде, начиная с первого бета-релиза React 16.

Для этого изменения мы предоставили [codemod](https://github.com/reactjs/react-codemod#error-boundaries), обеспечивающий автоматический перенос вашего кода.
