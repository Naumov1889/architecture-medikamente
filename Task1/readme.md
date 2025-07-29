## Table of content
- [Проблемы](#проблемы)
- [Типы данных и их защита](#типы-данных-и-их-защита)
- [Data Flow Diagram](#data-flow-diagram)


## Проблемы
Все операции с данными команда производит вручную.

Журналы приёма пациентов, учёт пациентов и платежей, медицинские карты, учёт анализов лежат на общем диске.

Внутренние потоки данных между IT-продуктами никак не контролируются. Ограничения по передаваемым данным обусловлены только бизнес-процессами компании. Кроме доменной аутентификации, со стороны систем обработки нет ограничений на доступ к данным.

Ручная запись клиентов.

Работа с конфиденциальными данными и PII (Personally Identifiable Information) не в полной мере отвечает требованиям российского законодательства.

## Типы данных и их защита

Любой бизнес-процесс в компании сопровождается обращением к конфиденциальным данным разного уровня.

### Персональные данные 
ФИО, Дата рождения, Телефон, Электронная почта, Адрес прописки, Место работы / учёбы.

Защита: 
- Шифрование — для хранения и передачи.
- Обезличивание — если данные нужны для аналитики без привязки к личности (замена на псевдонимы или хеши).

### Медицинские данные 
медкарта, информация о приёмах, диагнозы, назначения, результаты анализов.

Защита:
- Шифрование — обязательно при хранении и передаче.
- Обезличивание — особенно при передаче третьим лицам (исследования, внешние системы).

### Финансовые данные
платежи, финансовые отчеты, данные кассы, зарплаты - конфиденциальные данные.

Защита
- Шифрование — для всех данных (банковские транзакции, суммы, зарплаты).
- Обфускация — можно применять к частям (например, маскирование номеров карт, ИНН, счетов).

### Тегирование
Добавим в таблицы postgresql столбец tags или metadata. В них будут нужные теги. По этим тегам и пользователю реализуем ABAC (Attribute-Based Access Control), используя RLS (Row-Level Security) или логику в приложении.

### Ещё способы защиты
- HashiCorp Vault — для ключей и секретов
- PostgreSQL с шифрованием — вместо Excel-файлов
- ELK Stack — для аудита всех операций
- Keycloak — для ролевого доступа


## Data Flow Diagram
Остановимся на 2 процессах: запись на прием и сам прием.

### Запись на прием

![](./files/DFD-profile.svg)

<details>
<summary>plantuml</summary>

```plantuml
@startuml

skinparam rectangle {
  BackgroundColor White
  BorderColor Black
}

actor Пациент
rectangle "Сотрудник ресепшена" as Receptionist
rectangle "Компьютер ресепшена" as Workstation
database "Excel-файл на диске" as ExcelFile

Пациент --> Receptionist : паспортные данные,\nтелефон, почта
Receptionist --> Workstation : ввод данных
Workstation --> ExcelFile : сохраняет данные

@enduml
```
</details>

#### С применением способов защиты:
![](./files/DFD-profile-protected.svg)

<details>
<summary>plantuml</summary>

```plantuml
@startuml

skinparam rectangle {
  BackgroundColor White
  BorderColor Black
}

actor Пациент
rectangle "Сотрудник ресепшена" as Receptionist
rectangle "Компьютер ресепшена\n(обезличивание + шифрование)" as Workstation
database "Excel-файл на диске\n(зашифрованный)" as ExcelFile

Пациент --> Receptionist : паспортные данные,\nтелефон, почта
Receptionist --> Workstation : ввод данных\n(реальные → обезличенные)
Workstation --> ExcelFile : сохраняет данные\n(в зашифрованном виде)

@enduml
```
</details>

### Прием у врача

![](./files/DFD-appointment.svg)

<details>
<summary>plantuml</summary>

```plantuml
@startuml
skinparam rectangle {
  BackgroundColor White
  BorderColor Black
}

actor Пациент
rectangle "Врач" as Doctor
rectangle "Компьютер врача" as Workstation
database "Файл Excel:\nистория приёма,\nназначения" as ExcelFile
database "Файл с результатами\nанализов" as TestResults

' Поток: жалобы
Пациент --> Doctor : сообщает проблему

' Врач анализирует и назначает анализы
Doctor --> Workstation : ввод данных и назначений
Workstation --> ExcelFile : сохраняет историю приёма

' Пациент сдаёт анализы
Пациент --> TestResults : сдаёт анализы

' Результаты анализов сохраняются
TestResults --> TestResults : хранение результатов

' Врач изучает результаты и ставит диагноз
Doctor --> TestResults : читает результаты
Doctor --> Workstation : записывает диагноз
Workstation --> ExcelFile : сохраняет диагноз

@enduml
```
</details>


#### С применением способов защиты:
![](./files/DFD-appointment-protected.svg)

<details>
<summary>plantuml</summary>

```plantuml
@startuml
skinparam rectangle {
  BackgroundColor White
  BorderColor Black
}

actor Пациент
rectangle "Врач" as Doctor
rectangle "Компьютер врача\n(обезличивание + шифрование)" as Workstation
database "Файл Excel:\nистория приёма, назначения\n(зашифрованный)" as ExcelFile
database "Файл с результатами\nанализов (зашифрованный)" as TestResults

' Поток: жалобы
Пациент --> Doctor : сообщает проблему

' Врач анализирует и назначает анализы
Doctor --> Workstation : ввод жалоб, назначений\n(с обезличиванием)
Workstation --> ExcelFile : сохраняет данные\n(в зашифрованном виде)

' Пациент сдаёт анализы
Пациент --> TestResults : сдаёт анализы\n(по ID пациента)

' Результаты анализов сохраняются
TestResults --> TestResults : хранение результатов\n(зашифровано)

' Врач изучает результаты и ставит диагноз
Doctor --> TestResults : читает результаты\n(дешифровка внутри системы)
Doctor --> Workstation : записывает диагноз
Workstation --> ExcelFile : сохраняет диагноз\n(зашифрованный, обезличенный)

@enduml
```
<details>