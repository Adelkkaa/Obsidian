1) Поправить ошибку с key
2) Удалить navigate из extraArgs (Каждый ререндер создается новый стор)
3) Исправить функцию логина, navigate больше нет
4) Добавить removeAfterUnmount=true в ArticlesPage
5) Добавить _inited в articlePageSlice
6) Добавить services/initArticlesPage
7) В DynamicModuleLoader проверять вмонтирован ли уже редьюсер, если да, то не добавляем снова