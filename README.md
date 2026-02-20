# Лабораторная работа №2: Массовый импорт и транзакции

## Описание

Система управления объектами Person с поддержкой:
- Массового импорта через JSON файлы
- Транзакционной безопасности (all-or-nothing)
- Проверки уникальности на уровне бизнес-логики
- Истории операций импорта
- Нагрузочного тестирования через JMeter

## Технологический стек

**Backend:**
- Jakarta EE 10
- Hibernate 5.6
- PostgreSQL
- JAX-RS (Jersey)
- WebSocket

**Frontend:**
- React 18
- Axios

**Testing:**
- Apache JMeter 5.6+

## Новые возможности ЛР2

### 1. Массовый импорт Person

Импорт множества объектов Person через JSON файл с:
- Транзакционной безопасностью (всё или ничего)
- Проверкой уникальности passportID
- Проверкой уникальности комбинации name + coordinates
- Валидацией данных
- Созданием вложенных объектов (Coordinates, Location)

### 2. История импорта

- ID операции
- Статус (SUCCESS/FAILED)
- Пользователь
- Время выполнения
- Количество импортированных объектов
- Сообщение об ошибке

### 3. Бизнес-логика уникальности

**Реализовано на уровне Java (НЕ в БД):**
- `passportID` должен быть уникальным
- Комбинация `name + coordinates (x, y)` должна быть уникальной

### 4. Уровни изоляции транзакций

- **READ_COMMITTED** для простых CRUD операций
- **SERIALIZABLE** для операций импорта с проверкой уникальности
- Настраивается в `hibernate.cfg.xml`

### 5. JMeter тестирование

5 Thread Groups для проверки:
- Concurrent Create (проверка уникальности passportID)
- Concurrent Update (lost updates)
- Concurrent Delete
- Concurrent Import (транзакционная изоляция)
- Mixed Workload (общая стабильность)

## Структура проекта

```
is-lab2/
├── src/main/java/ru/ifmo/person/
│   ├── model/              # Модели (Person, Location, ImportHistory)
│   ├── dto/                # DTO объекты
│   ├── repository/         # Слой доступа к данным
│   ├── service/            # Бизнес-логика
│   ├── controller/         # REST контроллеры
│   ├── mapper/             # Маппинг DTO <-> Entity
│   └── enumeration/        # Enums (Color, Country, ImportStatus)
├── src/main/resources/
│   ├── hibernate.cfg.xml   # Конфигурация Hibernate
│   └── ru/ifmo/person/model/*.hbm.xml  # Hibernate mappings
├── frontend/
│   └── src/
│       ├── pages/          # React компоненты
│       │   └── Import.js   # NEW: Страница импорта
│       └── services/
│           └── api.js      # API клиент
├── test-import-data/       # NEW: Тестовые JSON файлы
│   ├── valid-import.json
│   ├── large-import-50.json
│   ├── duplicate-passport-import.json
│   └── conflict-passport-in-batch.json
├── jmeter/                 # NEW: JMeter тесты
│   ├── test-plan.jmx
│   └── README.md
├── TRANSACTION-ANALYSIS.md # NEW: Анализ изоляции транзакций
└── README.md
```

## Быстрый старт

### Установка JMeter на macOS

```bash
# Через Homebrew (рекомендуется)
brew install jmeter

# Проверка
jmeter --version
```

Подробные инструкции: [`jmeter/README.md`](jmeter/README.md)

### Сборка и развертывание

```bash
# Сборка backend
mvn clean package

# Развертывание на сервер (пример для Helios)
./deploy-helios.sh

# Или запуск локально (требует Wildfly/Payara)
# ...
```

### Запуск frontend

```bash
cd frontend
npm install
npm start
```

## Демонстрация работы

### 1. Успешный импорт

1. Открыть http://localhost:3000/import
2. Выбрать файл `test-import-data/valid-import.json`
3. Ввести username (например, "user1")
4. Нажать "Импортировать"
5. **Результат**: SUCCESS, 3 объекта импортировано

### 2. Ошибка уникальности passportID

1. Импортировать `valid-import.json` (создаст записи с известными passportID)
2. Импортировать `duplicate-passport-import.json`
3. **Результат**: FAILED, "Person with passportID 'RU1234567' already exists"

### 3. Ошибка уникальности name+coordinates

1. Импортировать `duplicate-name-coords-import.json`
2. **Результат**: FAILED, "Person with name 'Zero Two' and coordinates (250.0, 500) already exists"

### 4. Конфликт внутри батча

1. Импортировать `conflict-passport-in-batch.json`
2. **Результат**: FAILED, "Duplicate passportID 'CONFLICT001' found within import batch at positions 1 and 2"

### 5. Большой импорт (50 объектов)

1. Импортировать `large-import-50.json`
2. **Результат**: SUCCESS, 50 объектов импортировано
3. Демонстрирует работу с большими объемами данных

## Тестирование с JMeter

### Подготовка

1. Установить JMeter (см. выше)
2. Убедиться, что backend запущен
3. Открыть test plan:

```bash
cd jmeter
jmeter -t test-plan.jmx
```

### Запуск тестов

**GUI режим (для разработки):**
1. Открыть JMeter GUI
2. Загрузить `test-plan.jmx`
3. Настроить переменные (API_BASE_URL, если нужно)
4. Нажать Start (зеленый треугольник)
5. Наблюдать результаты в "View Results Tree"

**CLI режим (для production):**

```bash
# Все тесты
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/

# Посмотреть HTML отчет
open report/index.html
```

### Анализ результатов

**Проверить:**
1. Thread Group 1: Только 1 из 5 успешно создал Person
2. Thread Group 3: Только 1 из 2 успешно удалил Person
3. Thread Group 4: Проверить количество импортированных Person в БД
4. Thread Group 5: Нет deadlock'ов, система стабильна

**Запросы к БД:**

```sql
-- Проверить дубликаты passportID
SELECT passport_id, COUNT(*) 
FROM persons 
GROUP BY passport_id 
HAVING COUNT(*) > 1;

-- Проверить дубликаты name+coordinates
SELECT name, x, y, COUNT(*) 
FROM persons 
GROUP BY name, x, y 
HAVING COUNT(*) > 1;

-- История импорта
SELECT * FROM import_history 
ORDER BY created_at DESC 
LIMIT 20;
```

## Бизнес-логика уникальности

### Реализация

Проверки выполняются в `ImportService.importPersons()`:

```java
// 1. Проверка внутри батча
for (int i = 0; i < personDtos.size(); i++) {
    for (int j = 0; j < i; j++) {
        // Проверка passportID
        if (personDtos.get(i).getPassportID().equals(personDtos.get(j).getPassportID())) {
            throw new IllegalArgumentException("Duplicate passportID...");
        }
        // Проверка name+coordinates
        ...
    }
}

// 2. Проверка с существующими данными в БД
for (PersonDto dto : personDtos) {
    personService.validatePersonUniqueness(dto);
}
```

### PersonService.validatePersonUniqueness()

```java
public void validatePersonUniqueness(PersonDto personDto) {
    // Проверка passportID
    Person existingByPassport = personRepository.findByPassportID(personDto.getPassportID());
    if (existingByPassport != null && !existingByPassport.getId().equals(personDto.getId())) {
        throw new IllegalArgumentException("Person with passportID '" + 
            personDto.getPassportID() + "' already exists");
    }
    
    // Проверка name+coordinates
    Person existingByNameAndCoords = personRepository.findByNameAndCoordinates(
        personDto.getName(), 
        personDto.getCoordinates().getX(), 
        personDto.getCoordinates().getY()
    );
    if (existingByNameAndCoords != null && ...) {
        throw new IllegalArgumentException("Person with name '" + 
            personDto.getName() + "' and coordinates (...) already exists");
    }
}
```

**Важно**: Проверки выполняются на уровне Java, **НЕ** через UNIQUE constraints в БД.

## Транзакционная безопасность

### All-or-Nothing

```java
@ApplicationScoped
public class ImportService {
    public ImportHistoryDto importPersons(List<PersonDto> personDtos, String username) {
        ImportHistory history = new ImportHistory();
        
        try {
            // Проверки уникальности (может выбросить исключение)
            validateBatchUniqueness(personDtos);
            
            // Сохранение ВСЕХ Person в одной транзакции
            personRepository.saveAll(personsToSave);
            
            // Успех
            history.setStatus(ImportStatus.SUCCESS);
            history.setImportedCount(personsToSave.size());
            
        } catch (Exception e) {
            // Откат транзакции, НИ ОДИН объект не создан
            history.setStatus(ImportStatus.FAILED);
            history.setImportedCount(0);
            history.setErrorMessage(e.getMessage());
        }
        
        // Сохранить историю операции
        return importHistoryRepository.save(history);
    }
}
```

### PersonRepository.saveAll()

```java
public void saveAll(List<Person> persons) {
    Transaction tx = null;
    try (Session session = getSession()) {
        tx = session.beginTransaction();
        
        for (Person person : persons) {
            session.persist(person);
        }
        
        session.flush();
        tx.commit();  // Все объекты сохранены
        
    } catch (Exception e) {
        if (tx != null) tx.rollback();  // Откат ВСЕХ изменений
        throw new RuntimeException("Batch save failed: " + e.getMessage(), e);
    }
}
```

## Уровни изоляции транзакций

### Конфигурация

**hibernate.cfg.xml:**
```xml
<property name="hibernate.connection.isolation">2</property>
```

- `1` = READ_UNCOMMITTED (не поддерживается в PostgreSQL)
- `2` = READ_COMMITTED (по умолчанию)
- `4` = REPEATABLE_READ
- `8` = SERIALIZABLE (для Import операций)

### Обоснование выбора

См. подробный анализ в [`TRANSACTION-ANALYSIS.md`](TRANSACTION-ANALYSIS.md)

**Краткая выжимка:**
- **CRUD операции**: READ_COMMITTED (баланс производительности и безопасности)
- **Import операции**: SERIALIZABLE (предотвращение Phantom Reads при проверке уникальности)

### Проблемы и решения

**Phantom Read при проверке уникальности:**
```
Thread 1: проверяет passportID="JP001" (не найдено)
Thread 2: проверяет passportID="JP001" (не найдено)
Thread 1: создает Person с passportID="JP001"
Thread 2: создает Person с passportID="JP001"
Результат: ДУБЛИКАТ!
```

**Решение**: SERIALIZABLE для Import операций

## API Endpoints

### Import

**POST** `/api/import/persons?username={username}`
```json
[
  {
    "name": "Test Person",
    "coordinates": {"x": 100.5, "y": 200},
    "eyeColor": "BLUE",
    "hairColor": "BLACK",
    "location": {"x": 10, "y": 20.5, "z": 30, "name": "Location"},
    "height": 175,
    "weight": 70,
    "passportID": "JP1234567",
    "nationality": "GERMANY"
  }
]
```

**Response:**
```json
{
  "id": 1,
  "status": "SUCCESS",
  "createdAt": "2026-02-20T10:30:00",
  "username": "user1",
  "importedCount": 1,
  "errorMessage": null
}
```

### Import History

**GET** `/api/import/history?username={username}`

Возвращает список операций импорта:
- Без параметра `username` - все операции (режим "админа")
- С параметром `username` - только операции этого пользователя

## Ключевые моменты для защиты

### 1. Демонстрация импорта

- Показать успешный импорт (valid-import.json)
- Показать ошибку при дублировании passportID
- Показать ошибку при дублировании name+coordinates

### 2. Код уникальности

- Показать `validatePersonUniqueness()` в PersonService
- Подчеркнуть: **это бизнес-логика, а не БД constraint**
- Показать проверку внутри батча в ImportService

### 3. Транзакция

- Показать `saveAll()` в PersonRepository
- Объяснить all-or-nothing поведение
- Показать откат транзакции при ошибке

### 4. JMeter результаты

- Кратко объяснить каждый Thread Group
- Показать выбор уровней изоляции
- Обосновать READ_COMMITTED для CRUD и SERIALIZABLE для Import

## Документация

- **Анализ транзакций**: [`TRANSACTION-ANALYSIS.md`](TRANSACTION-ANALYSIS.md)
- **JMeter инструкции**: [`jmeter/README.md`](jmeter/README.md)
- **Тестовые данные**: [`test-import-data/README.md`](test-import-data/README.md)

## Авторы

Студент: s409858
Курс: Информационные системы
Лабораторная работа: №2

## Лицензия

Учебный проект ИТМО
