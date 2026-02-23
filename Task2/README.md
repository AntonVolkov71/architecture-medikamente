## Задание 2. Проектирование решения

## Цель
- Спроектировать MVP через пару месяцев с учётом Privacy by Design:
  - пациент сам записывается на приём через портал
  - ресепшен видит запись и делает напоминания пациенту и врачу
  - доступ к данным ограничен по ролям/атрибутам

## Контекст As-Is коротко
- Сейчас данные хранятся в Excel/JPG/PDF на сервере
- доступ ограничен в основном доменной аутентификацией
- нет аудита и контроля внутренних потоков.
- риски утечки PHI/PII и несоответствие требованиям законодательства РФ - ФЗ-152

## ToBe MVP
- уходим от Excel как хранилища данных
- централизованный контур управления доступом
- разделяем данные по доменам (PII / PHI / FIN / ...)
- добавляем аудит и контроль действий
- проектируем аналитику с учётом защиты данных

## Основные участники, сервисы
- Patient - использует портал для записи
- Reception - управляет расписанием и напоминаниями
- Doctor - работает с медицинской картой и результатами анализов
- Bookkeeper - бухгалтер, финансовые операция
- Warehouse keeper - учёт ТМЦ
- External Laboratory - лаборатория, передаёт результаты по API
- Payment Gateway / KKM - контур оплаты/фискализации

## Platform (To-Be - MVP) из трех слоев
- Business Services
  - Patient Portal
    - запись на приём
    - минимальный набор PII
    - нет доступа к чужим данным
  - Reception Portal
      - управление расписанием
      - подтверждения/напоминания
  - CRM Core
      - единый источник операционных данным
      - разделение доменов данных (PII/PHI/FIN/...)
      - хранение пациентов и записей
  - Billing Service
      - платежи
      - статусы
      - интеграция с 1С/ККМ/платёжным провайдером
  - Lab Integration Service
      - интеграция с лабораторией через защищённые API-контракты
      - не даёт возможности пациенту увидеть чужие данные

- Security & Governance
  - IAM (Keycloak)
      - OIDC/MFA
      - централизованная аутентификация
      - RBAC/ABAC (least privilege)
  - Policy Engine (ABAC/PDP-PEP)
      - единая точка проверки доступа (purpose/role/relationship)
  - Consent Management
    - хранение согласий пациента
    - фиксация целей обработки
    - возможность отзыва согласия
  - Audit & Logging
    - журналирование
    - централизованный аудит чтения/изменения PHI/PII
  - Data Catalog / Tagging
      - теги данных (TAG_PII, TAG_PHI, TAG_FINANCIAL, TAG_RAW, TAG_ANALYTICS)
      - Data Lineage и контроль политики доступа на основе тегов
  - DLP Engine
    - проверка выгрузок
    - предотвращение утечки чувствительных данных

- Analytics
  - Analytics Platform
    - строит отчёты и аналитику только по разрешённым наборам данных
    - перед доступом проверяет теги (tag-based access)
  - Data Lake
    - хранит выгрузки из CRM Core для аналитики
    - данные кладутся в зашифрованном виде
    - данные должны быть размечены тегами (PII/PHI/FIN и т.д.)
  - SIEM + Monitoring
    - обнаружение аномалий
    - алерты по инцидентам
    - инфраструктурные метрики
  - DLP Engine
    - проверяет выгрузки перед/после помещения в Data Lake
    - ищет чувствительные данные и фиксирует нарушения (DLP incidents)

## Потоки (Data Flows)
- Patient Portal -> API Gateway/CRM Core
    - создание записи (минимально необходимый набор данных)
- Reception Portal -> CRM Core
    - управление записью, напоминания
- CRM Core -> Notification Service/Exchange (MVP)
    - отправка напоминаний пациенту и врачу
- CRM Core <--> IAM
    - аутентификация/токены/сессии
- CRM Core -> Policy Engine
    - ABAC-проверки: кто запрашивает, зачем (purpose), по какому пациенту
- CRM Core -> Audit/SIEM
    - события доступа и изменения данных
- Laboratory -> Lab Integration Service -> CRM Core/Medical Records
    - результаты анализов по защищённому контракту (минимизация данных)
- Billing Service <--> Payment Gateway/KKM/1C
    - фискализация и учёт оплат

## Privacy by Design (коротко, по схеме)
- Доступ ограничен
  - вход в систему через IAM/Keycloak
  - права проверяются по ролям/атрибутам (RBAC/ABAC)
- Есть согласия
  - Consent Management хранит согласия пациента
  - CRM проверяет согласие перед обработкой PHI/PII (purpose-based)
- Есть аудит
  - Audit & Logging пишет события чтения/изменения PII/PHI
  - события уходят в SIEM
- Есть контроль выгрузок
  - выгрузки в аналитику идут в Data Lake
  - DLP проверяет, что ничего лишнего не утекает
- Есть теги данных
  - Data Catalog/Tagging хранит теги (PII/PHI/FIN/…)
  - CRM и Analytics используют теги для правил доступа

## Результат
- Диаграмма C4 Context To-Be (MVP) отражает:
  - 3 слоя платформы: Business Services / Security & Governance / Analytics
  - новые блоки для защиты данных:
    - IAM/Keycloak
    - Consent Management
    - Audit & Logging
    - Data Catalog/Tagging
    - DLP Engine
    - SIEM
  - интеграции:
    - лаборатория → Lab Integration Service → CRM Core
    - Billing Service → Payment Gateway
  - как данные попадают в аналитику:
    - CRM Core → Data Lake → Analytics Platform
    - Data Lake проверяется через DLP
    - инциденты и аудит уходят в SIEM