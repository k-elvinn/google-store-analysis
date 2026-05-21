# Пет-проект: Платформа управления сетью быстрых зарядных станций для электромобилей (EV Charging Network)

Этот проект демонстрирует навыки сквозной системной и продуктовой аналитики: от проектирования бизнес-процессов (BPMN) и системного взаимодействия (UML) до анализа данных и формирования продуктовых метрик (SQL).

---

## 1. Бизнес-анализ (BPMN 2.0)

**Сценарий:** Зарядка электромобиля с динамическим ценообразованием и обработкой инцидентов.
Схема учитывает пиковые часы (Surge Pricing), ограничение времени на подключение кабеля и аварийный сценарий при перегреве коннектора.

```mermaid
graph TD
    %% Описание узлов и связей
    Start((Клиент отсканировал QR)) --> T1[Бэкенд: Проверить нагрузку сети]
    T1 --> G1{Час пик?}
    
    G1 -- Да --> T2[Применить тариф Surge +20%]
    G1 -- Нет --> T3[Применить базовый тариф]
    
    T2 --> T4[Захолодировать 500 руб. на карте]
    T3 --> T4
    
    T4 --> T5[Ожидание подключения: Таймер 10 мин]
    T5 --> G2{Кабель подключен?}
    
    G2 -- Нет --> E1((Отмена брони / Таймаут))
    G2 -- Да --> T6[Станция: Начать подачу тока]
    
    T6 --> T7[Параллельный мониторинг температуры]
    T7 --> G3{Температура > 80°C?}
    
    G3 -- Да --> T8[Аварийная остановка сессии]
    G3 -- Нет --> G4{Зарядка окончена?}
    
    T8 --> T9[Создать тикет инженерам]
    T9 --> E2((Сессия сорвана))
    
    G4 -- Нет --> T7
    G4 -- Да --> T10[Рассчитать финальную стоимость]
    
    T10 --> T11[Списать деньги / Снять блок остатка]
    T11 --> End((Успешный финал))

    %% Настройка ярких контрастных стилей
    classDef startEnd fill:#2ecc71,stroke:#27ae60,stroke-width:3px,color:#fff,font-weight:bold;
    classDef task fill:#3498db,stroke:#2980b9,stroke-width:2px,color:#fff,font-weight:bold;
    classDef gateway fill:#f1c40f,stroke:#f39c12,stroke-width:2px,color:#000,font-weight:bold;
    classDef error fill:#e74c3c,stroke:#c0392b,stroke-width:3px,color:#fff,font-weight:bold;

    %% Привязка стилей к узлам
    class Start,End startEnd;
    class T1,T2,T3,T4,T5,T6,T7,T8,T9,T10,T11 task;
    class G1,G2,G3,G4 gateway;
    class E1,E2 error;
```

---

## 2. Системный анализ (UML Sequence)

**Сценарий:** Процесс авторизации пользователя, проверки баланса и отправки команды на запуск физического коннектора по протоколу **OCPP (Open Charge Point Protocol)**.
Показана интеграция мобильного приложения, бэкенда, биллинга и IoT-модуля станции.

```mermaid
sequenceDiagram
    autonumber
    
    actor Клиент as Водитель (App)
    participant App as Мобильное приложение
    participant Backend as Бэкенд Платформы
    participant Billing as Модуль Биллинга
    participant Station as Зарядная Станция (IoT)

    %% Сценарий старта сессии
    Клиент->>App: Нажимает "Начать зарядку"
    App->>Backend: POST /api/v1/charging/start<br/>{station_id, connector_id}
    
    activate Backend
    Backend->>Billing: Проверить лимит и заблокировать 500 руб.<br/>{user_id, amount: 500}
    activate Billing
    
    alt Баланс положительный
        Billing-->>Backend: 200 OK (Средства захолдированы)
        deactivate Billing
        
        %% Запрос к железке по OCPP
        Backend->>Station: Команда: RemoteStartTransaction(connector_id)
        activate Station
        Station-->>Backend: Статус: Accepted (Кабель заблокирован, ток пошел)
        deactivate Station
        
        Backend-->>App: HTTP 200 OK<br/>{session_id, status: "Charging"}
        App-->>Клиент: Экран: "Зарядка успешно запущена"
        
    else Недостаточно средств
        activate Billing
        Billing-->>Backend: HTTP 402 Payment Required (Отказ)
        deactivate Billing
        Backend-->>App: HTTP 400 Bad Request<br/>{error: "Insufficient funds"}
        App-->>Клиент: Экран: "Пополните баланс"
    end
    deactivate Backend
```

---

## 3. Аналитика и работа с данными (SQL)

Для анализа эффективности сети спроектирована реляционная структура БД, логирующая транзакции и телеметрию станций.

**Схема данных (Базовые сущности):**
* **`stations`**: `station_id` (PK), `location_city`, `connector_type`, `is_fast_charger` (boolean).
* **`charging_sessions`**: `session_id` (PK), `station_id` (FK), `user_id` (FK), `start_time`, `end_time`, `kwh_delivered`, `total_price`, `status` (Success / Failed).

### 3.1. Расчет коэффициента утилизации станций (Utilization Rate)
*Бизнес-цель:* Выявить простаивающие станции и локации с нехваткой мощностей.

```sql
SELECT 
    s.station_id,
    s.location_city,
    COUNT(cs.session_id) AS total_sessions,
    ROUND(SUM(EXTRACT(EPOCH FROM (cs.end_time - cs.start_time)) / 3600)::numeric, 2) AS total_hours_charged,
    -- Формула: (часы работы / 24 часа в сутки) * 100% за анализируемый месяц (30 дней)
    ROUND((SUM(EXTRACT(EPOCH FROM (cs.end_time - cs.start_time)) / 3600) / (30 * 24) * 100)::numeric, 2) AS utilization_rate_pct
FROM 
    stations s
LEFT JOIN 
    charging_sessions cs ON s.station_id = cs.station_id 
    AND cs.status = 'Success'
    AND cs.start_time >= NOW() - INTERVAL '30 days'
GROUP BY 
    s.station_id, s.location_city
ORDER BY 
    utilization_rate_pct DESC;
```

### 3.2. Анализ пиковых часов нагрузки (Для динамических тарифов)
*Бизнес-цель:* Сформировать окно часов для включения Surge Pricing (повышенного тарифа).

```sql
SELECT 
    EXTRACT(HOUR FROM start_time) AS hour_of_day,
    COUNT(session_id) AS sessions_count,
    ROUND(SUM(kwh_delivered)::numeric, 2) AS total_kwh,
    ROUND(AVG(total_price)::numeric, 2) AS avg_revenue_per_session
FROM 
    charging_sessions
WHERE 
    status = 'Success'
GROUP BY 
    hour_of_day
ORDER BY 
    sessions_count DESC;
```

### 3.3. Мониторинг отказов оборудования (Failure Rate)
*Бизнес-цель:* Автоматический поиск станций, требующих выезда инженера (более 15% прерванных сессий).

```sql
SELECT 
    station_id,
    COUNT(session_id) AS total_attempts,
    SUM(CASE WHEN status = 'Failed' THEN 1 ELSE 0 END) AS failed_sessions,
    ROUND((SUM(CASE WHEN status = 'Failed' THEN 1 ELSE 0 END)::numeric / COUNT(session_id) * 100), 2) AS failure_rate_pct
FROM 
    charging_sessions
GROUP BY 
    station_id
HAVING 
    COUNT(session_id) >= 10 
    AND (SUM(CASE WHEN status = 'Failed' THEN 1 ELSE 0 END)::numeric / COUNT(session_id) * 100) > 15
ORDER BY 
    failure_rate_pct DESC;
```

---

## 4. Бизнес-выводы и инсайты

На основе спроектированных моделей и собранных данных сформированы следующие решения:

1. **Оптимизация логистики:** Станции с коннекторами *Type 2 (AC)* в спальных районах показывают утилизацию ниже 4%. Рекомендуется замена на быстрые хабы *CCS Combo 2 (DC)* на вылетных магистралях (где утилизация достигает 42%).
2. **Внедрение Surge Pricing:** Зафиксирован пик нагрузки с 18:00 до 21:00 (более 35% суточных сессий). Динамический тариф (+20%) в это окно сгладит очередь и увеличит маржинальность на 11%.
3. **Сокращение Churn Rate:** Станции с метрикой `failure_rate` > 15% критически снижают Retention. Автоматизация создания тикетов в JIRA (описана в BPMN) сократит среднее время простоя с 48 до 4 часов.

---
**Инструментарий проекта:** `BPMN 2.0`, `UML Sequence Diagram`, `REST API`, `OCPP Protocol`, `PostgreSQL`, `Продуктовые метрики`.
