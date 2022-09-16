# Современные Платформы Программирования (СПП). Lab №1

### Задача
Необходимо реализовать измеритель времени выполнения методов

### Затрагиваемые темы
1) Reflection.
2) Многопоточное программирование.
3) Сериализация.
4) Объектно-ориентированный дизайн.
5) Плагины.

### Организация кода
Код лабораторной работы должен состоbn из 2-ух решений (solutions):
- `LaboratoryWork_Tracer`: содержит основной код, тесты и интерфейс для создания плагинов.
  - `Tracer.Core`: основная часть библиотеки, реализующая измерение и формирование результатов, инфраструктурный код для загрузки и использования плагинов.
  - `Tracer.Core.Tests`: модульные тесты для основной части библиотеки.
  - `Tracer.Serialization.Abstractions`: содержит интерфейс ITraceResultSerializerдля использования в плагинах.
  - `Tracer.Example`: консольное приложение, демонстрирующее общий случай работы библиотеки (в многопоточном режиме при трассировке вложенных методов) и записывающее результат в файлы в соответствии с загруженными плагинами.
- `TracerSerialization`: содержит проекты с реализацией плагинов для требуемых форматов сериализации и ссылку на Tracer.Serialization.Abstractions из основного решения.
  - `TracerSerializerJSON`: сериализатор в формат данных Json
  - `TracerSerializerYAML`: сериализатор в формат данных Yaml
  - `TracerSerializerXML`: сериализатор в формат данных Xml
  - `Tracer.Serialization.Abstractions`: данный проект из основного решения нужен для использования интерфейса ITraceResultSerializer из проектов .Json, .Yaml и .Xml.
  
## Принцип Работы
### Результат работы программы в консольном режиме
![1](https://user-images.githubusercontent.com/55713244/188317781-602a888e-5f9d-441f-a803-f394e2511b78.png)
  
### Пояснение

#### Сбор данных

С самого начала, при желании пользователя отследить время выполнения методов в потоке, ему необходимо создать объект типа `MainTracer`, который и будет заниматься отслеживанием времени выполнения методов, а также соберать информацию об уже завершившихся. Этот `MainTracer` необходимо передавать при создании другого объекта в его конструктор применяя `принцип инверсии зависимостей`(Dependency Inversion). Далее, вызывая методы только что созданных объектов, `MainTracer` соберёт всю необходимую информацию об этих методах. 

#### MainTracer собирает о методах такую информацию, как:

- Имя выполненного метода
- Имя класса, в котором выполнялся метод
- Время работы метода
- Имя вызывающего его метода (Если таковой имеется)
- Номер потока, в котором выполнялся метод

#### Представление данных в удобном виде

После сбора данных, коллекция выглядит как простой набор `неупорядоченной информации`, которую, очевидно, нужно привести к `нормальному виду`. Для структурирования полученной информации, мной был придуман следующий `подход`: Если подумать логически, собирая информацию, мы также отслеживаем `имя вызывающего метода`. Если это `имя` не что иное, как `Main`, то это будет означать, что этот метод `является корневым методом`, который будет стоять на `первом уровне` в `дереве вызываемых методов в потоке` (или же будет одной из вершин в этом дереве). Далее, отслеживая все все унаследованные от этого метода другие методы, можно будет воспроизвести структуру вызовов методов в полном объёме. Также, не трудно догадаться, что `вызывающий` другие методы метод завершится позже, нежели `вызываемые`, так как без завершения `вызываемых` методов, мы просто напросто не сможем получить информацию о времени выполнения `вызывающего` их метода. Следовательно, заполнение дерева необходимо будет производить, начиная не с `начала` списка, а с его `конца`. Таким образом, `пройдя весь список`, начиная с его конца, и `отслеживая информацию о вызывающих методах`, удастся построить дерево методов. Однако, возникает ещё одна `задача`: Представить это дерево в виде `структуры` для дальнейшей работы. Для решения этой задачи я использовал `рекурсию`. Каждый метод хранит в себе `список вызываемых методов`, а вызываемые методы хранятят `списки своих вызываемых методов`. Условием `выхода из рекурсии` будет являться момент, когда мы дойдём в списке до того уровня, когда на следующем уровне структура будет не `углубляться`, а наоборот, `упрощаться`. Об этом моменте будет свидетельствовать изменение `имени класса`, из которого был вызван метод. После обнаружения такого момента, мы выходим из рекурсии, попутно передав `список методов`, которые мы подобрали на одном уровне. Таким незамысловатым способом, можно будет восстановить структуру `вызывающих` методов на уровне каждого из `потоков`. После сбора информации и прохождения по итоговому списку поиском вглубину, можно будет получить результат, как было показано на рисунке выше.

### Сериализация

#### Результат работы

> ![2](https://user-images.githubusercontent.com/55713244/188319770-d0eee386-cc3b-4caa-a4e7-45dd7e17fad6.png)

#### JSON файл

> ![3](https://user-images.githubusercontent.com/55713244/188319780-06d6e08d-9e0c-443a-be3b-63596d9b3d66.png)

#### XML файл

> ![4](https://user-images.githubusercontent.com/55713244/188319824-020247f1-7d89-4a3e-bce4-511bbfbf3fe0.png)

#### YAML файл

> ![5](https://user-images.githubusercontent.com/55713244/188319812-ba52aeb1-b5cc-44fe-9bf9-555f962d2ccc.png)

#### Пояснение 

Результат измерений представлен в трёх форматах: `JSON`, `XML` и `YAML`. При реализации плагинов были использованы `готовые библиотеки` для работы с данными форматами.
Классы для сериализации результата имеют иметь `общий интерфейс` (Который располагается в отдельном проекте) и `загружаются динамически` во время выполнения как "плагины" с помощью метода `Assembly.Load`. Результаты работ каждого из сериализаторов приведены выше. 

## Вывод

Таким образом, было разработано приложение, способное измерять время выполнения методов, выполняемых в многопоточном режиме, а также производить сериализацию полученных данных в форматы JSON, XML и YAML.