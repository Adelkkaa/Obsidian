1) Идем по доке и подтягиваем все нужные зависимости
2) Инициализируем tsconfig.json
```
{
  "compilerOptions": {
    "outDir": "./dist/",
    // Подсвечивает все места где не указан тип
    // Однако это не означает, что теперь вовсе нельзя использовать any.
    // Это означает лишь, что в подобных ситуация разработчик должен явно писать any,
    "noImplicitAny": true,
    "module": "es6",
    // В какую спецификацию компилируем: для поддержки большинства браузеров
    "target": "es5",
    "jsx": "react",
    // Компилятор будет обрабатывать не только TS файлы, но и JS файлы
    "allowJs": true,
    "moduleResolution": "node",
    // esModuleInterop позволяет работать с пакетами, которые используют
    // common js как с обычными пакета с помощью import (require() module.export) = common js
    // import Moment from 'moment';
    // без флага esModuleInterop результат undefined
    // console.log(Moment);
    // c флагом результат [object Object]
    // console.log(Moment);
    "esModuleInterop": true,
    // Если какая-либо библиотека не имеет default import,
    // лоадеры вроде ts-loader или babel-loader автоматически создают их
    // вместо такого импорта
    // import * as React from 'react';
    // можно писать такой
    // import React from 'react';
    "allowSyntheticDefaultImports": true
  },
  "ts-node": {
    "compilerOptions": {
      "module": "CommonJS"
    }
  }
}
```

Переводим:

```
import path from "path";

import webpack from "webpack";

import HtmlWebpackPlugin from "html-webpack-plugin";

  

const config = {

mode: "development",

entry: path.resolve(__dirname, "src", "index.ts"),

output: {

filename: "[name].[contenthash].js",

path: path.resolve(__dirname, "dist"),

clean: true,

},

module: {

rules: [

{

test: /\.tsx?$/,

use: "ts-loader",

exclude: /node_modules/,

},

],

},

resolve: {

extensions: [".tsx", ".ts", ".js"],

},

plugins: [

new HtmlWebpackPlugin({

template: path.resolve(__dirname, "public", "index.html"),

}),

new webpack.ProgressPlugin(),

],

};

  

export default config;
```


