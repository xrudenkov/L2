%% Предпроцессинг транзакций POS -> Processing. Общая схема.
%% Файл содержит только Mermaid: вставьте его целиком в https://mermaid.live
%% Пояснения: preprocessing-architecture-notes.md. Схемы по слоям: diagrams/*.mmd
%% Обратный путь (шаги 6, 7, 8) нарисован линией без стрелки, направление показывает знак ◀ в подписи.
flowchart LR
    subgraph L1["1 · Терминалы (вне ЦОД)"]
        direction TB
        POS(["<b>POS-терминалы</b> ×100 000<br/>ISO 8583, TCP + TLS"])
        DNS["DNS<br/>имя входа → адрес"]
        TMS["Сервер настроек TMS<br/>таймауты, адреса, эхо"]
    end

    RT["<b>2 · Маршрутизаторы L3</b> ×2<br/>BGP, ECMP по потоку<br/>соединений не помнят"]
    GW["<b>3 · ISO-Gateway</b> ×8<br/>TLS, сокеты, sessionId<br/>F41 → кольцо → блок<br/>сам отвечает на эхо 0800"]

    subgraph CELL["4 · Блок b5 — один из 6: приложения в Kubernetes и своя БД"]
        direction TB
        TR["<b>Transaction Receiver</b><br/>проверки, MAC,<br/>ключ идемпотентности"]
        PS["<b>Processing Sender</b><br/>Submit, лимит,<br/>предохранитель, выбор A / B"]
        RR["<b>Result Receiver</b><br/>приём результата"]
        DLV["<b>Delivery</b><br/>подпись и доставка ответа"]
        TM["<b>Timeout Monitor</b><br/>сроки, GetStatus,<br/>автореверсы"]
        PG[("<b>5 · PostgreSQL блока</b><br/>primary + 2 реплики<br/>транзакции, состояния,<br/>ключи идемпотентности")]
    end

    HSM["<b>6 · HSM</b> ×7<br/>MAC, PIN, ключи"]
    CR["<b>8 · Callback Router</b> ×4<br/>correlationId → блок"]

    subgraph L9["9 · Процессинг (чёрный ящик)"]
        direction TB
        PA["<b>Процессинг A</b>"]
        PB["<b>Процессинг B</b>"]
    end

    subgraph SI["Стендины SI1, SI2"]
        direction TB
        SIAPP["Приложения стендина<br/>прогреты, без трафика"]
        SIPG[("Копии БД всех 6 блоков")]
    end

    subgraph CTL["Управление (вне пути транзакции)"]
        direction TB
        RC[("<b>etcd маршрутизации</b> ×5, Raft<br/>(Routing Config)<br/>кольцо, карта блоков,<br/>правила A / B, номер версии")]
        FO["<b>Failover Orchestrator</b> ×2<br/>перевод блока в стендин"]
        ADM["Инженер через конвейер<br/>кольцо, правила, миграция"]
    end

    subgraph L7["7 · Аудит и наблюдение"]
        direction TB
        AUD[["<b>Аудит</b><br/>Kafka → ClickHouse + WORM"]]
        MON["Логи, метрики, алерты"]
    end

    %% путь транзакции
    POS ==>|"① 0200"| RT
    RT ==>|"хеш соединения"| GW
    GW ==>|"② accept + sessionId"| TR
    TR ==>|"③ запись RECEIVED"| PG
    TR ==> PS
    PS ==>|"④ Submit → ACCEPTED"| PA
    PS -.->|"если A недоступен"| PB
    PA -.->|"⑤ Result"| CR
    PB -.-> CR
    RR ===|"◀ ⑥ в блок-владелец"| CR
    RR ==>|"запись итога"| PG
    RR ==> DLV
    GW ===|"◀ ⑦ deliver(sessionId)"| DLV
    POS ===|"◀ ⑧ 0210"| GW

    %% сроки и отмены
    TM --- PG
    TM -.->|"GetStatus, Reverse"| PA

    %% безопасность
    TR -->|"MAC, PIN"| HSM
    DLV -->|"MAC ответа"| HSM

    %% резерв и аудит
    PG -.->|"асинхронная репликация"| SIPG
    SIAPP --- SIPG
    PG -.->|"поток изменений"| AUD

    %% управление
    RC -.->|"подписка (watch):<br/>кольцо, карта блоков"| GW
    RC -.->|"подписка: карта блоков"| CR
    RC -.->|"подписка: кольцо"| TR
    FO -->|"запись: карта блоков"| RC
    ADM -->|"запись с проверкой"| RC
    FO -.->|"пробы"| PG

    %% настройки терминала
    POS -.-> DNS
    POS -.-> TMS

    classDef pos fill:#eef2f7,stroke:#5b6b7f,color:#1a1a1a
    classDef edge fill:#e8f4ea,stroke:#4f8a5b,color:#1a1a1a
    classDef cell fill:#e7effa,stroke:#4a6fa5,color:#1a1a1a
    classDef db fill:#f1eaf7,stroke:#7c5aa0,color:#1a1a1a
    classDef hsm fill:#fbe9e7,stroke:#b0524b,color:#1a1a1a
    classDef obs fill:#fff3dc,stroke:#b8860b,color:#1a1a1a
    classDef proc fill:#eeeeee,stroke:#555555,color:#1a1a1a
    classDef ctl fill:#fff8e1,stroke:#b8860b,color:#1a1a1a
    classDef etcd fill:#ffe9a8,stroke:#b8860b,stroke-width:3px,color:#1a1a1a
    class POS,DNS,TMS pos
    class RT,GW,CR edge
    class TR,PS,RR,DLV,TM,SIAPP cell
    class PG,SIPG db
    class HSM hsm
    class AUD,MON obs
    class PA,PB proc
    class FO,ADM ctl
    class RC etcd
    style CELL fill:#f5f8fd,stroke:#4a6fa5,stroke-width:2px
    style CTL fill:#fffcf0,stroke:#b8860b,stroke-dasharray:5 5
    style SI fill:#f5f8fd,stroke:#4a6fa5,stroke-dasharray:5 5
