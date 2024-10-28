Данная либа очень сильно зависима от версионирования
Порядок работы
1) Установить и идти по доке, как и с другими либами для тестов тут есть проблемы с конфигурацией абсолютных путей, глобальных стилей и так далее

Из важного можно отметить декораторы, которые помогут тестировать компонент внутри другого комопонента, например так выглядит декоратор для глобальных стилей:

```tsx
import 'app/styles/index.scss';

import { Story } from '@storybook/react';

export const StyleDecorator = (story: () => Story) => story();
```

preview.js выглядит вот так:
```tsx
import { addDecorator } from '@storybook/react';

import { Theme } from 'app/providers/ThemeProvider';

import { RouterDecorator } from 'shared/config/storybook/RouterDecorator/RouterDecorator';

import { StyleDecorator } from 'shared/config/storybook/StyleDecorator/StyleDecorator';

import { ThemeDecorator } from 'shared/config/storybook/ThemeDecorator/ThemeDecorator';

  

export const parameters = {

actions: { argTypesRegex: '^on[A-Z].*' },

	controls: {
		
		matchers: {
		
			color: /(background|color)$/i,
			
			date: /Date$/,
		
		},
	
	},

};

  

addDecorator(StyleDecorator);

addDecorator(ThemeDecorator(Theme.LIGHT));

addDecorator(RouterDecorator);
```

Также сторибук нужно учить работать с различными лодерами, в том же месте где лежит конфигурация создается webpack.config.ts

> Очень важно отметить, что в модули нужно вставлять наш путь до src в самое начало, чтобы сборщик искал испорты в первую очередь в нашей рабочей директории, а не в  node_modules (Такое мне понадобилось при отсутствии alias в проекте, где импорты были подобны "==entities/Counter==")



```ts
import webpack, { RuleSetRule } from 'webpack';

import path from 'path';

import { buildCssLoader } from '../build/loaders/buildCssLoader';

import { BuildPaths } from '../build/types/config';

  

export default ({ config }: {config: webpack.Configuration}) => {

	const paths: BuildPaths = {
	
		build: '',
		
		html: '',
		
		entry: '',
		
		src: path.resolve(__dirname, '..', '..', 'src'),
	
	};
	
	config.resolve.modules.unshift(paths.src);
	
	config.resolve.extensions.push('.ts', '.tsx');
	
	  
	
	// eslint-disable-next-line no-param-reassign
	
	config.module.rules = config.module.rules.map((rule: RuleSetRule) => {
	
	if (/svg/.test(rule.test as string)) {
	
		return { ...rule, exclude: /\.svg$/i };
	
	}
	
	  
	
	return rule;

	});

  

	config.module.rules.push({
	
		test: /\.svg$/,
		
		use: ['@svgr/webpack'],
		
		});
	
	config.module.rules.push(buildCssLoader(true));

  

	return config;

};
```

В целом говоря декораторы создаются для различных тем, роутеров и так далее, вот пример с темами

Button.stories.tsx
```tsx
import React from 'react';

import { ComponentStory, ComponentMeta } from '@storybook/react';

  

import { Theme } from 'app/providers/ThemeProvider';

  

import { ThemeDecorator } from 'shared/config/storybook/ThemeDecorator/ThemeDecorator';

import { Button, ThemeButton } from './Button';

  

export default {

	title: 'shared/Button',
	
	component: Button,
	
	argTypes: {
	
	backgroundColor: { control: 'color' },
	
	},

} as ComponentMeta<typeof Button>;

  

const Template: ComponentStory<typeof Button> = (args) => <Button {...args} />;

  

export const Primary = Template.bind({});

	Primary.args = {
	
	children: 'Test Text',

};

  

export const Clear = Template.bind({});

Clear.args = {

	children: 'Test Text',
	
	theme: ThemeButton.CLEAR,

};

  

export const Outline = Template.bind({});

Outline.args = {

	children: 'Text',
	
	theme: ThemeButton.OUTLINE,

};

  

export const OutlineDark = Template.bind({});

OutlineDark.args = {

	children: 'Text',
	
	theme: ThemeButton.OUTLINE,

};

OutlineDark.decorators = [ThemeDecorator(Theme.DARK)];
```

Storybook обычно используют вместе с инструментом, который позволяет тестировать по стори кейсам визуальные отличия, поэтому для этого подтянем Loki
1) Устанавливаем локи
2) Запускаем докер
3) Запускаем сторибук
4) Стартуем тесты

## Адаптация под async reducers (Redux)

Для того, чтобы сторибук работал корректно необходимо в декоратор передавать асинхронные редьюсеры, которые мы должны использовать

Про асинхронные редьюсеры [[Redux Оптимизация|читай тут]]

```tsx
import { Story } from '@storybook/react';
import { StateSchema, StoreProvider } from 'app/providers/StoreProvider';
import { DeepPartial, ReducersMapObject } from '@reduxjs/toolkit';
import { loginReducer } from 'features/AuthByUsername';

const defaultAsyncReducers: DeepPartial<ReducersMapObject<StateSchema>> = {
    loginForm: loginReducer,
};

export const StoreDecorator = (
    state: DeepPartial<StateSchema>,
    asyncReducers?: DeepPartial<ReducersMapObject<StateSchema>>,
) => (StoryComponent: Story) => (
    <StoreProvider
        initialState={state}
        asyncReducers={{ ...defaultAsyncReducers, ...asyncReducers }}
    >
        <StoryComponent />
    </StoreProvider>
);

```

Как видно из примера, мы передаем и initialState, и asyncReducers

Вот компонент StoreProvider

```tsx
import { ReactNode } from 'react';
import { Provider } from 'react-redux';
import { createReduxStore } from 'app/providers/StoreProvider';
import { StateSchema } from 'app/providers/StoreProvider/config/StateSchema';
import { DeepPartial, ReducersMapObject } from '@reduxjs/toolkit';

interface StoreProviderProps {
    children?: ReactNode;
    initialState?: DeepPartial<StateSchema>;
    asyncReducers?: DeepPartial<ReducersMapObject<StateSchema>>
}

export const StoreProvider = (props: StoreProviderProps) => {
    const {
        children,
        initialState,
        asyncReducers,
    } = props;

    const store = createReduxStore(initialState as StateSchema, asyncReducers as ReducersMapObject<StateSchema>);

    return (
        <Provider store={store}>
            {children}
        </Provider>
    );
};

```

createReduxStore

```ts
import { configureStore, ReducersMapObject } from '@reduxjs/toolkit';
import { counterReducer } from 'entities/Counter';
import { userReducer } from 'entities/User';
import { StateSchema } from './StateSchema';
import { createReducerManager } from './reducerManager';

export const createReduxStore = (initialState?: StateSchema, asyncReducers?: ReducersMapObject<StateSchema>) => {
    const rootReducers: ReducersMapObject<StateSchema> = {
        ...asyncReducers,
        counter: counterReducer,
        user: userReducer,
    };

    const reducerManager = createReducerManager(rootReducers);

    const store = configureStore({
        reducer: reducerManager.reduce,
        devTools: __IS_DEV__,
        preloadedState: initialState,
    });

    // @ts-ignore
    store.reducerManager = reducerManager;

    return store;
};

```

Использование:

```tsx
import React from 'react';
import { ComponentMeta, ComponentStory } from '@storybook/react';
import { StoreDecorator } from 'shared/config/storybook/StoreDecorator/StoreDecorator';
import LoginForm from './LoginForm';

export default {
    title: 'features/LoginForm',
    component: LoginForm,
    argTypes: {
        backgroundColor: { control: 'color' },
    },
} as ComponentMeta<typeof LoginForm>;

const Template: ComponentStory<typeof LoginForm> = (args) => <LoginForm {...args} />;

export const Primary = Template.bind({});
Primary.args = {};
Primary.decorators = [StoreDecorator({
    loginForm: { username: '123', password: 'asd' },
})];

export const withError = Template.bind({});
withError.args = {};
withError.decorators = [StoreDecorator({
    loginForm: { username: '123', password: 'asd', error: 'ERROR' },
})];

export const Loading = Template.bind({});
Loading.args = {};
Loading.decorators = [StoreDecorator({
    loginForm: { isLoading: true },
})];

```


## Мокаем ответы на запросы

К примеру у нас есть такой компонент, где данные получаются с запроса:

```tsx
import { useTranslation } from 'react-i18next';
import { memo } from 'react';
import { ArticleList } from '@/entities/Article';
import { Text, TextSize } from '@/shared/ui/Text/Text';
import { classNames } from '@/shared/lib/classNames/classNames';
import { VStack } from '@/shared/ui/Stack';
import {
    useArticleRecommendationsList,
} from '../../api/aritcleRecommendationsApi';

interface ArticleRecommendationsListProps {
    className?: string;
}

export const ArticleRecommendationsList = memo((props: ArticleRecommendationsListProps) => {
    const { className } = props;
    const { t } = useTranslation();
    const { isLoading, data: articles, error } = useArticleRecommendationsList(3);

    if (isLoading || error || !articles) {
        return null;
    }

    return (
        <VStack gap="8" className={classNames('', {}, [className])}>
            <Text
                size={TextSize.L}
                title={t('Рекомендуем')}
            />
            <ArticleList
                articles={articles}
                target="_blank"
            />
        </VStack>
    );
});

```

Чтобы замокать ответ в сторибуке, необходимо установить зависимость `storybook-addon-mock`

1) Добавляем аддон
```js
module.exports = {
    stories: [
        '../../src/**/*.stories.@(js|jsx|ts|tsx)',
    ],
    addons: [
        '@storybook/addon-links',
        '@storybook/addon-essentials',
        '@storybook/addon-interactions',
        'storybook-addon-mock/register', +++
    ],
    framework: '@storybook/react',
    core: {
        builder: 'webpack5',
    },
};

```

2) Используем
```tsx
import React from 'react';
import { ComponentStory, ComponentMeta } from '@storybook/react';

import withMock from 'storybook-addon-mock';
import { StoreDecorator } from '@/shared/config/storybook/StoreDecorator/StoreDecorator';
import { Article } from '@/entities/Article';
import { ArticleRecommendationsList } from './ArticleRecommendationsList';

export default {
    title: 'features/ArticleRecommendationsList',
    component: ArticleRecommendationsList,
    argTypes: {
        backgroundColor: { control: 'color' },
    },
    decorators: [withMock],
} as ComponentMeta<typeof ArticleRecommendationsList>;

const Template: ComponentStory<typeof ArticleRecommendationsList> = (args) => <ArticleRecommendationsList {...args} />;

const article: Article = {
    id: '1',
    img: '',
    createdAt: '',
    views: 123,
    user: { id: '1', username: '123' },
    blocks: [],
    type: [],
    title: '123',
    subtitle: 'asfsa',
};

export const Normal = Template.bind({});
Normal.args = {};
Normal.decorators = [StoreDecorator({})];
Normal.parameters = {
    mockData: [
        {
            url: `${__API__}/articles?_limit=3`,
            method: 'GET',
            status: 200,
            response: [
                { ...article, id: '1' },
                { ...article, id: '2' },
                { ...article, id: '3' },
            ],
        },
    ],
};

```

> Главное, чтобы адрес совпадал, включая все query параметры

