# РОЗДІЛ 2. ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model: Levels 1–2)
Архітектурне проєктування системи виконано за індустріальною методологією Саймона Брауна (C4 Model) із застосуванням парадигми Diagrams-as-Code у середовищі Mermaid.js.

### 2.1.1. Рівень 1: Діаграма системного контексту (System Context Diagram)
Діаграма першого рівня визначає межі системи, ключові ролі користувачів (Person) та зовнішні системи (External Systems), з якими здійснюється мережева взаємодія.

```mermaid
flowchart TD
    subgraph Enterprise [" Організаційний периметр підприємства "]
        direction TB
        Scanner["<b>Hybrid SAST System</b><br/>[Software System]<br/>Гібридна система статичного аудиту безпеки коду Java"]
        Ollama["<b>Local LLM Inference Engine</b><br/>[External System]<br/>Локальний сервер Ollama (DeepSeek-Coder)"]
    end

    Dev["<b>Software Engineer</b><br/>[Person]<br/>Розробник Java-додатків"]
    Sec["<b>AppSec Auditor</b><br/>[Person]<br/>Інженер з аудиту безпеки коду"]
    GitHub["<b>GitHub / GitLab</b><br/>[External System]<br/>Зовнішній репозиторій вихідного коду"]

    Dev -->|HTTPS / Web Browser| Scanner
    Sec -->|HTTPS / Web Browser| Scanner
    Scanner -->|HTTP / REST порт 11434| Ollama
    Scanner -->|HTTPS / REST API v3| GitHub
```

* **Трасування вимог та інженерні протоколи (Cross-linking):**
  * Система **Hybrid SAST System** реалізує функціональну вимогу [[SRS-FUN-01](./srs.md#2-functional-requirements)] щодо прийому вихідного коду Java через веб-інтерфейс або посилання на GitHub.
  * Контейнер **Local LLM Inference Engine** забезпечує дотримання нефункціональної вимоги конфіденційності коду [[SRS-NFR-SEC-01](./srs.md#3-non-functional-requirements)] (100% On-Premises ізоляція без витоку коду на публічні хмарні платформи).
  * Інтеграція з **GitHub / GitLab** відповідає за автоматичне відкриття верифікованих Pull Request за вимогою [[SRS-FUN-08](./srs.md#2-functional-requirements)].

---

### 2.1.2. Рівень 2: Діаграма контейнерів (Container Diagram)
Діаграма другого рівня деталізує систему до рівня окремо розгорнутих та контейнеризованих сервісів (Docker Containers), баз даних та брокерів повідомлень.

```mermaid
flowchart TD
    User["<b>User / Security Engineer</b><br/>[Person]<br/>Розробник або аналітик безпеки"]

    subgraph Platform [" Контейнерний контур (Hybrid SAST Platform) "]
        direction TB
        SPA["<b>Web Frontend SPA</b><br/>[React 18, Vite, Cytoscape.js]<br/>Клієнтський дашборд: граф атаки, diff коду та створення PR"]
        Gateway["<b>API Gateway / Backend Core</b><br/>[Spring Boot 3, Java 21]<br/>Автентифікація, сесії та координація задач"]
        CPG["<b>CPG & Slicing Engine</b><br/>[Java 21, Spoon, JGraphT]<br/>Побудова AST/CFG/DFG та Taint Slicing"]
        Worker["<b>AI Remediation Worker</b><br/>[Java 21, LangChain4j]<br/>Інференс LLM та компіляторна валідація"]
        DB[("<b>PostgreSQL Database</b><br/>[PostgreSQL 16]<br/>Звіти аудитів, метадані та класифікатор CWE")]
        Redis[("<b>Redis Broker</b><br/>[Redis 7]<br/>Черги асинхронних завдань та кеш зрізів")]
    end

    LLM["<b>Local LLM Container</b><br/>[Ollama Engine]<br/>Локальний нейромережевий інференс"]
    GitHubExt["<b>GitHub API</b><br/>[REST API v3]<br/>Зовнішня система контролю версій"]

    User -->|HTTPS / Browser| SPA
    SPA -->|HTTPS / REST JSON| Gateway
    Gateway -->|JDBC / TCP порт 5432| DB
    Gateway -->|RESP / TCP порт 6379| Redis
    Redis -->|RESP / TCP| CPG
    CPG -->|RESP / TCP| Redis
    Redis -->|RESP / TCP| Worker
    Worker -->|HTTP / REST порт 11434| LLM
    Worker -->|HTTPS / REST API v3| GitHubExt
```

* **Трасування архітектурних рішень та вимог:**
  * Контейнер **CPG & Slicing Engine** спирається на прийняте та затверджене архітектурне рішення [[ADR-001](./adr/0001-choice-of-technology.md)] щодо використання бібліотек Spoon та JGraphT для розбору коду та побудови графа CPG.
  * Контейнер **AI Remediation Worker** виконує функціональну вимогу [[SRS-FUN-06](./srs.md#2-functional-requirements)] та реалізує пісочницю компіляторної валідації патчів за вимогою [[SRS-FUN-07](./srs.md#2-functional-requirements)].
  * Брокер **Redis Broker** забезпечує виконання вимог до продуктивності [[SRS-NFR-PERF-02](./srs.md#3-non-functional-requirements)] (через асинхронність) та відмовостійкості [[SRS-NFR-REL-01](./srs.md#3-non-functional-requirements)] (завдяки Dead Letter Queue).

---

## 2.2. Компонентне моделювання критичного сервісу (C4 Model Level 3: Component)
Для проєктування третього рівня декомпозовано найбільш обчислювально складний контейнер системи — **CPG & Slicing Engine**. Внутрішню архітектуру організовано за канонами Clean Architecture із суворою інверсією залежностей.

```mermaid
flowchart TD
    QueueIn[("<b>Redis Task Queue</b><br/>[Черга завдань аналізу]")]
    QueueOut[("<b>Redis Slice Queue</b><br/>[Черга зрізів для LLM]")]

    subgraph Engine [" CPG & Slicing Engine (Clean Architecture) "]
        direction TB
        
        subgraph Inbound [" Вхідний шар (Inbound Adapters) "]
            Consumer["<b>AnalysisTaskConsumer</b><br/>[Spring AMQP / Redis Listener]<br/>Прийом повідомлень на аудит"]
            Validator["<b>TaskPayloadValidator</b><br/>[Spring Validation]<br/>Валідація структури DTO"]
            Handler["<b>GlobalErrorHandler</b><br/>[Exception Handler]<br/>Обробка збоїв компілятора та відправка в DLQ"]
        end

        subgraph Application [" Шар застосунку (Application Services) "]
            Coordinator["<b>WorkflowCoordinator</b><br/>[Application Service]<br/>Координація етапів: Parse -> CPG -> Slice"]
        end

        subgraph Domain [" Доменне ядро (Domain Core Services) "]
            AstBuilder["<b>SpoonAstModelBuilder</b><br/>[Eclipse JDT Core]<br/>Синтаксичний розбір та типи"]
            CfgBuilder["<b>ControlFlowGraphBuilder</b><br/>[CFG Builder]<br/>Граф переходів та умов"]
            DfgBuilder["<b>DataFlowGraphBuilder</b><br/>[DFG Builder]<br/>Def-Use ланцюжки даних"]
            CpgIntegrator["<b>CpgIntegrator</b><br/>[JGraphT Integration]<br/>Синтез мультиграфа CPG"]
            TaintAnalyzer["<b>TaintAnalyzer</b><br/>[Taint Traversal]<br/>Пошук Source-to-Sink за CWE"]
            Slicer["<b>ContextProgramSlicer</b><br/>[Program Slicing]<br/>Вилучення зрізу 15-30 рядків"]
        end

        subgraph Outbound [" Вихідний шар (Outbound Adapters) "]
            Publisher["<b>SlicePublisher</b><br/>[Redis Adapter]<br/>Серіалізація та публікація зрізу"]
        end
    end

    QueueIn -->|RESP / TCP| Consumer
    Consumer -->|Java Call| Validator
    Validator -->|Method Call| Coordinator
    Coordinator -.->|Exception Handle| Handler

    Coordinator -->|Interface Call / DI| AstBuilder
    AstBuilder -->|AST Model| CfgBuilder
    AstBuilder -->|Symbol Table| DfgBuilder
    CfgBuilder -->|CFG Edges| CpgIntegrator
    DfgBuilder -->|DFG Edges| CpgIntegrator

    CpgIntegrator -->|CPG Multigraph| TaintAnalyzer
    TaintAnalyzer -->|Taint Path Coordinates| Slicer
    Slicer -->|Code Slice DTO| Publisher
    Publisher -->|RESP / TCP| QueueOut
```

* **Аналіз архітектурної дисципліни шарів (Clean Architecture):**
  * **Inbound Layer:** Компоненти `AnalysisTaskConsumer`, `TaskPayloadValidator` та `GlobalErrorHandler` повністю ізолюють внутрішню логіку від зовнішніх протоколів та забезпечують відмовостійкість за вимогою [[SRS-NFR-REL-01](./srs.md#3-non-functional-requirements)].
  * **Domain Core Layer:** Модулі `SpoonAstModelBuilder`, `ControlFlowGraphBuilder`, `DataFlowGraphBuilder`, `CpgIntegrator`, `TaintAnalyzer` та `ContextProgramSlicer` містять чисту бізнес-логіку і спираються на рішення [[ADR-001](./adr/0001-choice-of-technology.md)].
  * **Outbound Layer:** Адаптер `SlicePublisher` забезпечує слабку зв'язність (Loose Coupling) із наступним етапом обробки.

---

## 2.3. Документування архітектурних рішень (Architecture Decision Records)
Усі критичні архітектурні компроміси та обґрунтування технологічного вибору зафіксовані в офіційних документах:
* [ADR-001: Вибір технологічного стеку для синтаксичного розбору, побудови CPG та слайсингу Java-коду](./adr/0001-choice-of-technology.md) — рішення у статусі `Approved` (порівняння Spoon, JavaParser та Joern).
