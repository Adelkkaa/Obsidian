
> Redux - это библиотека, предназначенная для упраления состояним в javascript

Redux вдоновлялся [[Flux |flux архитектурой]], поэтому ключевую роль здесь играют чистые функции - reducer's

Основными состовляющими redux являются 
1) Хранилище (store) - Источник данных
2) Dispatcher - Диспетчер, который сигнализирует о вызове каких-либо методов
3) Reducer's - функции, которые определяют как изменяется состояние приложения в ответ на дейтсвия. Они принимают текущее состояние и действие, возвращают новое состояние
4) Action Creators - упрощение для чистого redux, которое позволяет вызвать нужное действие 

> Reducer - чистая функция, которая всегда возвращает новые данные, не мутируя старые

Простейший пример реализации самописного redux

```js
class Store {
  constructor(reducer, initialState) {
    this.state = initialState;
    this.listeners = [];
    this.reducer = reducer;
  }

  getState() {
    return this.state;
  }

  dispatch(action) {
    const newState = this.reducer(this.state, action);
    this.state = newState;
    this.notify();
  }

  subscribe(listener) {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter(sub => sub !== listener);
    };
  }

  notify() {
    this.listeners.forEach(listener => listener());
  }
}

function createStore(reducer, initialState) {
  return new Store(reducer, initialState);
}
```


```js
// Определение действия
const INCREMENT = 'INCREMENT';

// Редьюсер
function counterReducer(state = { count: 0 }, action) {
  switch (action.type) {
    case INCREMENT:
      return { ...state, count: state.count + 1 };
    default:
      return state;
  }
}

// Создание хранилища
const store = createStore(counterReducer);

// Подписка на изменения состояния
const unsubscribe = store.subscribe(() => {
  console.log('State changed:', store.getState().count);
});

// Действие
store.dispatch({ type: INCREMENT });

// Отмена подписки
unsubscribe();

// Получение состояния
console.log(store.getState()); // { count: 1 }
```
Если при конфигурации стора была передано начальное значение, то оно заполняется, иначе же undefined, при вызове какого-либо действия вызывается reducer с заполненным начальным состоянием.

Конечно же, это не весь redux и в react'e он реализован не так просто, но концептуально понятно каким образом работает это хранилище данных








