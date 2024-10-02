### Установка зависимостей 

```
npm install --dev webpack webpack-cli
```


> [!NOTE] Пояснения
> webpack - основная зависимость, в которой хранится весь код, нужный для работы бандлера
> webpack-cli - обертка для запуска webpack из коммандной строки (Command Line Interface)


Следующим шагом необходимо создать простейшний конфигурационный файл

```
// path — встроенный в Node.js модуль
const path = require('path')

module.exports = {
	mode: 'development',
  // Указываем путь до входной точки:
  entry: path.resolve(__dirname, src, index.js),
  // Описываем, куда следует поместить результат работы:
  output: {
    // Путь до директории (важно использовать path.resolve):
    path: path.resolve(__dirname, 'dist'),
    // Имя файла со сборкой:
    filename: 'bundle.js'
  }
}

```


Далее в package.json добавляем скрипт для запуска:
```
{ "scripts": { "build": "webpack" } }
```

Ву-аля, наш код был собран и его можно исполнять

Для того чтобы исключить генерацию с одинаковыми именами необходимо в output-filename указывать уникальный хэш
```
output: {

filename: '[name].[contenthash].js',

path: path.resolve(__dirname, 'dist'),

},
```

Для прогресса и генерации HTML по шаблону подтягиваем плагины:

```
plugins: [

new HtmlWebpackPlugin({

template: path.resolve(__dirname, 'public', 'index.html')

}),

new webpack.ProgressPlugin()

]
```

