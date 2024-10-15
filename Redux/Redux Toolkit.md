> Redux Toolkit - это удобная библиотека для управления состоянием, использующая фундамент, заложенный [[Redux - База|Redux]]

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

## Использование Async Thunk вместо RTK Query

По-хорошему, я никогда не видел использования Async Thunk в коммерческих проектах, но если говорить просто, то в createSlice передаётся extraReducer, который обрабатывает состояния Async Thunk (pending, fulfilled, rejected)

Пример AsyncThunk:

```ts
import { createAsyncThunk } from '@reduxjs/toolkit';
import axios from 'axios';
import { User, userActions } from 'entities/User';

interface LoginByUsernameProps {
    username: string;
    password: string;
}

export const loginByUsername = createAsyncThunk<User, LoginByUsernameProps, { rejectValue: string }>(
    'login/loginByUsername',
    async (authData, thunkAPI) => {
        try {
            const response = await axios.post<User>('http://localhost:8000/login', authData);

            if (!response.data) {
                throw new Error();
            }

            localStorage.setItem(USER_LOCALSTORAGE_KEY, JSON.stringify(response.data));
            thunkAPI.dispatch(userActions.setAuthData(response.data));

            return response.data;
        } catch (e) {
            console.log(e);
            return thunkAPI.rejectWithValue('error');
        }
    },
);

```

Как видно из кода здесь мы можем при помощи dispatch вызывать любые actions
Также все взаимодействие с reject'ами происходит при помощи thunkAPI

Такие thunkApi обрабатываются в extraReducer'ах

```ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { ILoginSchema } from '../types/LoginSchema';
import { loginByUsername } from '../services/loginByUsername';

const initialState: ILoginSchema = {
    username: '',
    password: '',
    isLoading: false,
    error: undefined,
};

const loginSlice = createSlice({
    name: 'login',
    initialState,
    reducers: {
        setUsername(state, action: PayloadAction<string>) {
            state.username = action.payload;
        },
        setPassword(state, action: PayloadAction<string>) {
            state.password = action.payload;
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(loginByUsername.pending, (state) => {
                state.error = undefined;
                state.isLoading = true;
            })
            .addCase(loginByUsername.fulfilled, (state) => {
                state.isLoading = false;
            })
            .addCase(loginByUsername.rejected, (state, action) => {
                state.isLoading = false;
                state.error = action.payload;
            });
    },
});

export const { actions: loginActions } = loginSlice;
export const { reducer: loginReducer } = loginSlice;

```

Такие async thunk вызываются через dispatch
```tsx
dispatch(loginByUsername({ username: username.toString(), password: password.toString() }));
```

