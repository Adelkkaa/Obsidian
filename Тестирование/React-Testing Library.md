> По хорошему говоря он использует JEST, поэтому сейчас я поясню как базовый конфиг jest адаптировать под проверку jsx


> Устанавливаем зависимости

```json
"@testing-library/jest-dom": version,
"@testing-library/react": version,
"@babel/preset-react": version,
"identity-obj-proxy": version
```

> Далее пробуем написать простейшний тест с кнопкой

```tsx
import { render, screen } from '@testing-library/react';

import { Button } from './Button';

  

describe('Button', () => {

	test('correct render', () => {
	
		render(<Button>TEST</Button>);
		
		expect(screen.getByText('TEST')).toBeInTheDocument();

	});

});
```

> У нас повалятся ошибки, в первую очередь стоить кастомизировать babel.config

```json
{

"presets": [

	"@babel/preset-env",
	
	"@babel/preset-typescript",
	
	[
	
		"@babel/preset-react",
		
		{
		
			"runtime": "automatic"
		
		}
	
	]

],

"plugins": ["i18next-extract"]

}
```

> Далее исправляем ошибку связанную с невозможность чтения абсолютных путей по алиасам

jest.config.ts
```ts
modulePaths: ['<rootDir>/src'],
```

> Далее учим jest работать со стилями 
```ts
moduleNameMapper: {
	'\\.s?css$': 'identity-obj-proxy',
},
```

> Чтобы Jest мог корректно строить дерево компонентов каждому тесту нужен импорт jest-dom, чтобы везде это не писать создаем файл setupTests.ts

```ts
import '@testing-library/jest-dom';
```

> И в конфиге jest.config.ts указываем свойство, которое будет подтягивать этот файл перед каждым тестом

```ts
setupFilesAfterEnv: [
	'<rootDir>/config/jest/setupTests.ts',
],
```

> Также для того, чтобы компоненты, где используются svg могли корректно считываться, необходимо дополнить правило для moduleNameWrapper

```ts
moduleNameMapper: {
	'\\.s?css$': 'identity-obj-proxy',
	'\\.svg': path.resolve(__dirname, 'jestEmptyComponent.tsx'),
},
```

jestEmptyComponent: 

```tsx
import React from 'react';

const jestEmptyComponent = function () {
	return <div />;
};

export default jestEmptyComponent;
```