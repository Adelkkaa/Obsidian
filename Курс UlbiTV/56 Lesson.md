1) Дополнить db.json
2) Создать селекторы и состояние в slice для page и limit
3) Избавиться от глобального класса page-wrapper и использовать shared/ui/page
4) На всех страницах заменить корневой элемент на <Page />
5) Написать хук useInfiniteScroll, инициализировать его внутри Page [[Intersection Observer|Конспект тут]]
6) Подкорректировать ArticlesPage