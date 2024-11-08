Cypress можно использовать для создания e2e тестов

1) Установка зависимостей `npm install cypress --save-dev`
2) Запуск:  `cypress open`
3) Конфигурируем в веб интерфейсе создание e2e тестов

### Конфигурация

`cypress.config.ts`
```ts
import { defineConfig } from 'cypress';

export default defineConfig({
    e2e: {
        setupNodeEvents(on, config) {
            // implement node event listeners here
        },
        baseUrl: 'http://localhost:5173/',
    },
});

```

При помощи baseUrl мы определяем базовый адрес, где запускается наше приложение, для того чтобы при том же роутинге не переходить по полным адресам

Простейший тест на переход по страницам (`cypress/e2e/common/routing.ts`):
```ts
import { selectByTestId } from '../../helpers/selectByTestId';

describe('Роутинг', () => {
    describe('Пользователь НЕ авторизован', () => {
        it('Переход на главную страницу', () => {
            cy.visit('/');
            cy.get(selectByTestId('MainPage')).should('exist');
        });
        it('Переход открывает страницу профиля', () => {
            cy.visit('/profile/1');
            cy.get(selectByTestId('MainPage')).should('exist');
        });
        it('Переход открывает несуществующий маршрут ', () => {
            cy.visit('/fasfasfasf');
            cy.get(selectByTestId('NotFoundPage')).should('exist');
        });
    });
    describe('Пользователь авторизован', () => {
        beforeEach(() => {
            cy.login();
        });
        it('Переход открывает страницу профиля', () => {
            cy.visit('/profile/1');
            cy.get(selectByTestId('ProfilePage')).should('exist');
        });

        it('Переход открывает страницу со списком статей', () => {
            cy.visit('/articles');
            cy.get(selectByTestId('ArticlesPage')).should('exist');
        });
    });
});

```

Как видно из примера тесты производятся на два сценария:
1) Пользователь не авторизован
2) Пользователь авторизован 

Для авторизации пользователя используется `cy.login()` - Это не дефолтная функция, данная функция определяется в commands и делается это следующим образом:

У нас есть директория supports, где есть файлы `e2e.ts` и `commands.ts`
Для удобства создаем директорию `commands/`, куда помещается наша команда
`cypress/supports/commands/login.ts`
```ts
import { USER_LOCALSTORAGE_KEY } from '../../../src/shared/const/localstorage';

export const login = (username: string = 'testuser', password: string = '123') => {
    cy.request({
        method: 'POST',
        url: 'http://localhost:8000/login',
        body: {
            username,
            password,
        },
    }).then(({ body }) => {
        window.localStorage.setItem(USER_LOCALSTORAGE_KEY, JSON.stringify(body));
    });
};

```

В `cypress/supports/commands.ts` добавляется команда на логин, также для ts декларируем глобально нашу новую функцию
```ts
import { login } from './commands/login';

Cypress.Commands.add('login', login);

declare global {
  namespace Cypress {
    interface Chainable {
      login(email?: string, password?: string): Chainable<void>
    }
  }
}

export {};
```

Также для правильной работы typescript в директории cypress создается `tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["es5", "dom"],
    "types": ["cypress", "node"],
    "allowJs": true,
    "isolatedModules": false
  },
  "extends": "../tsconfig.json",
  "include": ["**/*.ts"]
}

```

> Ну и конечно же стоит упомянуть, что для того чтобы тесты работали, проект должен быть запущен по адресу, указанном в baseUrl

