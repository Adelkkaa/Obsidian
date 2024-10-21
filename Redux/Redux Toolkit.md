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


### createSelector

Также в RTK можно создать селектор, который будет отдавать данные на основе данных с других селекторов

Пример:
```ts
import { createSelector } from '@reduxjs/toolkit';
import { getUserAuthData } from 'entities/User';
import { RoutePath } from 'shared/config/routeConfig/routeConfig';
import MainIcon from 'shared/assets/icons/main-20-20.svg';
import AboutIcon from 'shared/assets/icons/about-20-20.svg';
import ProfileIcon from 'shared/assets/icons/profile-20-20.svg';
import ArticleIcon from 'shared/assets/icons/article-20-20.svg';
import { SidebarItemType } from '../types/sidebar';

export const getSidebarItems = createSelector(
    getUserAuthData,
    (userData) => {
        const sidebarItemsList: SidebarItemType[] = [
            {
                path: RoutePath.main,
                Icon: MainIcon,
                text: 'Главная',
            },
            {
                path: RoutePath.about,
                Icon: AboutIcon,
                text: 'О сайте',
            },
        ];

        if (userData) {
            sidebarItemsList.push(
                {
                    path: RoutePath.profile + userData.id,
                    Icon: ProfileIcon,
                    text: 'Профиль',
                    authOnly: true,
                },
                {
                    path: RoutePath.articles,
                    Icon: ArticleIcon,
                    text: 'Статьи',
                    authOnly: true,
                },
            );
        }

        return sidebarItemsList;
    },
);

```

getUserAuthData: 
```ts
import { StateSchema } from 'app/providers/StoreProvider';

export const getUserAuthData = (state: StateSchema) => state.user.authData;

```

createSelector принимает любое количество селекторов в качестве аргументов, а также последним аргументом принимает callback, аргументы которого - это результаты получения данных из вышеперечисленных слекторов (предыдущих аргументов)
### Использования extra в async thunk

В async thunk можно передать различные аргументы в экстра, например baseApi, функцию navigate и так далее. ==Главное передавать так, чтобы создание стора не происходило после каждого изменения в зависимостях==

Чтобы это сделать необходимо при конфигурации стора указать middleware

```ts
import { configureStore, ReducersMapObject } from '@reduxjs/toolkit';
import { counterReducer } from 'entities/Counter';
import { userReducer } from 'entities/User';
import { $api } from 'shared/api/api';
import { NavigateOptions, To } from 'react-router-dom';
import { StateSchema, ThunkExtraArg } from './StateSchema';
import { createReducerManager } from './reducerManager';

export const createReduxStore = (
    initialState?: StateSchema,
    asyncReducers?: ReducersMapObject<StateSchema>,
    navigate?: (to: To, options?: NavigateOptions) => void,
) => {
    const rootReducers: ReducersMapObject<StateSchema> = {
        ...asyncReducers,
        counter: counterReducer,
        user: userReducer,
    };

    const reducerManager = createReducerManager(rootReducers);

    const extraArg: ThunkExtraArg = {
        api: $api,
        navigate,
    };

    const store = configureStore({
        reducer: reducerManager.reduce,
        devTools: __IS_DEV__,
        preloadedState: initialState,
        middleware: (getDefaultMiddleware) => getDefaultMiddleware({
            thunk: {
                extraArgument: extraArg,
            },
        }),
    });

    // @ts-ignore
    store.reducerManager = reducerManager;

    return store;
};

export type AppDispatch = ReturnType<typeof createReduxStore>['dispatch'];

```

Вот так происходит использование 
```ts
import { createAsyncThunk } from '@reduxjs/toolkit';
import { User, userActions } from 'entities/User';
import { USER_LOCALSTORAGE_KEY } from 'shared/const/localstorage';
import { ThunkConfig } from 'app/providers/StoreProvider';

interface LoginByUsernameProps {
    username: string;
    password: string;
}

export const loginByUsername = createAsyncThunk<
    User,
    LoginByUsernameProps,
    ThunkConfig<string>
>(
    'login/loginByUsername',
    async (authData, thunkApi) => {
        const { extra, dispatch, rejectWithValue } = thunkApi;

        try {
            const response = await extra.api.post<User>('/login', authData);

            if (!response.data) {
                throw new Error();
            }

            localStorage.setItem(USER_LOCALSTORAGE_KEY, JSON.stringify(response.data));
            dispatch(userActions.setAuthData(response.data));
            extra.navigate('/profile');
            return response.data;
        } catch (e) {
            console.log(e);
            return rejectWithValue('error');
        }
    },
);

```

Вкратце объясню типы в generic
1 Тип - Возвращаемое значение
2 Тип - Передаваемое значение
3 Тип - Который содержит ThunkApiConfig

В данном случае ThunkApiConfig: 
```ts
export interface ThunkConfig<T> {
    rejectValue: T;
    extra: ThunkExtraArg;
}
```

### Нормализация данных и createEntityAdapter

> Нормализация данных нужна для оптимизации производительности работы нашего приложения, она помогает избежать дублирования данных. Зачастую нормализация позволяет также производить изменение, удаление какого либо элемента в данных за линейную сложность O(n), ибо каждый элемент можно найти по ключу.

Пример не нормализованных данных:
```ts
const blogPosts = [
  {
    id: 'post1',
    author: { username: 'user1', name: 'User 1' },
    body: '......',
    comments: [
      {
        id: 'comment1',
        author: { username: 'user2', name: 'User 2' },
        comment: '.....'
      },
      {
        id: 'comment2',
        author: { username: 'user3', name: 'User 3' },
        comment: '.....'
      }
    ]
  },
  {
    id: 'post2',
    author: { username: 'user2', name: 'User 2' },
    body: '......',
    comments: [
      {
        id: 'comment3',
        author: { username: 'user3', name: 'User 3' },
        comment: '.....'
      },
      {
        id: 'comment4',
        author: { username: 'user1', name: 'User 1' },
        comment: '.....'
      },
      {
        id: 'comment5',
        author: { username: 'user3', name: 'User 3' },
        comment: '.....'
      }
    ]
  }
  // and repeat many times
]
```

Пример нормализованных данных
```ts
{
    posts : {
        byId : {
            "post1" : {
                id : "post1",
                author : "user1",
                body : "......",
                comments : ["comment1", "comment2"]
            },
            "post2" : {
                id : "post2",
                author : "user2",
                body : "......",
                comments : ["comment3", "comment4", "comment5"]
            }
        },
        allIds : ["post1", "post2"]
    },
    comments : {
        byId : {
            "comment1" : {
                id : "comment1",
                author : "user2",
                comment : ".....",
            },
            "comment2" : {
                id : "comment2",
                author : "user3",
                comment : ".....",
            },
            "comment3" : {
                id : "comment3",
                author : "user3",
                comment : ".....",
            },
            "comment4" : {
                id : "comment4",
                author : "user1",
                comment : ".....",
            },
            "comment5" : {
                id : "comment5",
                author : "user3",
                comment : ".....",
            },
        },
        allIds : ["comment1", "comment2", "comment3", "comment4", "comment5"]
    },
    users : {
        byId : {
            "user1" : {
                username : "user1",
                name : "User 1",
            },
            "user2" : {
                username : "user2",
                name : "User 2",
            },
            "user3" : {
                username : "user3",
                name : "User 3",
            }
        },
        allIds : ["user1", "user2", "user3"]
    }
}
```
В нормализации данных может помочь createEntityAdapter, который выглядит следующим образом 
```ts
import {
    createEntityAdapter,
    createSlice, PayloadAction,
} from '@reduxjs/toolkit';

import { StateSchema } from 'app/providers/StoreProvider';
import {
    fetchCommentsByArticleId,
} from 'pages/ArticleDetailsPage/model/services/fetchCommentsByArticleId/fetchCommentsByArticleId';

export interface Comment {
    id: string;
    user: User;
    text: string;
}

export interface ArticleDetailsCommentsSchema extends EntityState<Comment>{
    isLoading?: boolean;
    error?: string;
}

const commentsAdapter = createEntityAdapter<Comment>({
    selectId: (comment) => comment.id,
});

export const getArticleComments = commentsAdapter.getSelectors<StateSchema>(
    (state) => state.articleDetailsComments || commentsAdapter.getInitialState(),
);

const articleDetailsCommentsSlice = createSlice({
    name: 'articleDetailsCommentsSlice',
    initialState: commentsAdapter.getInitialState<ArticleDetailsCommentsSchema>({
        isLoading: false,
        error: undefined,
        ids: [],
        entities: {},
    }),
    reducers: {},
    extraReducers: (builder) => {
        builder
            .addCase(fetchCommentsByArticleId.pending, (state) => {
                state.error = undefined;
                state.isLoading = true;
            })
            .addCase(fetchCommentsByArticleId.fulfilled, (
                state,
                action: PayloadAction<Comment[]>,
            ) => {
                state.isLoading = false;
                commentsAdapter.setAll(state, action.payload);
            })
            .addCase(fetchCommentsByArticleId.rejected, (state, action) => {
                state.isLoading = false;
                state.error = action.payload;
            });
    },
});

export const { reducer: articleDetailsCommentsReducer } = articleDetailsCommentsSlice;

```

Данные в таком случае выглядят следующим образом:
```ts
articleDetailsComments: {
    ids: [
      '1',
      '2',
      '3'
    ],
    entities: {
      '1': {
        id: '1',
        text: 'some comment',
        articleId: '1',
        userId: '1',
        user: {
          id: '1',
          username: 'admin',
          password: '123',
          role: 'ADMIN',
          avatar: 'https://cdn.britannica.com/95/196495-050-4790CC60/The-Bozo-Show-the-Clown-1960.jpg'
        }
      },
      '2': {
        id: '2',
        text: 'some comment 2',
        articleId: '1',
        userId: '1',
        user: {
          id: '1',
          username: 'admin',
          password: '123',
          role: 'ADMIN',
          avatar: 'https://cdn.britannica.com/95/196495-050-4790CC60/The-Bozo-Show-the-Clown-1960.jpg'
        }
      },
      '3': {
        id: '3',
        text: 'some comment 3',
        articleId: '1',
        userId: '1',
        user: {
          id: '1',
          username: 'admin',
          password: '123',
          role: 'ADMIN',
          avatar: 'https://cdn.britannica.com/95/196495-050-4790CC60/The-Bozo-Show-the-Clown-1960.jpg'
        }
      }
    },
    isLoading: false
  }
```