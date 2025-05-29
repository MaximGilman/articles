---
layout: post
title: "Ошибки в React хуках - очевидные и предсказуемые?"
date: 2025-03-30
---

Представим, что мы пишем реализацию компонента `MyComponent` на React:

```ts
export default function App() {
  return <MyComponent />;
}
```


Ниже будут представлены варианты реализаций компонента `MyComponent`, сможете определить, какие из них будут работать корректно?

Для каждого варианта будет плашка с ответом и пояснением почему происходит именно так.

<details>
<summary> Вопрос 1</summary>

```ts
const MyComponent = () => {
  const [count, setCount] = useState(0);

  return (
    <>
      <h1>State Counter: {count}</h1>
      <button onClick={() => setCount(prevCount => prevCount++)}>Increment State</button>
    </>
  );
}
```

<details > 
<summary> Ответ</summary>

**Компонент работает неправильно**
> Оператор **prevCount++** сначала возвращает текущее значение **prevCount**, а затем увеличивает его на единицу. 
> 
> В результате обновление состояния происходит с текущим значением, а не с увеличенным. То есть состояние не изменяется.
</details>
</details>


<hr>

<details> 
<summary> Вопрос 2 </summary>

```ts
const MyComponent = () => {
  const refValue = useRef(0);

  return (
    <>
      <h1>State Counter: {refValue.current}</h1>
      <button onClick={() => (refValue.current = refValue.current + 1)}>
        Increment Ref
      </button>
    </>
  );
};
```

<details > 
<summary> Ответ</summary>

**Компонент работает неправильно**
> Хук **useRef** создаёт объект, который сохраняется между рендерами, но изменение свойства **current** (**refValue.current**) не вызывает перерисовку компонента.
> 
> Когда пользователь нажимает кнопку, значение **refValue.current** обновляется, но React не перерисовывает компонент, поэтому пользователь продолжает видеть старое значение.
</details>
</details>


<hr>

<details> 
<summary> Вопрос 3</summary>

```ts
const MyComponent = () => {
  let count = 0;
  return (
    <>
      <h1>State Counter: {count}</h1>
      <button onClick={() => count++}>Increment</button>
    </>
  );
}
```


<details > 
<summary> Ответ</summary>

**Компонент работает неправильно**
> Значение переменной **count** обновляется внутри обработчика кнопки, но это изменение происходит локально в памяти, и компонент ничего об этом "не знает".
>
> Изменение значения локальной переменной не триггерит повторного рендеринга компонента, потому что React отслеживает изменения только через состояние.
</details>
</details> 

<hr>


<details > 
<summary> Вопрос 4</summary>

```ts
const MyComponent = () => {
  const [count, setCount] = useState(0);
  const [coefficient, setCoefficient] = useState(0);

  const increaseByCoefficient = () => {
    setCoefficient(coefficient + 1);
    setCount(count + coefficient);
  };

  return (
    <>
      <h1>State Counter: {count}</h1>
      <button onClick={increaseByCoefficient}>Add multiplier </button>
    </>
  );
}
```

<details > 
<summary> Ответ</summary>

**Компонент работает неправильно**
> Обновление стейта происходит асинхронно.
>
> Коэффициент в переменной **coefficient** еще не обновился, а уже используется для увеличения **count**, что приводит к увеличению счетчика на предыдущее значение коэффициента.
</details>
</details> 

<hr>

<details>
<summary> Вопрос 5</summary>

```ts
const MyComponent = () => {
  const [count, setCount] = useState(0);
  const valueRef = useRef(0);

  useEffect(() => {
    valueRef.current = count;
  }, [count]);


  return (
    <>
      <h1>State Counter: {valueRef.current}</h1>
      <button onClick={() => setCount(valueRef.current + 1)}>Increment Count</button>
    </>
  );
}
```

<details>
<summary> Ответ</summary>

**Компонент работает недетерминированно**
> Несмотря на то, что **count**/**setCount** приведут к ререндеру, в компоненте отображается значение из **valueRef.current**, которое может быть несогласованным, относительно текущего состояния **count**.
>
> Хотя **useEffect** синхронизирует **valueRef.current** с **count**, асинхронный характер обновлений состояния означает, что могут возникнуть  случаи, когда **valueRef.current** и **count** отличаются, что приводит к неожиданному поведению.
</details>
</details>


<hr>

<details>
<summary> Вопрос 6</summary>

```ts
type State = { count: number; step: number };
type Action =
  | { type: "increment" }
  | { type: "setStep"; payload: number }
  | { type: "reset" };

const counterReducer = (state: State, action: Action): State => {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + state.step };
    case "setStep":
      return { ...state, step: action.payload };
    case "reset":
      return { count: 0, step: 1 };
    default:
      throw new Error("Unknown action type");
  }
};

export const MyComponent = () => {
  const [state, dispatch] = useReducer(counterReducer, { count: 0, step: 1 });

  return (
    <div>
      <h1>State Counter: {state.count}</h1>
      <button onClick={() => dispatch({ type: "increment" })}>Increment</button>
    </div>
  );
};
```


<details>
<summary> Ответ</summary>

**Компонент работает верно**

> Несмотря на использование **useReducer** в качестве хука для отслеживания состояния - компонент работает корректно.
</details>
</details>

<hr>

<details>
<summary> Вопрос 7</summary>

```ts
const MyComponent : FC<{count: number}>= (count)  => {
  return (
    <div>
      <h1>State Counter: {count}</h1>
    </div>
  );
};
```
<details>
<summary> Ответ</summary>

**Компонент работает неправильно**

| Если вы используете линтеры или typechecker'ы - такой код даже не соберется. Ошибка формальная, но при этом малозаметная.

> Причина в том, что **props** передается как параметр функции без деструктуризации, а это не соответствует ожидаемому типу для React-компонентов.
>
> Вместо **(count) => {...}** надо **({ count }) => { ... }**
</details>
</details>

<hr>

<details>
<summary> Вопрос 8</summary>

```ts
const MyComponent = () => {
  const [count, setCount] = useState(0);

  const increment = useCallback(() => {
    setCount(count + 1); 
  }, []); 

  return (
    <div>
      <h1>State Counter: {count}</h1>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```
<details>
<summary> Ответ</summary>

**Компонент работает неправильно**

> Callback используется с пустым массивом зависимостей **[]**. Это означает, что функция увеличения запоминается один раз и никогда не обновляется в течение жизненного цикла компонента.
>
> Поскольку значение `**count** используется для задания значения внутри функции - то ее мемоизированное значение всегда ссылается на начальное значение **count**, т.е. **(0 + 1)** и не учитывает последующие обновления состояния.
</details>
</details>



### Сколько вопросов ответили верно? Можете поделиться в [комментариях](https://t.me/reboot_repeatedly/104).
