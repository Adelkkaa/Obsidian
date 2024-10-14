> Redux Toolkit - это удобная библиотека для урпавления состоянием, использующая фундамент, заложенный [[Redux - База|Redux]]

Redux Toolkit упрощает жизнь:
Во первых, в отличие от redux, в rtk не нужно делать функции чистыми, rtk сделает это за вас под капотом
Во вторых, rtk содержит в себе rtk query, который очень сильно облегчает жизнь при написании api

### Порядок разворачивания

1) Для начала необходимо сконфигурировать store

```ts
import { configureStore } from '@reduxjs/toolkit'
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux'

export const store = configureStore({
  reducer: {
    anyReducer,
    [baseApi.reducerPath]: baseApi.reducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: false,
    }).concat([
      baseApi.middleware,
    ]),
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch

export const useAppDispatch = () => useDispatch<AppDispatch>()
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector

```

Все типы снизу помогут обращаться уже к типизированным значениям, тобишь значения достаем из useAppSelector, а не из useSelector 

2) Конфигурация reducer'a

```ts
import type { PayloadAction } from '@reduxjs/toolkit'
import { createSlice } from '@reduxjs/toolkit'
import type { RootState } from '@app/store.ts'

interface AuthState {
  user: null
  access: string | null
  refresh: string | null
}

const initialState: AuthState = {
  user: null,
  access: localStorage.getItem('access') || null,
  refresh: localStorage.getItem('refresh') || null,
}

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    setCredentials: (
      state,
      { payload: { access, refresh } }: PayloadAction<{ access: string; refresh: string }>,
    ) => {
      localStorage.setItem('access', access)
      localStorage.setItem('refresh', refresh)

      state.access = access
      state.refresh = refresh
    },
    refreshToken: (state, { payload: { access } }: PayloadAction<{ access: string }>) => {
      localStorage.setItem('access', access)
      state.access = access
    },
    logOut: () => initialState,
  },
})

export const authActions = authSlice.actions

export const auth = authSlice.reducer
export const selectCurrentToken = (state: RootState) => state.auth.access

```

Выше приведена базовая настройка простейшего редьюсера

3) Далее необходимо обернуть всё приложение в Provider

```tsx
<Provider store={store}>
	{children}
</Provider>
```

4) Использование
```tsx
  const { access } = useAppSelector((state) => state.auth)
  const { setCredentials } = authActions

  const dispatch = useAppDispatch()

  dispatch(setCredentials({access: 'hello', refresh: 'bye'}))
```

5) Конфигурация Rtk Query

Настройка baseApi: 
```tsx
import { createApi } from '@reduxjs/toolkit/query/react'
import { baseQuery } from '../baseQuery/baseQuery'

export const baseApi = createApi({
  reducerPath: 'baseApi',
  baseQuery,
  endpoints: () => ({}),
})

```

Настройка baseQuery: 
```tsx
import { fetchBaseQuery } from '@reduxjs/toolkit/dist/query/react'

export const baseQuery = fetchBaseQuery({
  baseUrl: '/api/v1/',
  credentials: 'include',
})

```

baseUrl такой, потому что распределением url адресов должен заниматься прокси сервер)


