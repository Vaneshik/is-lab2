# Лабораторная работа №2 - Защита

**Студент:** s409858  
**Курс:** Информационные системы  
**Тема:** Массовый импорт и транзакции

---

## Содержание

1. [Теоретическая часть](#теоретическая-часть)
2. [Практическая реализация](#практическая-реализация)
3. [Сценарий демонстрации](#сценарий-демонстрации)
4. [Ответы на вопросы](#ответы-на-вопросы)

---

## Теоретическая часть

### 1. Понятие бизнес-логики в программных системах

**Бизнес-логика** — это набор правил и алгоритмов, определяющих, как данные создаются, обрабатываются и изменяются в соответствии с требованиями бизнеса.

**Уровень бизнес-логики в многоуровневой архитектуре:**

```
┌─────────────────────────┐
│  Presentation Layer     │  ← UI, Frontend, REST API
├─────────────────────────┤
│  Business Logic Layer   │  ← Бизнес-правила, валидация
├─────────────────────────┤
│  Data Access Layer      │  ← Работа с БД, Repository
├─────────────────────────┤
│  Database Layer         │  ← Хранение данных
└─────────────────────────┘
```

**Ответственность Business Logic Layer:**
- Валидация данных (бизнес-правила, а не просто типы)
- Проверка уникальности
- Вычисления и агрегация
- Координация транзакций
- Обработка бизнес-процессов

**Пример в проекте:**
```java
// PersonService.validatePersonUniqueness() - это бизнес-логика
public void validatePersonUniqueness(PersonDto personDto) {
    // Правило: passportID должен быть уникальным
    Person existing = personRepository.findByPassportID(personDto.getPassportID());
    if (existing != null) {
        throw new IllegalArgumentException("Person with passportID already exists");
    }
}
```

---

### 2. Jakarta Enterprise Beans (EJB)

**EJB** — это серверные компоненты, реализующие бизнес-логику в Jakarta EE приложениях.

**Виды бинов:**

| Тип | Назначение | Особенности |
|-----|-----------|-------------|
| **Session Beans** | Бизнес-логика, обработка запросов | Stateless, Stateful, Singleton |
| **Message-Driven Beans (MDB)** | Асинхронная обработка сообщений | Работа с JMS |
| **Entity Beans** (устарело) | Персистентность данных | Заменены на JPA Entities |

**Session Beans подробнее:**

1. **Stateless Session Bean**
   - Не хранит состояние между вызовами
   - Масштабируется горизонтально
   - Пример: `@Stateless public class OrderService { ... }`

2. **Stateful Session Bean**
   - Сохраняет состояние клиента между вызовами
   - Каждому клиенту — свой экземпляр
   - Пример: корзина покупок

3. **Singleton Session Bean**
   - Единственный экземпляр на весь контейнер
   - Управление общими ресурсами
   - Пример: кэш, счетчики

**В проекте используется CDI вместо EJB:**
```java
@ApplicationScoped  // аналог Stateless + Singleton
public class ImportService {
    // бизнес-логика импорта
}
```

---

### 3. EJB Session Beans. Жизненный цикл

**Stateless Session Bean:**

```
    [Does Not Exist]
           ↓
      @PostConstruct
           ↓
    [Method Ready Pool]  ← контейнер управляет пулом экземпляров
           ↓
       business method() 
           ↓
      @PreDestroy
           ↓
    [Does Not Exist]
```

**Stateful Session Bean:**

```
    [Does Not Exist]
           ↓
      @PostConstruct
           ↓
    [Method Ready]
           ↓
    business method()
           ↓
      @PrePassivate  (контейнер сохраняет состояние на диск)
           ↓
      [Passive]
           ↓
      @PostActivate  (контейнер восстанавливает состояние)
           ↓
    [Method Ready]
           ↓
      @PreDestroy / @Remove
           ↓
    [Does Not Exist]
```

**Singleton Session Bean:**

```
    [Does Not Exist]
           ↓
      @PostConstruct
           ↓
    [Ready]  ← единственный экземпляр
           ↓
    business methods()
           ↓
      @PreDestroy (при остановке контейнера)
           ↓
    [Does Not Exist]
```

**Аннотации:**
- `@PostConstruct` — инициализация после создания
- `@PreDestroy` — очистка перед уничтожением
- `@PrePassivate` / `@PostActivate` — для Stateful
- `@Remove` — явное удаление Stateful bean

---

### 4. Понятие транзакции. ACID

**Транзакция** — это последовательность операций с данными, которая выполняется как единое целое (атомарно).

**ACID — свойства транзакций:**

| Свойство | Описание | Пример |
|----------|----------|--------|
| **Atomicity (Атомарность)** | Все операции выполняются или ни одна | Перевод денег: либо оба счета обновляются, либо ни один |
| **Consistency (Согласованность)** | БД остается в корректном состоянии | Сумма на всех счетах не меняется |
| **Isolation (Изолированность)** | Параллельные транзакции не мешают друг другу | Два перевода не видят промежуточные состояния |
| **Durability (Долговечность)** | После commit изменения сохранены навсегда | После подтверждения перевода сбой не отменит его |

**В проекте:**
```java
// Atomicity: либо импортируются ВСЕ Person, либо НИ ОДИН
personRepository.saveAll(personsToSave);  // одна транзакция

// При ошибке - rollback всех изменений
catch (Exception e) {
    if (tx != null) tx.rollback();
}
```

---

### 5. Виды конфликтов при многопользовательской работе. Уровни изоляции

**Проблемы параллельного доступа:**

| Проблема | Описание | Пример |
|----------|----------|--------|
| **Dirty Read** | Чтение незакоммиченных данных | Транзакция A читает данные, измененные транзакцией B, которая потом откатывается |
| **Non-Repeatable Read** | Разные результаты при повторном чтении | Транзакция A читает запись, B изменяет ее, A снова читает — результат другой |
| **Phantom Read** | Появление новых записей при повторном запросе | A читает все Person с height > 180. B добавляет нового. A повторяет запрос — новая запись появилась |
| **Lost Update** | Потеря изменений | A и B читают Person, оба изменяют, оба сохраняют — изменения A теряются |

**Уровни изоляции транзакций (от слабого к сильному):**

| Уровень | Dirty Read | Non-Rep. Read | Phantom Read | Производительность |
|---------|------------|---------------|--------------|---------------------|
| **READ UNCOMMITTED** | ✗ Возможен | ✗ Возможен | ✗ Возможен | Очень высокая |
| **READ COMMITTED** | ✓ Запрещен | ✗ Возможен | ✗ Возможен | Высокая |
| **REPEATABLE READ** | ✓ Запрещен | ✓ Запрещен | ✗ Возможен | Средняя |
| **SERIALIZABLE** | ✓ Запрещен | ✓ Запрещен | ✓ Запрещен | Низкая |

**В проекте:**
```xml
<!-- hibernate.cfg.xml -->
<property name="hibernate.connection.isolation">2</property>
<!-- 1=READ_UNCOMMITTED, 2=READ_COMMITTED, 4=REPEATABLE_READ, 8=SERIALIZABLE -->
```

**Обоснование:**
- **CRUD операции**: READ_COMMITTED (баланс производительности и безопасности)
- **Import операции**: SERIALIZABLE рекомендуется для предотвращения Phantom Reads при проверке уникальности

**Пример Phantom Read в проекте:**
```
Thread 1: проверяет passportID="JP001" → НЕ НАЙДЕНО
Thread 2: проверяет passportID="JP001" → НЕ НАЙДЕНО
Thread 1: создает Person с passportID="JP001"
Thread 2: создает Person с passportID="JP001"
Результат: ДУБЛИКАТ в БД!
```

---

### 6. Транзакции на уровне бизнес-логики vs БД

**Транзакции на уровне БД:**
- Управляются СУБД
- Ограничены одним подключением
- Автоматический откат при сбое соединения
- Используют блокировки и MVCC

**Транзакции на уровне бизнес-логики:**
- Управляются приложением (JTA, Spring)
- Могут охватывать несколько ресурсов (БД, JMS, внешние сервисы)
- Требуют явного управления (commit/rollback)
- Могут включать бизнес-проверки до commit

**Отличия:**

| Аспект | БД транзакция | Бизнес-логика транзакция |
|--------|---------------|--------------------------|
| **Границы** | BEGIN...COMMIT в БД | @Transactional метод |
| **Ресурсы** | Одна БД | Несколько БД, JMS, REST API |
| **Откат** | Автоматически при сбое | Программно или по исключению |
| **Бизнес-правила** | Через constraints | Проверяются в коде |
| **Компенсация** | Нет | Возможна (Saga) |

**Пример в проекте:**
```java
// Транзакция на уровне бизнес-логики
public ImportHistoryDto importPersons(List<PersonDto> personDtos, String username) {
    try {
        // 1. Бизнес-проверки (до обращения к БД)
        validateBatchUniqueness(personDtos);
        
        // 2. БД транзакция внутри
        personRepository.saveAll(personsToSave);  // BEGIN...COMMIT
        
        // 3. WebSocket уведомление (вне БД транзакции)
        PersonWebSocket.notifyClients("created", person.getId());
        
        return SUCCESS;
    } catch (Exception e) {
        // 4. Логирование в import_history (отдельная транзакция)
        return FAILED;
    }
}
```

---

### 7. Java Transaction API (JTA)

**JTA** — стандартный Java API для работы с распределенными транзакциями.

**Основные интерфейсы:**

```java
// javax.transaction.UserTransaction
UserTransaction ut = ...;
ut.begin();
try {
    // операции с несколькими ресурсами
    dataSource1.update(...);
    jmsQueue.send(...);
    ut.commit();
} catch (Exception e) {
    ut.rollback();
}

// javax.transaction.TransactionManager
// управление транзакциями на уровне контейнера

// javax.transaction.Transaction
// текущая транзакция
```

**Основные принципы:**

1. **XA протокол** — two-phase commit (2PC)
   - Phase 1: Prepare (все ресурсы готовы к commit?)
   - Phase 2: Commit (все подтверждают)

2. **TransactionManager** координирует распределенные транзакции

3. **XAResource** — интерфейс для ресурсов (БД, JMS)

**Пример:**
```java
@Stateless
public class OrderService {
    @Resource
    private UserTransaction ut;
    
    public void createOrder() throws Exception {
        ut.begin();
        try {
            // обновить БД
            orderDAO.insert(...);
            // отправить сообщение в JMS
            jmsProducer.send(...);
            ut.commit();
        } catch (Exception e) {
            ut.rollback();
            throw e;
        }
    }
}
```

---

### 8. Управление транзакциями в Jakarta EE

**Декларативное управление (Container-Managed Transactions, CMT):**

```java
@Stateless
@TransactionManagement(TransactionManagementType.CONTAINER)  // по умолчанию
public class PersonService {
    
    @TransactionAttribute(TransactionAttributeType.REQUIRED)  // по умолчанию
    public void createPerson(Person p) {
        // транзакция управляется контейнером
        em.persist(p);
    }
    
    @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
    public void logError(String message) {
        // новая транзакция, независимо от внешней
    }
}
```

**Атрибуты транзакций:**

| Атрибут | Описание |
|---------|----------|
| `REQUIRED` | Использует существующую или создает новую (по умолчанию) |
| `REQUIRES_NEW` | Всегда создает новую транзакцию |
| `MANDATORY` | Требует существующую, иначе исключение |
| `NOT_SUPPORTED` | Выполняется без транзакции |
| `SUPPORTS` | Использует транзакцию, если есть |
| `NEVER` | Выбрасывает исключение, если транзакция существует |

**Программное управление (Bean-Managed Transactions, BMT):**

```java
@Stateless
@TransactionManagement(TransactionManagementType.BEAN)
public class ImportService {
    
    @Resource
    private UserTransaction ut;
    
    public void importData() throws Exception {
        ut.begin();
        try {
            // ручное управление
            validateData();
            saveData();
            ut.commit();
        } catch (Exception e) {
            ut.rollback();
            throw e;
        }
    }
}
```

**В проекте используется JPA/Hibernate с ручным управлением:**
```java
public void saveAll(List<Person> persons) {
    Transaction tx = null;
    try (Session session = getSession()) {
        tx = session.beginTransaction();
        
        for (Person person : persons) {
            session.persist(person);
        }
        
        tx.commit();
    } catch (Exception e) {
        if (tx != null) tx.rollback();
        throw new RuntimeException("Batch save failed", e);
    }
}
```

---

### 9. Управление транзакциями в Spring

**Декларативное управление (аннотация @Transactional):**

```java
@Service
public class PersonService {
    
    @Transactional  // по умолчанию: propagation=REQUIRED, isolation=DEFAULT, readOnly=false
    public void createPerson(Person p) {
        personRepository.save(p);
    }
    
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.SERIALIZABLE,
        timeout = 30,
        rollbackFor = Exception.class,
        noRollbackFor = CustomException.class
    )
    public void importPersons(List<Person> persons) {
        // новая транзакция с SERIALIZABLE изоляцией
    }
    
    @Transactional(readOnly = true)
    public List<Person> findAll() {
        // оптимизация для read-only
        return personRepository.findAll();
    }
}
```

**Propagation (распространение):**

| Тип | Описание |
|-----|----------|
| `REQUIRED` | Использует текущую или создает новую (по умолчанию) |
| `REQUIRES_NEW` | Всегда создает новую, приостанавливает текущую |
| `NESTED` | Вложенная транзакция (savepoint) |
| `MANDATORY` | Требует существующую транзакцию |
| `SUPPORTS` | Выполняется в транзакции, если есть |
| `NOT_SUPPORTED` | Приостанавливает текущую транзакцию |
| `NEVER` | Выбрасывает исключение при наличии транзакции |

**Программное управление:**

```java
@Service
public class ImportService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void importData() {
        transactionTemplate.execute(status -> {
            try {
                // операции в транзакции
                validateData();
                saveData();
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
    
    // или через PlatformTransactionManager
    @Autowired
    private PlatformTransactionManager txManager;
    
    public void manualTransaction() {
        TransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = txManager.getTransaction(def);
        try {
            // операции
            txManager.commit(status);
        } catch (Exception e) {
            txManager.rollback(status);
        }
    }
}
```

**Конфигурация:**

```java
@Configuration
@EnableTransactionManagement
public class AppConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```

**Важно:**
- `@Transactional` работает только на public методах
- Не работает при вызове из того же класса (self-invocation)
- Откат по умолчанию только для RuntimeException и Error

---

## Практическая реализация

### Что реализовано в проекте

#### 1. Массовый импорт Person через JSON

**Endpoint:**
```http
POST /api/import/persons?username=user1
Content-Type: application/json

[
  {
    "name": "Test Person",
    "coordinates": {"x": 100.5, "y": 200},
    "eyeColor": "BLUE",
    "hairColor": "BLACK",
    "location": {"x": 10, "y": 20.5, "z": 30, "name": "Tokyo"},
    "height": 175,
    "weight": 70,
    "passportID": "JP1234567",
    "nationality": "GERMANY"
  }
]
```

#### 2. Бизнес-логика уникальности (НЕ в БД!)

**PersonService.validatePersonUniqueness():**
```java
public void validatePersonUniqueness(PersonDto personDto) {
    // 1. Проверка passportID
    Person existingByPassport = personRepository.findByPassportID(personDto.getPassportID());
    if (existingByPassport != null && 
        (personDto.getId() == null || existingByPassport.getId() != personDto.getId())) {
        throw new IllegalArgumentException(
            "Person with passportID '" + personDto.getPassportID() + "' already exists"
        );
    }
    
    // 2. Проверка name + coordinates
    Person existingByNameAndCoords = personRepository.findByNameAndCoordinates(
        personDto.getName(), 
        personDto.getCoordinates().getX(), 
        personDto.getCoordinates().getY()
    );
    if (existingByNameAndCoords != null && ...) {
        throw new IllegalArgumentException(
            "Person with name '" + personDto.getName() + 
            "' and coordinates (...) already exists"
        );
    }
}
```

#### 3. Проверка уникальности внутри батча

**ImportService.importPersons():**
```java
for (int i = 0; i < personDtos.size(); i++) {
    for (int j = 0; j < i; j++) {
        PersonDto existingDto = personDtos.get(j);
        
        // Проверка passportID внутри батча
        if (personDto.getPassportID().equals(existingDto.getPassportID())) {
            throw new IllegalArgumentException(
                "Duplicate passportID '" + personDto.getPassportID() + 
                "' found within import batch at positions " + (j+1) + " and " + (i+1)
            );
        }
        
        // Проверка name+coordinates внутри батча
        if (personDto.getName().equals(existingDto.getName()) &&
            personDto.getCoordinates().getX() == existingDto.getCoordinates().getX() &&
            personDto.getCoordinates().getY() == existingDto.getCoordinates().getY()) {
            throw new IllegalArgumentException(
                "Duplicate name+coordinates (...) found within import batch..."
            );
        }
    }
}
```

#### 4. Транзакционная безопасность (All-or-Nothing)

**PersonRepository.saveAll():**
```java
public void saveAll(List<Person> persons) {
    Transaction tx = null;
    try (Session session = getSession()) {
        tx = session.beginTransaction();
        
        for (Person person : persons) {
            session.persist(person);
        }
        
        session.flush();
        tx.commit();  // ✓ Все объекты сохранены
        
    } catch (Exception e) {
        if (tx != null) tx.rollback();  // ✗ Откат ВСЕХ изменений
        throw new RuntimeException("Batch save failed: " + e.getMessage(), e);
    }
}
```

#### 5. История импорта (ImportHistory)

**Модель:**
```java
public class ImportHistory {
    private Long id;
    private ImportStatus status;       // SUCCESS / FAILED
    private Date createdAt;
    private String username;
    private Integer importedCount;
    private String errorMessage;
}
```

**Сохранение истории:**
```java
public ImportHistoryDto importPersons(List<PersonDto> personDtos, String username) {
    ImportHistory history = new ImportHistory();
    history.setUsername(username);
    
    try {
        // валидация и сохранение
        personRepository.saveAll(personsToSave);
        
        history.setStatus(ImportStatus.SUCCESS);
        history.setImportedCount(personsToSave.size());
        
    } catch (Exception e) {
        history.setStatus(ImportStatus.FAILED);
        history.setImportedCount(0);
        history.setErrorMessage(e.getMessage());
    }
    
    // История сохраняется ВСЕГДА (и при успехе, и при ошибке)
    return importHistoryRepository.save(history);
}
```

#### 6. JMeter тесты для проверки изоляции

**5 Thread Groups:**

1. **Concurrent Create (Same PassportID)** — 5 потоков создают Person с одинаковым passportID
   - Ожидание: только 1 успешно создаст, остальные получат 400

2. **Concurrent Update (Same Person)** — 3 потока обновляют одного Person
   - Ожидание: все 3 успешны, последний wins (без lost updates)

3. **Concurrent Delete (Same Person)** — 2 потока удаляют одного Person
   - Ожидание: один получает 204, другой 404

4. **Concurrent Import (Overlapping Data)** — 3 потока импортируют с пересекающимися passportID
   - Ожидание: зависит от isolation level, но не должно быть дубликатов

5. **Mixed Workload (CRUD + Import)** — 10 потоков выполняют CRUD и Import одновременно
   - Ожидание: нет deadlock'ов, система стабильна

---

## Сценарий демонстрации

### Подготовка

1. Убедиться, что backend запущен на helios:
```bash
./deploy-helios.sh
```

2. Запустить SSH-туннель (в отдельном терминале):
```bash
./setup-local-tunnel.sh
```

3. Запустить frontend:
```bash
cd frontend && npm start
```

4. Загрузить тестовые данные:
```bash
./load-test-data.sh
```

### Демонстрация 1: Успешный импорт

1. Открыть http://localhost:3000/import
2. Выбрать файл `test-import-data/valid-import.json`
3. Ввести username "demo_user"
4. Нажать "Импортировать"

**Результат:**
- Статус: SUCCESS
- Импортировано: 3 объекта
- История обновилась автоматически

### Демонстрация 2: Ошибка дубликата passportID

1. Повторно импортировать `valid-import.json`

**Результат:**
- Статус: FAILED
- Ошибка: "Person with passportID 'JP1234567' already exists"
- Ни один объект не создан (all-or-nothing)

### Демонстрация 3: Ошибка дубликата name+coordinates

1. Импортировать `duplicate-name-coords-import.json`

**Результат:**
- Статус: FAILED
- Ошибка: "Person with name 'Zero Two' and coordinates (250.0, 500) already exists"

### Демонстрация 4: Конфликт внутри батча

1. Импортировать `conflict-passport-in-batch.json`

**Результат:**
- Статус: FAILED
- Ошибка: "Duplicate passportID 'CONFLICT001' found within import batch at positions 1 and 2"

### Показ кода

#### 1. Бизнес-логика уникальности

Открыть `src/main/java/ru/ifmo/person/service/PersonService.java`:

```java
// Строки 121-143
public void validatePersonUniqueness(PersonDto personDto) {
    // Проверка passportID
    if (personDto.getPassportID() != null && !personDto.getPassportID().isEmpty()) {
        Person existingByPassport = personRepository.findByPassportID(personDto.getPassportID());
        if (existingByPassport != null && 
            (personDto.getId() == null || existingByPassport.getId() != personDto.getId())) {
            throw new IllegalArgumentException("Person with passportID '" + 
                personDto.getPassportID() + "' already exists");
        }
    }
    
    // Проверка name+coordinates
    if (personDto.getName() != null && personDto.getCoordinates() != null) {
        Person existingByNameAndCoords = personRepository.findByNameAndCoordinates(
            personDto.getName(), 
            personDto.getCoordinates().getX(), 
            personDto.getCoordinates().getY()
        );
        if (existingByNameAndCoords != null && 
            (personDto.getId() == null || existingByNameAndCoords.getId() != personDto.getId())) {
            throw new IllegalArgumentException("Person with name '" + 
                personDto.getName() + "' and coordinates (...) already exists");
        }
    }
}
```

**Подчеркнуть:** Это НЕ UNIQUE constraint в БД, это проверка в Java коде!

#### 2. Проверка внутри батча

Открыть `src/main/java/ru/ifmo/person/service/ImportService.java`:

```java
// Строки 59-79
for (int i = 0; i < personDtos.size(); i++) {
    for (int j = 0; j < i; j++) {
        PersonDto existingDto = personDtos.get(j);
        
        // Проверка passportID
        if (personDto.getPassportID() != null && 
            personDto.getPassportID().equals(existingDto.getPassportID())) {
            throw new IllegalArgumentException(
                "Duplicate passportID '" + personDto.getPassportID() + 
                "' found within import batch at positions " + (j+1) + " and " + (i+1));
        }
        
        // Проверка name+coordinates
        if (personDto.getName().equals(existingDto.getName()) &&
            personDto.getCoordinates().getX() == existingDto.getCoordinates().getX() &&
            personDto.getCoordinates().getY() == existingDto.getCoordinates().getY()) {
            throw new IllegalArgumentException("Duplicate name+coordinates...");
        }
    }
}
```

#### 3. Транзакционная безопасность

Открыть `src/main/java/ru/ifmo/person/repository/PersonRepository.java`:

```java
// Строки 318-335
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

**Подчеркнуть:** Если хоть одна Person не прошла валидацию, ВСЕ изменения откатываются.

### Демонстрация 5: JMeter тесты

1. Открыть JMeter:
```bash
cd jmeter
jmeter -t test-plan.jmx
```

2. Показать Thread Groups:
   - Thread Group 1: Concurrent Create
   - Thread Group 4: Concurrent Import

3. Запустить тесты

4. Показать результаты в "View Results Tree"

5. Проверить БД:
```sql
-- Проверка дубликатов passportID
SELECT passport_id, COUNT(*) 
FROM persons 
GROUP BY passport_id 
HAVING COUNT(*) > 1;
-- Результат: 0 строк (дубликатов нет)

-- История импорта
SELECT id, status, username, imported_count, error_message 
FROM import_history 
ORDER BY created_at DESC 
LIMIT 10;
```

---

## Ответы на вопросы

### Почему уникальность реализована в бизнес-логике, а не в БД?

**Ответ:**
1. **Гибкость:** Бизнес-правила могут быть сложнее, чем UNIQUE constraint (например, уникальность комбинации полей с условиями)
2. **Понятные сообщения об ошибках:** Java код может вернуть "Person with passportID 'JP001' already exists", вместо generic SQL error
3. **Единообразие:** Все валидации (Bean Validation, уникальность, бизнес-правила) в одном месте
4. **Тестируемость:** Легче unit-тестировать Java код, чем constraints в БД
5. **Требование задания:** ТЗ явно требует реализацию на уровне бизнес-логики

**Минусы:**
- Возможны race conditions при конкурентном доступе (решается через isolation levels)
- Чуть ниже производительность (два запроса вместо одного INSERT)

### Как обеспечивается транзакционная безопасность?

**Ответ:**
1. **Hibernate Transaction API:**
```java
tx = session.beginTransaction();
try {
    // операции
    tx.commit();
} catch (Exception e) {
    tx.rollback();
}
```

2. **All-or-Nothing семантика:** Либо все Person сохраняются, либо ни один

3. **Валидация ДО транзакции:** Проверки уникальности выполняются перед `saveAll()`, чтобы не начинать транзакцию зря

4. **Логирование истории:** ImportHistory сохраняется в отдельной транзакции, чтобы даже при откате основной транзакции история осталась

### Какой уровень изоляции выбран и почему?

**Ответ:**

**READ_COMMITTED (по умолчанию):**
- Достаточен для большинства CRUD операций
- Хороший баланс производительности и безопасности
- Предотвращает Dirty Reads

**SERIALIZABLE (рекомендуется для Import):**
- Предотвращает Phantom Reads при проверке уникальности
- Два параллельных импорта не смогут создать дубликаты
- Снижение производительности приемлемо для редких операций импорта

**В коде:**
```xml
<!-- hibernate.cfg.xml -->
<property name="hibernate.connection.isolation">2</property>
<!-- Можно менять на 8 для SERIALIZABLE -->
```

### Как работают JMeter тесты?

**Ответ:**

JMeter создает **параллельные HTTP-запросы** к REST API:

1. **Thread Group 1:** 5 потоков одновременно отправляют POST с одинаковым passportID
   - Проверка: только 1 получает 201, остальные 400

2. **Thread Group 4:** 3 потока импортируют с пересекающимися данными
   - Проверка: нет дубликатов в БД после теста

3. **Анализ результатов:**
   - View Results Tree — детали каждого запроса
   - Summary Report — статистика (throughput, ошибки)
   - SQL запросы для проверки консистентности БД

### Что будет, если два пользователя одновременно импортируют одни и те же данные?

**Ответ:**

**С READ_COMMITTED:**
- Возможен race condition: оба проверят "не найдено" и создадут дубликат
- **Решение:** SERIALIZABLE isolation

**С SERIALIZABLE:**
- Первая транзакция создаст Person
- Вторая транзакция увидит существующий Person и выбросит исключение
- Результат: только одна успешная операция импорта

**В проекте:**
Даже с READ_COMMITTED риск минимален, т.к. проверка и сохранение в одной транзакции. Но для 100% гарантии нужен SERIALIZABLE.

---

## Команды для проверки

```bash
# Сборка и деплой
./deploy-helios.sh

# Загрузка тестовых данных
./load-test-data.sh

# Запуск JMeter
cd jmeter && jmeter -t test-plan.jmx

# Проверка БД
psql -U s409858 -d studs -h se.ifmo.ru -p 5432 -c "
SELECT passport_id, COUNT(*) 
FROM persons 
GROUP BY passport_id 
HAVING COUNT(*) > 1;
"

# История импорта
psql -U s409858 -d studs -h se.ifmo.ru -p 5432 -c "
SELECT * FROM import_history 
ORDER BY created_at DESC 
LIMIT 10;
"
```

---

## Выводы

1. **Бизнес-логика уникальности** реализована на уровне Java (PersonService), а не в БД constraints
2. **Транзакционная безопасность** обеспечивается через Hibernate Transaction API (all-or-nothing)
3. **Уровень изоляции** READ_COMMITTED для CRUD, SERIALIZABLE рекомендуется для Import
4. **JMeter тесты** подтверждают корректность работы при конкурентном доступе
5. **История импорта** сохраняется всегда (и при успехе, и при ошибке)

---

**Подготовлено для защиты лабораторной работы №2**  
**Студент:** s409858  
**Дата:** 20.02.2026
