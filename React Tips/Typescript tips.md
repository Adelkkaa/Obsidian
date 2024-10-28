> В данной статье находятся подсказки по ts

## TS isolated modules

При сборке билда мы используем сторонние транспиляторы кода, например babel, которые, в свою очередь, по дефолту работают с одним файлом за раз, а значит, что они не могут применять преобразования кода, которые зависят от понимания всей системы типов.

Эти ограничения могут привести к проблемам во время выполнения некоторых функций TypeScript, таких как перечисления const и пространства имен. Установка флага isolatedModules позволяет TypeScript предупреждать вас, если вы пишете определенный код, который не может быть правильно интерпретирован однофайловым процессом транспиляции.

При указании флага в tsConfig
```json
"isolatedModules": true

```
1) TS начинает ругаться, если ts файл пустой
2) Импорты типов должны строго задаваться как
```ts
import type { SomeType } from 'somewhere'
```

## Circular Dependency

![[Pasted image 20241028110728.png]]

Пример кольцевой зависимости:
Есть три файла:
1) types/user.ts
2) consts/userConsts.ts
3) index.ts

Кольцевая зависимость - это когда модуль импортируется оттуда, куда он и экспортируется
index.ts
```ts
export {
    getUserAuthData,
} from './model/selectors/getUserAuthData/getUserAuthData';

export {
    getUserInited,
} from './model/selectors/getUserInited/getUserInited';

export { isUserAdmin, isUserManager, getUserRoles } from './model/selectors/roleSelectors';

export {
    userReducer,
    userActions,
} from './model/slice/userSlice';

export type {
    UserSchema,
    User,
} from './model/types/user';
export { UserRole } from './model/consts/userConsts';

```

user.ts
```ts
import { UserRole } from '@/entities/user';

export interface User {
    id: string;
    username: string;
    avatar?: string;
    roles?: UserRole[];
}

export interface UserSchema {
    authData?: User;

    _inited: boolean;
}

```

userConsts.ts
```ts
export enum UserRole {
    ADMIN = 'ADMIN',
    USER = 'USER',
    MANAGER = 'MANAGER',
}

```

В файле user.ts образуется кольцевая зависимость, потому что в index.ts импортируется user.ts, а user.ts в свою очередь импортирует модуль из index.ts

> Детектить такие кольцевые зависимости можно при помощи плагинов сборщиков, для webpack есть circular-dependency-plugin

Для его работы необходимо его просто проинициализировать

```ts
 new CircularDependencyPlugin({
            exclude: /node_modules/,
            failOnError: true,
	}),
```

## Сборка ts при помощи babel + webpack

> Вообще, я рассматривал подобное в [[Webpack Core#19) Babel|Этой статье]], но здесь я покажу как при помощи babelLoader собрать проект на ts, а также вынести проверку типов в отдельный процесс

Объявление лоадеров:
```ts
import webpack from 'webpack';
import { BuildOptions } from './types/config';
import { buildCssLoader } from './loaders/buildCssLoader';
import { buildBabelLoader } from './loaders/buildBabelLoader';

export const buildLoaders = (options: BuildOptions): webpack.RuleSetRule[] => {
    const { isDev } = options;

    // const tsLoader = {
    //     test: /\.tsx?$/,
    //     use: 'ts-loader',
    //     exclude: /node_modules/,
    // };

    const codeBabelLoader = buildBabelLoader({ ...options, isTsx: false });
    const tsxCodeBabelLoader = buildBabelLoader({ ...options, isTsx: true });

    const cssLoader = buildCssLoader(isDev);

    const svgLoader = {
        test: /\.svg$/i,
        issuer: /\.[jt]sx?$/,
        use: ['@svgr/webpack'],
    };

    const fileLoader = {
        test: /\.(png|jpe?g|gif)$/i,
        use: [
            {
                loader: 'file-loader',
            },
        ],
    };

    return [
        fileLoader,
        svgLoader,
        codeBabelLoader,
        tsxCodeBabelLoader,
        cssLoader,
    ];
};

```

Создание babelLoader:
```ts
import { BuildOptions } from '../types/config';
import babelRemovePropsPlugin from '../../babel/babelRemovePropsPlugin';

interface BuildBabelLoaderProps extends BuildOptions {
    isTsx?: boolean;
}

export function buildBabelLoader({ isDev, isTsx }: BuildBabelLoaderProps) {
    return {
        test: isTsx ? /\.(jsx|tsx)$/ : /\.(js|ts)$/,
        exclude: /node_modules/,
        use: {
            loader: 'babel-loader',
            options: {
                presets: ['@babel/preset-env'],
                plugins: [
                    [
                        'i18next-extract',
                        {
                            locales: ['ru', 'en'],
                            keyAsDefaultValue: true,
                        },
                    ],
                    [
                        '@babel/plugin-transform-typescript',
                        {
                            isTsx,
                        },
                    ],
                    '@babel/plugin-transform-runtime',
                    isTsx && [
                        babelRemovePropsPlugin,
                        {
                            props: ['data-testid'],
                        },
                    ],
                    isDev && require.resolve('react-refresh/babel'),
                ].filter(Boolean),
            },
        },
    };
}

```

Удаление data-test-id из сборок:
```ts
import { PluginItem } from '@babel/core';

export default function (): PluginItem {
    return {
        visitor: {
            Program(path, state) {
                const forbidden = state.opts.props || [];

                path.traverse({
                    JSXIdentifier(current) {
                        const nodeName = current.node.name;

                        if (forbidden.includes(nodeName)) {
                            current.parentPath.remove();
                        }
                    },
                });
            },
        },
    };
}

```

Вынос проверку типов в отдельный процесс: (fork-ts-checker)
```ts
import HtmlWebpackPlugin from 'html-webpack-plugin';
import webpack from 'webpack';
import MiniCssExtractPlugin from 'mini-css-extract-plugin';
import ReactRefreshWebpackPlugin from '@pmmmwh/react-refresh-webpack-plugin';
import { BundleAnalyzerPlugin } from 'webpack-bundle-analyzer';
import CopyPlugin from 'copy-webpack-plugin';
import CircularDependencyPlugin from 'circular-dependency-plugin';
import ForkTsCheckerWebpackPlugin from 'fork-ts-checker-webpack-plugin';
import { BuildOptions } from './types/config';

export const buildPlugins = ({
    paths,
    isDev,
    apiUrl,
    project,
}: BuildOptions): webpack.WebpackPluginInstance[] => {
    const plugins = [
        new HtmlWebpackPlugin({
            template: paths.html,
        }),
        new MiniCssExtractPlugin({
            filename: 'css/[name].[contenthash:8].css',
            chunkFilename: 'css/[name].[contenthash:8].css',
        }),
        new webpack.ProgressPlugin(),
        new webpack.DefinePlugin({
            __IS_DEV__: JSON.stringify(isDev),
            __API__: JSON.stringify(apiUrl),
            __PROJECT__: JSON.stringify(project),
        }),
        new CopyPlugin({
            patterns: [
                { from: paths.locales, to: paths.buildLocales },
            ],
        }),
        new CircularDependencyPlugin({
            exclude: /node_modules/,
            failOnError: true,
        }),
        new ForkTsCheckerWebpackPlugin({
            typescript: {
                diagnosticOptions: {
                    semantic: true,
                    syntactic: true,
                },
                mode: 'write-references',
            },
        }),
    ];

    if (isDev) {
        plugins.push(new ReactRefreshWebpackPlugin());
        plugins.push(new webpack.HotModuleReplacementPlugin());
        plugins.push(new BundleAnalyzerPlugin({ openAnalyzer: false }));
    }
    return plugins;
};

```

