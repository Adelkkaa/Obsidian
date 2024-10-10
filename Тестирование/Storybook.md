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
	
	config.resolve.modules.push(paths.src);
	
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

