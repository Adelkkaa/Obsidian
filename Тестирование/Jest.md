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

