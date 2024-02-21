# ADR7 - Системное проектирование WellDriver

## Решение ПРИНЯТО

## Контекст

* Требования к системе WellDiver прописаны в документе [WellDriver](../WellDriver.md)
* Контекстная диаграмма была проработана в [ADR1](ADR1.md)
* Выбор архитектурного подхода был разобран в [ADR2](ADR2.md)
* Компонентная схема системы была принята в [ADR3](ADR3.md)
* По оценке модифицируемости была принята новая схема компонентов по доменным областям в [ADR5](ADR5.md)
* В [ADR6](ADR6.md) были проработаны варианты взаимодействия сервисов, а также приняты решения, улучшающие качество архитектуры по ключевым характеристикам
* Необходимо проработать системные вопросы, касающиеся упаковки сервисов в контейнеры, способов хранения данных в БД, а также вопросы развертывания системы WellDriver на многих удаленных объектах 


## Диаграмма контейнеров

![Диаграмма контейнеров](ADR7_data/containers_diagram.png)

В процессе предварительного архитектурного анализа было выделено несколько контейнеров для системы WellDriver:
* Внутренние сервисы Frontend - контейнер, содержащий компоненты frontend:
	* wd-frontend - основной компонент frontend
	* wd-charts-frontend - компонент frontend, реализующий построение графиков
	* wd-reports-frontend - компонент frontend, реализующий вывод отчетов
* Внутренние сервисы Backend - контейнер, содержащий компоненты backend:
	* wd-backend - основной сервис backend
	* wd-aurh - сервис аутентификации и авторизации
	* wd-charts-backend - backend для реализации графиков
	* wd-reports-backend - backend для реализации отчетов
	* wd-validator - сервис валидации
	* operation-definition - сервис определения операций
	* file-service - файловый сервис
	* wd-calculator - расчетный сервис
	* wd-collector - сервис записи в БД
	* wd-transmitter - сервис передачи данных в облако
* Node OPC - контейнер микросервиса node-opc для получения данных от различных автоматических источников по разным протоколам
* Контейнеры поддерживающих сервисов:
	* nginx - web-сервер и прокси сервер
	* postgres - сервер базы данных PostgreSQL
	* mongodb - сервер базы данных Mongo DB
	* redis - кэш данных
	* rabbitmq - брокер сообщений RabbitMQ
* Взаимодействие компонентов контейнеров frontend-backend происходит по REST API.
* Взаимодействие компонентов node-opc, wd-validator, wd-collector wd-calculator, wd-transmitter происходит через очереди брокера сообщений асинхпронно.
* Доступ к БД PostgreSQL осуществляется по ORM/SQL
* Доступ к БД Mongo DB осуществляется по ORM записью данных параметров бурения.
* Сервис redis используется для кэширования наиболее часто данных
* Удаленные сервисы сбора данных подключаются к сервису node-opc по различным протоколам через opc интерфейс.
* Внешние поставщики данных бурения подключаются по специальному протоколу WITSML.
* Пользователи подключаются к web-серверу и взаимодействуют с компонетами frontend по протоколу HTTP(S).
* Разделение компонентов на контейнеры позволяет модифицировать их по отдельности, но нужно продумать систему версионирования.


## Декомпозиция слоя данных

* Все поступающие данные классифицируются по источникам и различаются по способам хранения в базах данных:
	* Данные параметров бурения (временные ряды) передаются через сервис node-opc, проходят через первичный валидатор wd-validator и затем поступают в компонент wd-collector, который записывает информацию в БД Mongo DB (mongodb) и далее в кэш redis.
	* Данные внешних сервисов и информация, вносимая через web-интерфейс (информация, относящаяся к бизнес-домену - скважины, оборудование, буровая установк и т.п.) проходит через валидатор и также поступает в компонент wd-collector, который записывает ее в БД PostgreSQL. 
* Временные ряды используются для анализа и воспроизведения графиков.
* Для вычисления, определения оперций и т.п. используются данные из всех БД, в зависимости от вида.
* Предполагается использовать паттерн CQRS для разделения записи (mongodb) и чтения (postgres), также необходимо модифицировать сервис wd-collector.
* Для ускорения доступа к данным временных рядов используется компонент redis, который получает информацию из mongodb
* Для обеспечения надежной передачи данных в облако реализован паттерн Transactional OutBox, где компонент  wd-collector осуществляет транзакционную (по БД) запись информации в БД PostgreSQL и в таблицу Outbox, а wd-transmitter транзакционно (по БД) считывает информацию из таблицы Outbox и отправляет данные в очередь для передачи в облако. 

## Диаграмма развертывания

![Диаграмма развертывания](ADR7_data/deployment_diagram.png)
 
* Диаграмма развертывания, по сути, повторяет вид контейнерной диаграммы и показывет размещение контейнеров приложения на сервере WellDriver
* Здесь же показаны компоненты приложения внутри контейнеров и способы взаимодействия.

## Оценка атрибутов качества по заданным критериям

* Доступность - здесь подразумевается то, что сервис должен быть в работе (доступен) в любое время  с вероятностью 99,9% (исходя из специфики работы). Это гарантирутся выбранными архитектурными решениями по бесперебойной и надежной передаче данных через брокер сообщений. 
* Производительность - время обработки в нормальных условиях (при наличии связи) < 10 секунд (реальное время). Здесь используются принятые решения по асинхронной модели передачи данных по подписке с помощью высокопроизводительного брокера сообщений RabbitMQ и хранения оперативных данных в кэше. Использование паттерна CQRS и разделение данных по виду источника позволяет гибко управлять потоками данных, что также повышает производительность.
* Надежность - использование в межсервисных коммуникациях надежного брокера сообщений RabbitMQ с возможностью подтверждения передачи гарантирует надежную передачу данных без потерь. Использование паттерна Transactional OutBox гарантирует сохранение и передачу данных с соблюдением консистентности данных в конечном счете.
* Развертываемость  - обеспечивается разграничением компонентов по контейнерам и реализацией раздельной загрузки контейнеров с помощью CI/CD.
* Модифицируемость системы - возможно дальнейшее разделение компонентов по контейнерам и дальнейшая реализация их в качестве микросервисов.


# Решение:
По системному анализу целесообразна упаковка в контейнеры согласно приведенной схеме. По способам работы с данными необходимо применить паттерн CQRS для разделения методов работы с информацией. Диаграмма развертывания показывает возможность автоматической установки системы на один сервер на объекте.
 
# Последствия решения:

* Подход позволяет производить автоматическое развертывание системы на удаленных объектах.
* Необходима стройная система версионирования.
* Необходимость настройки CI/CD для автоматического версионирования и развертывания.
* В дальнейшем возможно более мелкое разбиение компонентов на контейнеры для размещения на разных серверах системы.
	
## Возможные риски

* Дальнейшее разделение компонентов по контейнерам может привести к избыточному потреблению памяти.
* Рефакторинг сервиса wd_collector.