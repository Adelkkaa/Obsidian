Для простейшего объявления dev-server необходимо всего то подтянуть нужные зависимости и после этого объявить две опции
```
{
	port: options.port,
	open: true,
}
```

Сюда также можно опрокинуть адрес, который будет проксировать запросы, как с vite

Также для корректной работы необходимы source-map

> Source Map - позволяет показать в каком файле произошла ошибка, объявляется через devtool в конфиге

```
devtool: isDev ? 'inline-source-map' : undefined,
```

Также через скрипты можно опрокидывать env переменные 
```
"scripts": {

	"start": "webpack serve --env port=3033",
	
	"build:prod": "webpack --env mode=production",
	
	"build:dev": "webpack --env mode=development"

},
```

Принимать их нужно в таком формате 

```
import path from "path";

import { BuildEnv, BuildPaths } from "./config/build/types/config";

import { buildWebpackConfig } from "./config/build/buildWebpackConfig";

import webpack from "webpack";

  

const config = (env: BuildEnv): webpack.Configuration => {

	const paths: BuildPaths = {
	
	entry: path.resolve(__dirname, 'src', 'index.ts'),
	
	build: path.resolve(__dirname, 'build'),
	
	html: path.resolve(__dirname, 'public', 'index.html'),
	
	}
	
	const mode = env.mode || 'development';
	
	const isDev = mode === 'development';
	
	const PORT = env.port || 3000
	
	return buildWebpackConfig({
	
		mode: mode,
		
		paths,
		
		isDev,
		
		port: PORT
	
	})

}

  

export default config;
```

Типизация этих переменных самописная см. ниже

```
export interface BuildEnv {

	mode: BuildMode;
	
	port: number;

}
```