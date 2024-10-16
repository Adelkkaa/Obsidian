> Jest предназначен для покрытия кода unit тестами (в основном)

### Начало работы с Jest

Для начала просто идём по доке и инициализируем нашу тестовую среду командой jest --init

Далее подтягивается конфиг файл для jest'a
Вот минимальный конфиг, который вынесен в отдельную папку:
```json
export default {

clearMocks: true,

testEnvironment: 'jsdom',

coveragePathIgnorePatterns: [

'\\\\node_modules\\\\',

],

moduleFileExtensions: [

'js',

'jsx',

'ts',

'tsx',

'json',

'node',

],

moduleDirectories: [

'node_modules',

],

testMatch: [

// Обнаружил разницу между МАК ОС и ВИНДОУС!!!

'<rootDir>src/**/*(*.)@(spec|test).[tj]s?(x)',

],

rootDir: '../../',

};
```

Запускается командой 
```json
"scripts": {
	"unit": "jest --config ./config/jest/jest.config.ts"
},
```

Чтобы eslint не ругался на использование внутренних функций jest'a необходимо в конфиге прописать 
```ts
env: {
	browser: true,
	es2021: true,
	jest: true,
},
```

### Простейший пример тестов

Тестируем функцию classNames

```ts
type Mods = Record<string, boolean | string>;

  

export function classNames(

cls: string,

mods: Mods = {},

additional: string[] = [],

): string {

return [

	cls,
	
	...additional.filter(Boolean),
	
	...Object.entries(mods)
	
	.filter(([className, value]) => Boolean(value))
	
	.map(([className]) => className),
	
	].join(' ');

}
```

Тесты:

```ts
import { classNames } from './classNames';

  
describe('classNames', () => {

	test('with only first param', () => {
	
		expect(classNames('someClass')).toBe('someClass');
	
	});
	
	  
	
	test('with additional class', () => {
	
		const expected = 'someClass class1 class2';
		
		expect(classNames('someClass', {}, ['class1', 'class2']))
		
		.toBe(expected);
	
	});
	
	  
	
	test('with mods', () => {
	
		const expected = 'someClass class1 class2 hovered scrollable';
		
		expect(classNames(
		
		'someClass',
		
		{ hovered: true, scrollable: true },
		
		['class1', 'class2'],
		
		)).toBe(expected);

	});

  

	test('with mods false', () => {
	
		const expected = 'someClass class1 class2 hovered';
		
		expect(classNames(
		
		'someClass',
		
		{ hovered: true, scrollable: false },
		
		['class1', 'class2'],
		
		)).toBe(expected);
	
	});

  

	test('with mods undefined', () => {
	
		const expected = 'someClass class1 class2 hovered';
		
		expect(classNames(
		
		'someClass',
		
		{ hovered: true, scrollable: undefined },
		
		['class1', 'class2'],
		
		)).toBe(expected);
	
	});

});
```

> Expect - это основной инструмент для написания тестов в Jest. Он предоставляет функции для проверки ожидаемого поведения кода.

> Describe - это функция, которая позволяет группировать набор тестов

Основные методы для сравнения результатов:
1. `toBe()`: Проверяет равенство значений.
2. `toEqual()`: Проверяет глубокое равенство объектов или массивов.
3. `toBeTruthy()` и `toBeFalsy()`: Проверяет логическое значение.
4. `toBeNull()` и `toBeUndefined()`: Проверяет на null или undefined.
5. `toBeDefined()`: Проверяет, что значение не undefined.
6. `isNaN()`: Проверяет, является ли значение NaN.
7. `toHaveLength()`: Проверяет длину строки или массива.
8. `toMatch()`: Проверяет соответствие регулярному выражению.
9. `toContain()`: Проверяет наличие элемента в массиве.
10. `toThrow()`: Проверяет, что вызванный функцию выбросила ошибку.

## Тестирование Redux Toolkit

В целом говоря тестируют обычно 
1) selector's
2) slice
3) async thunk

Пример slice:

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

Нужно декларировать тип DeepPartial

```ts
type DeepPartial<T> = T extends object ? {
    [P in keyof T]?: DeepPartial<T[P]>;
} : T;
```

### Тестирование Selector's

Любой редьюсер в slice чистая функция, которая принимает state, action: PayloadAction


Можно вынести установки имени пользователя в отдельный селектор 

```ts
import { StateSchema } from 'app/providers/StoreProvider';

export const getLoginPassword = (state: StateSchema) => state?.loginForm?.password || '';
```

StateSchema - это тип всех редьюсеров, содержащихся в хранилище 
```ts
export interface StateSchema {
    counter: CounterSchema;
    user: UserSchema;
    // Асинхронные редюсеры
    loginForm?: ILoginSchema;
}
```

Для того, чтобы протестировать селектор - необходимо замокать стейт

```ts
import { StateSchema } from 'app/providers/StoreProvider';
import { getLoginPassword } from './getLoginPassword';

describe('getLoginPassword.test', () => {
    test('should return isLoading', () => {
        const state: DeepPartial<StateSchema> = {
            loginForm: {
                password: 'hello',
            },
        };

        expect(getLoginPassword(state as StateSchema)).toBe('hello');
    });
    test('should return empty state', () => {
        const state: DeepPartial<StateSchema> = {};

        expect(getLoginPassword(state as StateSchema)).toBe('');
    });
});

```

> DeepPartial здесь нужен для того, чтобы тестировать конкретное значение, иначе придётся каждый раз прописывать весь state

### Тестирование slice

Очевидно, что тестирование slice практически идентично тестированию selector'a

Ниже приведен пример тестирования вышеуказанного slice

```ts
import { ILoginSchema } from '../types/LoginSchema';
import { loginActions, loginReducer } from './loginSlice';

describe('loginSlice', () => {
    test('setUsername', () => {
        const state: ILoginSchema = {
            isLoading: false,
            error: undefined,
            username: '',
            password: '',
        };
        expect(loginReducer(state, loginActions.setUsername('user')))
            .toEqual({
                isLoading: false,
                error: undefined,
                username: 'user',
                password: '',
            });
    });

    test('setPassword', () => {
        const state: ILoginSchema = {
            isLoading: false,
            error: undefined,
            username: '',
            password: '',
        };
        expect(loginReducer(state, loginActions.setPassword('password')))
            .toEqual({
                isLoading: false,
                error: undefined,
                username: '',
                password: 'password',
            });
    });
});

```

Его можно передавать также через DeepPartial, на выходе получим тесты следующего вида

```ts
import { ILoginSchema } from '../types/LoginSchema';
import { loginActions, loginReducer } from './loginSlice';

describe('loginSlice', () => {
    test('setUsername', () => {
        const state: DeepPartial<ILoginSchema> = {
            username: '',
        };
        expect(loginReducer(state as ILoginSchema, loginActions.setUsername('user')))
            .toEqual({
                username: 'user',
            });
    });

    test('setPassword', () => {
        const state: DeepPartial<ILoginSchema> = {
            password: '',
        };
        expect(loginReducer(state as ILoginSchema, loginActions.setPassword('password')))
            .toEqual({
                password: 'password',
            });
    });
});

```


### Тестирование async thunk

Для того, чтобы протестировать async thunk для начала необходимо замокать axios

```ts

jest.mock('axios');

const mockedAxios = jest.mocked(axios, true);
```

Далее необходимо написать класс, который будет вызывать наш action

```ts
import { StateSchema } from 'app/providers/StoreProvider';
import { AsyncThunkAction } from '@reduxjs/toolkit';

type ActionCreatorType<Return, Arg, RejectedValue>
    = (arg: Arg) => AsyncThunkAction<Return, Arg, { rejectValue: RejectedValue }>;

export class TestAsyncThunk<Return, Arg, RejectedValue> {
    dispatch: jest.MockedFn<any>;

    getState: () => StateSchema;

    actionCreator: ActionCreatorType<Return, Arg, RejectedValue>;

    constructor(actionCreator: ActionCreatorType<Return, Arg, RejectedValue>) {
        this.actionCreator = actionCreator;
        this.dispatch = jest.fn();
        this.getState = jest.fn();
    }

    async callThunk(arg: Arg) {
        const action = this.actionCreator(arg);
        const result = await action(this.dispatch, this.getState, undefined);

        return result;
    }
}

```

Как видно, при вызове async thunk вызывается action, где первым аргументом передаётся замоканный dispatch, вторым замоканный getState, третим extra, который undefined

Вот пример тестируемого async thunk

```ts
import { createAsyncThunk } from '@reduxjs/toolkit';
import axios from 'axios';
import { User, userActions } from 'entities/User';
import { USER_LOCALSTORAGE_KEY } from 'shared/const/localstorage';

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

Тест выглядит следующим образом 
```ts
import axios from 'axios';
import { userActions } from 'entities/User';
import { TestAsyncThunk } from 'shared/lib/tests/TestAsyncThunk/TestAsyncThunk';
import { loginByUsername } from './loginByUsername';

jest.mock('axios');

const mockedAxios = jest.mocked(axios, true);

describe('loginByUsername.test', () => {
    test('success login', async () => {
        const userValue = { username: '123', id: '1' };
        mockedAxios.post.mockReturnValue(Promise.resolve({ data: userValue }));

        const thunk = new TestAsyncThunk(loginByUsername);
        const result = await thunk.callThunk({ username: '123', password: '123' });

        expect(thunk.dispatch).toHaveBeenCalledWith(userActions.setAuthData(userValue));
        expect(thunk.dispatch).toHaveBeenCalledTimes(3);
        expect(mockedAxios.post).toHaveBeenCalled();
        expect(result.meta.requestStatus).toBe('fulfilled');
        expect(result.payload).toEqual(userValue);
    });

    test('error login', async () => {
        mockedAxios.post.mockReturnValue(Promise.resolve({ status: 403 }));
        const thunk = new TestAsyncThunk(loginByUsername);
        const result = await thunk.callThunk({ username: '123', password: '123' });

        expect(thunk.dispatch).toHaveBeenCalledTimes(2);
        expect(mockedAxios.post).toHaveBeenCalled();
        expect(result.meta.requestStatus).toBe('rejected');
        expect(result.payload).toBe('error');
    });
});

```

В async thunk обычно тестируют количество вызовов dispatch, статусы, payload

В удачном случае dispatch вызывается 3 раза
1 - сам вызов redux-thunk
2 - вызов thunkAPI.dispatch
3 - вызов dispatch, когда происходит return, когда статус становится fulfiled

При неудачном запросе dispatch вызывается 2 раза:
1 - Сам вызов redux-thunk
2 - вызов   return thunkAPI.rejectWithValue('error');

