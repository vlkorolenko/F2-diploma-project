# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model: Levels 1–2)
Архітектурне проєктування системи виконано за індустріальною методологією Саймона Брауна (C4 Model) із застосуванням парадигми Diagrams-as-Code у нотації Mermaid.js.

### 2.1.1. Рівень 1: Діаграма системного контексту (System Context Diagram)
Діаграма першого рівня відображає межі системи, користувачів (Person) та зовнішні взаємодії з корпоративними і хмарними сервісами (System_Ext).

```mermaid
C4Context
    title C4 Model Level 1: System Context Diagram

    Person(developer, "Software Engineer", "Розробник Java-додатків")
    Person(security, "AppSec Auditor", "Аудитор безпеки коду")

    System(hybridScanner, "Hybrid SAST System", "Гібридний сканер безпеки на базі Spoon CPG та локальної LLM")

    System_Ext(ollamaService, "Local LLM Engine", "Локальний сервер Ollama (DeepSeek-Coder)")
    System_Ext(github, "GitHub / GitLab", "Зовнішній репозиторій вихідного коду")

    Rel_D(developer, hybridScanner, "Завантаження коду / Запуск аудиту", "HTTPS / Web")
    Rel_D(security, hybridScanner, "Тріаж вразливостей та схвалення PR", "HTTPS / Web")
    Rel_R(hybridScanner, ollamaService, "Перевірка зрізу коду", "HTTP / REST (11434)")
    Rel_R(hybridScanner, github, "Клонування та створення PR", "HTTPS / REST v3")
```

* **Трасування та наскрізні зв'язки (Cross-linking):**
  * Спроєктована система **Hybrid SAST System** реалізує функціональну вимогу безпеки [[SRS-FUN-01](../srs.md#2-functional-requirements)] щодо прийому вихідного коду через API або ZIP-архіви.
  * Контейнер **Local LLM Engine** гарантує виконання нефункціональної вимоги конфіденційності [[SRS-NFR-SEC-01](../srs.md#3-non-functional-requirements)] — повне функціонування в On-Premises контурі без витоку даних на сторонні хмарні сервери.
  * Взаємодія з **GitHub / GitLab** забезпечує автоматизовану генерацію безпечних виправлень згідно з вимогою [[SRS-FUN-08](../srs.md#2-functional-requirements)].

---

### 2.1.2. Рівень 2: Діаграма контейнерів (Container Diagram)
Діаграма другого рівня розкриває сервісну декомпозицію системи на окремо контейнеризовані Docker-модулі відповідно до обраного технологічного стеку.

```mermaid
C4Container
    title C4 Model Level 2: Container Diagram

    Person(user, "User / Security Engineer", "Користувач системи")

    Container(spa, "Web Frontend SPA", "React 18, Vite", "Клієнтський дашборд: граф атаки, diff коду та створення PR")
    Container(apiGateway, "API Gateway / Backend Core", "Spring Boot 3, Java 21", "Автентифікація, керування сесіями та API")

    Container(cpgEngine, "CPG & Slicing Engine", "Java 21, Spoon, JGraphT", "Побудова AST/CFG/DFG та Taint Slicing")
    Container(remediationWorker, "AI Remediation Worker", "Java 21, LangChain4j", "Взаємодія з LLM та валідація патчів")

    ContainerDb(postgres, "PostgreSQL DB", "PostgreSQL 16", "Збереження звітів, метаданих та CWE")
    ContainerDb(redis, "Redis Broker", "Redis 7", "Черги завдань аудиту та кеш зрізів")

    Container_Ext(localLlm, "Local LLM Container", "Ollama", "Локальний нейромережевий інференс")
    System_Ext(githubExt, "GitHub API", "REST v3", "Зовнішній Git-провайдер")

    Rel_D(user, spa, "Перегляд звітів та графа", "HTTPS")
    Rel_D(spa, apiGateway, "REST API виклики", "HTTPS / JSON")

    Rel_D(apiGateway, postgres, "Метадані та історія", "JDBC / TCP (5432)")
    Rel_R(apiGateway, redis, "Публікація задач", "RESP / TCP (6379)")

    Rel_D(redis, cpgEngine, "Отримання задач", "RESP / TCP")
    Rel_R(cpgEngine, redis, "Збереження зрізів", "RESP / TCP")

    Rel_U(redis, remediationWorker, "Вичитка зрізів коду", "RESP / TCP")
    Rel_R(remediationWorker, localLlm, "Верифікація патчу", "HTTP / REST (11434)")
    Rel_R(remediationWorker, githubExt, "Публікація PR", "HTTPS / REST")
```

* **Трасування та інженерні протоколи:**
  * Контейнер **CPG & Slicing Engine** базується на затвердженому інженерному рішенні [[ADR-001](./adr/0001-choice-of-technology.md)] щодо використання бібліотек Spoon та JGraphT для розбору коду та побудови графа CPG.
  * Контейнер **AI Remediation Worker** реалізує автоматичний синтез виправлень [[SRS-FUN-06](../srs.md#2-functional-requirements)] та пісочницю компілятора [[SRS-FUN-07](../srs.md#2-functional-requirements)].
  * Брокер **Redis Broker** гарантує високу продуктивність за вимогою [[SRS-NFR-PERF-02](../srs.md#3-non-functional-requirements)] та надійність із Dead Letter Queue згідно з [[SRS-NFR-REL-01](../srs.md#3-non-functional-requirements)].

---

## 2.2. Компонентне моделювання критичного сервісу (C4 Model Level 3: Component)
Для деталізації на компонентному рівні обрано найскладніше ядро системи — бекенд-контейнер **CPG & Slicing Engine**. Архітектуру структуровано за патерном Clean Architecture з чітким розмежуванням обов'язків (Separation of Concerns).

```mermaid
C4Component
    title C4 Model Level 3: Component Diagram (CPG & Slicing Engine)

    Container(redisBroker, "Redis Task Queue", "Redis 7", "Черга завдань аналізу")
    Container(sliceQueue, "Redis Slice Queue", "Redis 7", "Черга зрізів для ШІ")

    Container_Boundary(engineBoundary, "CPG & Slicing Engine") {
        Component(inboundConsumer, "AnalysisTaskConsumer", "Spring AMQP", "Вхідний консюмер завдань")
        Component(inputValidator, "TaskPayloadValidator", "Validation", "Валідація вхідних даних")
        Component(errorHandler, "GlobalErrorHandler", "Exception Handler", "Централізований перехоплювач збоїв")

        Component(coordinator, "WorkflowCoordinator", "App Service", "Оркестрація пайплайну аналізу")

        Component(astBuilder, "SpoonAstModelBuilder", "Domain Service", "Побудова AST через Eclipse JDT")
        Component(cfgBuilder, "ControlFlowGraphBuilder", "Domain Service", "Генерація графа переходів CFG")
        Component(dfgBuilder, "DataFlowGraphBuilder", "Domain Service", "Генерація зв'язків змінних DFG")
        Component(cpgIntegrator, "CpgIntegrator", "Domain Service", "Синтез мультиграфа CPG на JGraphT")
        Component(taintAnalyzer, "TaintAnalyzer", "Domain Service", "Трасування від Source до Sink")
        Component(slicer, "ContextProgramSlicer", "Domain Service", "Вилучення зрізу на 15-30 рядків")

        Component(outboundPublisher, "SlicePublisher", "Outbound Adapter", "Публікація зрізу в чергу")
    }

    Rel_D(redisBroker, inboundConsumer, "Завдання на аудит", "RESP")
    Rel_R(inboundConsumer, inputValidator, "Валідація DTO", "Java Call")
    Rel_D(inputValidator, coordinator, "Старт аналізу", "Method Call")
    Rel_R(coordinator, errorHandler, "Обробка помилок", "Exception Handle")

    Rel_D(coordinator, astBuilder, "Розбір коду", "Interface Call / DI")
    Rel_R(astBuilder, cfgBuilder, "Передача AST", "Java Call")
    Rel_D(astBuilder, dfgBuilder, "Таблиця символів", "Java Call")
    Rel_R(cfgBuilder, cpgIntegrator, "Ребра CFG", "JGraphT Call")
    Rel_R(dfgBuilder, cpgIntegrator, "Ребра DFG", "JGraphT Call")

    Rel_D(cpgIntegrator, taintAnalyzer, "Готовий CPG", "Graph Traversal")
    Rel_R(taintAnalyzer, slicer, "Маршрут Taint Path", "Method Call")
    Rel_D(slicer, outboundPublisher, "Зріз коду", "DTO")
    Rel_D(outboundPublisher, sliceQueue, "Публікація зрізу", "RESP")
```

* **Аналіз архітектурної дисципліни шарів (Clean Architecture):**
  * **Inbound Layer:** Компоненти `AnalysisTaskConsumer`, `TaskPayloadValidator` та `GlobalErrorHandler` ізолюють домен від транспортного протоколу черги та забезпечують відмовостійкість за вимогою [[SRS-NFR-REL-01](../srs.md#3-non-functional-requirements)].
  * **Domain Core Layer:** Сервіси `SpoonAstModelBuilder`, `ControlFlowGraphBuilder`, `DataFlowGraphBuilder` та `CpgIntegrator` реалізують алгоритмічний розбір коду та базуються на рішенні [[ADR-001](./adr/0001-choice-of-technology.md)].
  * **Taint & Slicing Services:** Компоненти `TaintAnalyzer` та `ContextProgramSlicer` виконують детерміноване скорочення простору аналізу на 95–98% відповідно до вимоги [[SRS-FUN-05](../srs.md#2-functional-requirements)].

---

## 2.3. Документування архітектурних рішень (Architecture Decision Records)
Усі критичні архітектурні компроміси зафіксовані за міжнародним індустріальним стандартом:
* [ADR-001: Вибір технологічного стеку для синтаксичного розбору, побудови CPG та слайсингу Java-коду](./adr/0001-choice-of-technology.md) — офіційне рішення у статусі `Approved` (порівняльний аналіз Spoon, JavaParser та Joern).
