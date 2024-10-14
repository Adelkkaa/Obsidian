
> Flux архитектура - это паттерн, придуманный командой Facebook, который облегчает разработку крупных проектов. [[Redux - База |Например в Redux]]

## Основные принципы Flux

1) Dispatcher
2) Stores
3) View

### Dispatcher

Диспетчер представляет собой корневое звено, которое управляет потоком данных. 
Он регестрирует хранилища и колбэки - обратные вызовы. Когда извне диспетчер получает какое-то действие, то он оповещает эти хранилища о поступившем действии

### Stores

Хранилища содержат состояния приложения и его логику.
Каждое отдельное хранилище управляет своей зоной ответственности

### View

Представления оформляют визуальную часть приложения. Суть в том, что есть какой-то корневой компонент, который содержит все остальные компоненты внутри себя, он прослушивает события, которые исходят от хранилища. Получив какое-либо событие, controller-view передает данные, полученные от хранилища, другим компонентам.

Когда controller-view получает событие от хранилища, то вначале controller-view запрашивает у хранилища все необходимые данные. Затем он вызывает свой метод setState() или forceUpdate(), который приводит к выполнению у компонента метода render(). А это в свою очередь приводит к вызову метода render() и обновлению дочерних компонентов.

### Action (Действие)

Действие представляет функцию, которая может содержать некоторые данные, которые передаются диспетчеру. Действие может быть вызвано обработчиками событий в компонентах, например, по нажатию на кнопку, либо инициатором действий может какой-нибудь другой внешний источник, например, сервер. Через диспетчер хранилище получает действие и соответствующим образом реагирует на него.

![[Pasted image 20241014194618.png]]
<center>Пример flux архитектуры</center>

