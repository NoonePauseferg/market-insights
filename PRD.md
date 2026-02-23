# PRD: AlphaLab — Automated Trading Research Platform

**Version:** 1.3 (Python-only, CPU-first, Apple M4)  
**Date:** 2026-02-23  
**Status:** Approved — Ready for Implementation  
**Author:** [Owner]

---

## 1. Executive Summary

AlphaLab — платформа для автоматизированного поиска, генерации и тестирования торговых гипотез на рынках криптовалют (CEX) и российском фондовом рынке (MOEX: акции, облигации, фьючерсы/опционы FORTS). Система собирает максимально широкий спектр данных (OHLCV, order book, on-chain, деривативы, новости/сентимент), применяет ML-модели для автоматической генерации гипотез, бэктестирует их и доставляет результаты через интерактивного Telegram-бота. Пользователь торгует руками на основе полученных инсайтов. Бюджет на платные источники данных: $0 — используются только бесплатные API (Binance, Bybit, MOEX ISS, Tinkoff Invest API).

**Ключевой принцип: Data Quality First** — система бесполезна без высококачественных данных. Каждый источник данных проходит валидацию, нормализацию и оценку качества до попадания в аналитический pipeline.

---

## 2. Problem Statement

Трейдер имеет доступ к множеству рынков и данных, но:

- Ручной анализ не масштабируется — невозможно отслеживать сотни инструментов и тысячи паттернов одновременно.
- Большинство идей остаются непроверенными из-за отсутствия инфраструктуры для быстрого бэктестирования.
- Данные из разных источников разрозненны, имеют разные форматы и качество.
- Нет системного подхода к записи, приоритизации и тестированию гипотез.
- ML-модели требуют регулярного обучения и переобучения на свежих данных.

---

## 3. Goals & Non-Goals

### Goals

- **G1**: Создать единое хранилище исторических и real-time данных высокого качества для крипты и акций США.
- **G2**: Автоматически генерировать торговые гипотезы на основе ML и статистических методов.
- **G3**: Обеспечить быстрый и достоверный бэктест любой гипотезы (автоматической или пользовательской).
- **G4**: Доставлять результаты и инсайты через интерактивного Telegram-бота с командами, статусами и графиками.
- **G5**: Хранить все данные, результаты и метаданные в базе данных (не в файлах).
- **G6**: Обеспечить переобучение ML-моделей по расписанию и по триггеру.

### Non-Goals (v1)

- Автоматическое исполнение ордеров (auto-trading). Архитектура должна позволить это в будущем, но v1 — research only.
- Web UI / dashboard (Telegram-бот покрывает все потребности v1).
- Мультипользовательский режим — система для одного оператора.
- GPU-зависимые модели (Transformers, full NLP fine-tune) — отложены. CPU-first стратегия.

---

## 4. User Stories

| ID | Story | Priority |
|---|---|---|
| US-1 | Как трейдер, я хочу видеть автоматически найденные аномалии и паттерны, чтобы быстрее принимать решения. | P0 |
| US-2 | Как трейдер, я хочу отправить свою гипотезу через Telegram и получить результат бэктеста. | P0 |
| US-3 | Как трейдер, я хочу получать push-уведомления, когда система находит сильный сигнал. | P0 |
| US-4 | Как трейдер, я хочу видеть графики equity curve и распределение P&L прямо в Telegram. | P1 |
| US-5 | Как трейдер, я хочу мониторить статус сбора данных и качество данных через бота. | P1 |
| US-6 | Как трейдер, я хочу управлять очередью гипотез (приоритет, отмена, пересчёт). | P1 |
| US-7 | Как трейдер, я хочу чтобы ML-модели переобучались на свежих данных автоматически. | P1 |
| US-8 | Как трейдер, я хочу видеть историю всех протестированных гипотез и их результаты. | P2 |

---

## 5. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        TELEGRAM BOT                              │
│  Commands · Status · Charts · Hypothesis Input · Notifications   │
└──────────────────────────────┬──────────────────────────────────┘
                               │ gRPC / REST
┌──────────────────────────────▼──────────────────────────────────┐
│                       API GATEWAY (FastAPI)                       │
│         Auth · Rate Limit · Request Routing · WebSocket           │
└──┬──────────┬──────────┬──────────┬──────────┬─────────────────┘
   │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼
┌──────┐ ┌──────┐ ┌──────────┐ ┌──────┐ ┌────────────┐
│ Data │ │Hypo- │ │Backtest  │ │  ML  │ │ Notification│
│Ingest│ │thesis│ │ Engine   │ │Pipe- │ │  Service    │
│      │ │Engine│ │          │ │line  │ │             │
└──┬───┘ └──┬───┘ └────┬─────┘ └──┬───┘ └──────┬─────┘
   │        │          │          │             │
   ▼        ▼          ▼          ▼             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MESSAGE QUEUE (Redis Streams)               │
│              Task Queue · Event Bus · Pub/Sub                    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                       STORAGE LAYER                              │
│  TimescaleDB (time-series) · PostgreSQL (metadata) · Redis      │
└─────────────────────────────────────────────────────────────────┘
```

### Принципы архитектуры

- **100% Python**: единый язык для всего стека. Numpy/Numba для performance, asyncio для concurrency.
- **Модульность**: каждый сервис — отдельный процесс, общение через очередь сообщений.
- **Data Quality Gate**: данные не попадают в аналитику без прохождения валидации.
- **Idempotency**: любой pipeline можно перезапустить без дублирования данных.
- **Observability**: метрики, логи и health-checks для каждого компонента.

---

## 6. Data Layer — Sources, Quality & Storage

### 6.1 Data Sources (Free-Only, $0 Budget)

> **Constraint:** Все источники данных — бесплатные. Это даёт отличное покрытие для крипты, но ограничивает глубину данных по акциям США и on-chain аналитике. Архитектура позволяет подключить платные источники позже, если появится бюджет.

#### Crypto (CEX) — ОТЛИЧНОЕ покрытие

Binance и Bybit предоставляют практически все нужные данные бесплатно через REST API и WebSocket.

| Data Type | Source | Frequency | Rate Limits | Priority |
|---|---|---|---|---|
| OHLCV (свечи) | Binance API, Bybit API, CCXT (unified) | 1m, 5m, 15m, 1h, 4h, 1d | Binance: 1200 req/min, Bybit: 120 req/min | P0 |
| Order Book (L2) | Binance WS `depth20@100ms`, Bybit WS | Real-time snapshot 100ms–1s | WS: без ограничений (до 1024 streams) | P0 |
| Trades (tick data) | Binance WS `aggTrade`, Bybit WS | Real-time | WS: без ограничений | P1 |
| Funding Rate | Binance Futures `/fapi/v1/fundingRate`, Bybit | 8h (historical), real-time (WS) | Included in general limits | P0 |
| Open Interest | Binance `/fapi/v1/openInterest`, Bybit | 5m polling | Included in general limits | P0 |
| Liquidations | Binance WS `forceOrder`, Bybit WS | Real-time | WS: без ограничений | P1 |
| Long/Short Ratio | Binance `/futures/data/globalLongShortAccountRatio` | 5m | Included in general limits | P2 |
| Taker Buy/Sell Volume | Binance `/futures/data/takerlongshortRatio` | 5m | Included in general limits | P2 |

**Library:** CCXT (Python) — единый интерфейс к 100+ биржам. Позволяет добавлять OKX, Kraken и другие без переписывания кода.

#### Crypto (On-chain) — DIY-подход

Без Glassnode/Nansen собираем данные напрямую из блокчейна и бесплатных агрегаторов.

| Data Type | Source | Frequency | Limits | Priority |
|---|---|---|---|---|
| Whale transactions (ETH/ERC-20) | Etherscan API (free key) | Polling 15s | 5 calls/sec | P1 |
| Whale transactions (BTC) | Blockchain.com API, Mempool.space | Polling 30s | Generous | P1 |
| Exchange inflow/outflow (approx.) | Etherscan (known exchange wallets list) + DIY aggregation | 1m | 5 calls/sec | P1 |
| DeFi TVL, yields, DEX volumes | DeFiLlama API (fully free, no key) | 5m | No hard limit (be respectful) | P1 |
| Gas prices, network activity | Etherscan API, Blocknative free tier | 1m | 5 calls/sec | P2 |
| Token holder distribution | Etherscan token API | 1h | 5 calls/sec | P2 |
| Dune Analytics queries | Dune API free tier | On-demand | 10 queries/month, community queries unlimited viewing | P2 |

**DIY Exchange Flow Tracking:**
Список известных кошельков бирж (Binance, Coinbase, Kraken и т.д.) — открытый; можно отслеживать крупные переводы через Etherscan API. Менее точно, чем Nansen, но бесплатно и работает.

**Ограничение:** Etherscan free tier (5 calls/sec) — bottleneck. Для масштабирования можно запустить собственную Ethereum-ноду (бесплатно, но требует ~2TB SSD и время на синхронизацию).

#### Российский рынок (MOEX) — ОТЛИЧНОЕ покрытие

Два бесплатных источника покрывают 95% потребностей: MOEX ISS API (без авторизации) + Tinkoff Invest API (бесплатно с брокерским счётом).

**MOEX ISS API (iss.moex.com) — исторические данные, справочники, аналитика:**

| Data Type | Endpoint | Frequency | Limits | Priority |
|---|---|---|---|---|
| OHLCV акции (TQBR) | `/engines/stock/markets/shares/securities/{ticker}/candles` | 1m, 10m, 1h, 1d, 1w, 1M | ~50 req/sec, без ключа | P0 |
| OHLCV облигации (ОФЗ: TQOB, корп: TQCB) | `/engines/stock/markets/bonds/securities/{ticker}/candles` | 1m, 10m, 1h, 1d | Same | P0 |
| OHLCV фьючерсы (FORTS) | `/engines/futures/markets/forts/securities/{ticker}/candles` | 1m, 10m, 1h, 1d | Same | P0 |
| Order Book snapshot | `/engines/stock/markets/shares/securities/{ticker}/orderbook` | По запросу (top 20 levels) | Same | P1 |
| Текущие котировки (all tickers) | `/engines/stock/markets/shares/securities.json` | Polling 1-5s | Same | P0 |
| Индексы (IMOEX, RGBI, RTS) | `/engines/stock/markets/index/securities` | Real-time | Same | P0 |
| Облигации: купоны, доходность, НКД, дюрация | `/securities/{ticker}/bondization` + marketdata | По запросу | Same | P0 |
| Фьючерсы: OI, расчётная цена | `/engines/futures/markets/forts/securities` (marketdata) | 5m polling | Same | P1 |
| Опционы: страйки, объёмы | `/engines/futures/markets/options/securities` | 5m polling | Same | P2 |
| Дивиденды, корпоративные действия | `/securities/{ticker}/dividends` | Event-driven | Same | P1 |
| Справочник инструментов | `/securities.json` | 1d | Same | P0 |

**Tinkoff Invest API (invest-public-api.tinkoff.ru) — real-time streaming:**

| Data Type | Method | Frequency | Limits | Priority |
|---|---|---|---|---|
| Streaming свечи | `MarketDataStream.SubscribeCandles` | Real-time (1m, 5m, 15m, 1h) | 300 req/min unary; streaming unlimited | P0 |
| Streaming Order Book | `MarketDataStream.SubscribeOrderBook` | Real-time (до 50 levels!) | Same | P0 |
| Streaming Trades (сделки) | `MarketDataStream.SubscribeTrades` | Real-time tick | Same | P1 |
| Streaming Last Price | `MarketDataStream.SubscribeLastPrice` | Real-time | Same | P0 |
| Исторические свечи | `MarketData.GetCandles` | 1m (до 1 дня назад), 1d (без ограничений) | Same | P0 |
| Стакан snapshot | `MarketData.GetOrderBook` | По запросу (до 50 levels) | Same | P1 |
| Инструменты (metadata) | `Instruments.*` | По запросу | Same | P0 |
| Торговый статус | `MarketData.GetTradingStatus` | По запросу | Same | P1 |

**Python SDK:** `pip install tinkoff-investments` — официальный, gRPC-based, хорошо документирован.

**Стратегия по источникам:**
- **MOEX ISS** = primary для исторических данных (глубже история, нет ограничений по давности для дневных свечей).
- **Tinkoff API** = primary для real-time streaming (order book 50 levels, trades, candles).
- Cross-validation: сверяем данные двух источников для quality scoring.

**Специфика облигаций (отдельный pipeline):**

| Metric | Source | Notes |
|---|---|---|
| Кривая доходности ОФЗ | MOEX ISS (RGBI index + individual OFZ yields) | Строим сами из отдельных выпусков |
| G-spread (корп. vs ОФЗ) | Рассчитываем: корп. yield − OFZ yield с matching duration | Сигнал кредитного риска |
| НКД (accumulated coupon income) | MOEX ISS bondization endpoint | Для точного расчёта P&L |
| Дюрация, convexity | MOEX ISS / рассчитываем | Для процентного риска |
| Ключевая ставка ЦБ РФ | CBR API (cbr.ru) | Главный драйвер облигаций |
| Купонный календарь | MOEX ISS bondization | Для cash flow анализа |

**Специфика FORTS (фьючерсы/опционы):**

| Metric | Source | Notes |
|---|---|---|
| Open Interest | MOEX ISS marketdata | Ключевой индикатор |
| Базис (фьючерс vs спот) | Рассчитываем | Арбитражные возможности |
| Volatility smile (опционы) | MOEX ISS + расчёт IV через Black-Scholes | P2, сложная реализация |
| Экспирации | MOEX ISS справочник | Roll-over сигналы |

#### Sentiment & News — ХОРОШЕЕ покрытие

**Крипта:**

| Data Type | Source | Frequency | Limits | Priority |
|---|---|---|---|---|
| Crypto news + sentiment | CryptoPanic API (free tier) | 1m polling | Free: basic filtering | P1 |
| Reddit sentiment | Reddit API (OAuth, free) | 5m polling | 100 requests/min | P1 |
| Reddit (r/cryptocurrency, r/CryptoMarkets) | Reddit API | 5m | Same | P1 |
| Fear & Greed Index (crypto) | Alternative.me API | 1d | Free, no key | P1 |
| Google Trends | pytrends (unofficial) | 1h | Throttling possible | P2 |

**Российский рынок (бесплатно, несложно):**

| Data Type | Source | Frequency | Limits | Priority |
|---|---|---|---|---|
| Smart-Lab посты и комментарии | Web scraping (BeautifulSoup / Scrapy) | 15m | Respectful scraping, ~1 req/s | P2 |
| Новости РБК, Интерфакс | RSS feeds (бесплатно) | 5m polling | Unlimited | P1 |
| ТАСС экономика | RSS feed | 5m polling | Unlimited | P2 |
| Telegram-каналы (публичные) | Telethon (бесплатная библиотека) | Real-time | Требует Telegram аккаунт | P2 |
| Решения ЦБ РФ | CBR RSS / scraping cbr.ru | Event-driven | N/A | P1 |

**Рекомендуемые TG-каналы для мониторинга:** @raborynokda, @cbrstocks, @markettwits, @AK47pfl — можно добавлять/убирать через конфиг.

**NLP Pipeline для Sentiment:**
RSS, Reddit и Smart-Lab → raw текст → FinBERT для английского, ruBERT (DeepPavlov) для русского → sentiment score [-1.0, 1.0]. Обе модели open-source, бесплатные, запускаются локально: quantized inference на Apple M4 MPS (быстро) или CPU (чуть медленнее, но достаточно). GPU не требуется для inference.

#### Macro — ОТЛИЧНОЕ покрытие

| Data Type | Source | Frequency | Limits | Priority |
|---|---|---|---|---|
| Ключевая ставка ЦБ РФ | CBR API (cbr.ru/DailyInfoWebServ) | Event-driven + 1d polling | Free | P0 |
| Курс USD/RUB, EUR/RUB | MOEX ISS (CETS), ЦБ РФ | Real-time (MOEX) / 1d (CBR) | Free | P0 |
| Инфляция РФ (ИПЦ) | Росстат / CBR | Monthly | Free | P1 |
| ОФЗ yield curve | Рассчитываем из MOEX данных | 1d | N/A | P0 |
| DXY, US Treasury yields | FRED API (free, key required) | 1d | 120 req/min | P1 |
| Нефть Brent | MOEX ISS (FORTS: BR futures) или yfinance (BZ=F) | Real-time / 1d | Free | P0 |
| Корреляции BTC/IMOEX/RUB/Brent | Рассчитываем внутри | 1d | N/A | P2 |

#### Сводка: покрытие данных по рынкам

| Capability | Crypto (CEX) | MOEX (Акции) | MOEX (Облигации) | MOEX (FORTS) |
|---|---|---|---|---|
| OHLCV (daily) | ✅ Отличное | ✅ Отличное (ISS + Tinkoff) | ✅ Отличное | ✅ Отличное |
| OHLCV (intraday) | ✅ Отличное (real-time) | ✅ Отличное (Tinkoff streaming) | ✅ Хорошее | ✅ Хорошее |
| Order Book | ✅ Отличное (real-time WS) | ✅ Отличное (Tinkoff 50 levels!) | ⚠️ Ограниченное (низкая ликвидность) | ✅ Хорошее |
| Funding/OI/Liquidations | ✅ Отличное | N/A | N/A | ✅ OI через ISS |
| On-chain | ⚠️ DIY, рабочее | N/A | N/A | N/A |
| Sentiment | ✅ Хорошее | ⚠️ Базовое (RSS + Smart-Lab) | N/A | N/A |
| Fundamentals | N/A | ✅ Дивиденды (ISS) | ✅ Купоны, доходность, НКД | ✅ Базис, экспирации |
| Macro | ✅ Отличное (FRED) | ✅ Отличное (CBR + MOEX) | ✅ Ключевая ставка = main driver | ✅ Нефть, валюта |

**Вердикт:** Покрытие данных для MOEX — ОТЛИЧНОЕ, лучше чем бесплатные данные по акциям США. Tinkoff API даёт real-time стриминг с 50-уровневым стаканом, MOEX ISS — глубокую историю без ограничений. Оба источника официальные и стабильные.

### 6.2 Data Quality Framework

Это критически важная подсистема. ML на мусорных данных = мусорные результаты.

#### Ingestion Validation Pipeline

Каждый поступивший батч данных проходит последовательность проверок:

```
Raw Data → Schema Validation → Gap Detection → Outlier Detection  
         → Cross-source Reconciliation → Normalization → Quality Score  
         → Validated Storage
```

**Schema Validation:**
- Проверка типов данных (float, int, timestamp).
- Проверка обязательных полей.
- Проверка границ (цена > 0, volume >= 0, timestamp в разумном диапазоне).
- Проверка формата timestamps (UTC, единый формат).

**Gap Detection:**
- Обнаружение пропущенных свечей / периодов.
- Классификация гэпов: ожидаемый (выходные для акций), неожиданный (сбой API), нулевой объём.
- Автоматический backfill из альтернативного источника или интерполяция с пометкой `interpolated=true`.
- Алерт в Telegram, если гэп превышает пороговое значение.

**Outlier Detection:**
- Z-score и IQR-фильтры для цен и объёмов.
- Сравнение с данными из альтернативных источников.
- Аномальные значения помечаются флагом `is_outlier`, но не удаляются — решение за аналитическим слоем.
- Flash crash / flash spike detection с отдельной логикой.

**Cross-source Reconciliation:**
- Для каждого инструмента минимум 2 источника данных.
- При расхождении цены закрытия > 0.1% — пометка `reconciliation_conflict`.
- Выбор primary / secondary источника по reliability-рейтингу.

**Quality Score (0–100):**
Каждый батч данных получает оценку по формуле:

```
QS = w1 * completeness + w2 * timeliness + w3 * accuracy + w4 * consistency
```

Где:
- `completeness` — доля непропущенных записей.
- `timeliness` — задержка относительно ожидаемого времени.
- `accuracy` — совпадение с альтернативными источниками.
- `consistency` — отсутствие внутренних противоречий.

Данные с QS < 60 — не используются в ML/backtest (карантин).  
Данные с QS 60–80 — используются с пометкой `low_quality`.  
Данные с QS > 80 — полноценное использование.

#### Data Freshness Monitoring

Каждый источник данных имеет SLA по свежести. При нарушении — алерт в Telegram и автоматический переключатель на fallback-источник.

| Source Type | Expected Lag | Alert Threshold | Notes |
|---|---|---|---|
| Crypto OHLCV 1m | < 5s | > 30s | Binance/Bybit — reliable |
| Crypto Order Book | < 1s | > 5s | WebSocket — reliable |
| Funding Rate | < 1m | > 10m | Exchange API — reliable |
| MOEX OHLCV (Tinkoff stream) | < 2s | > 15s | gRPC streaming — reliable |
| MOEX Order Book (Tinkoff) | < 1s | > 5s | gRPC streaming — reliable |
| MOEX Historical (ISS) | < 30s | > 5m | REST polling — reliable |
| News/Sentiment (RSS + Reddit) | < 5m | > 20m | Polling-based, varies |
| On-chain (Etherscan) | < 30s | > 5m | Rate limited at 5 req/s |
| Macro (CBR, FRED) | < 1h after release | > 6h | Updated infrequently |

### 6.3 Storage

#### TimescaleDB (time-series data)

Основное хранилище для всех временных рядов. Выбран вместо InfluxDB из-за:
- SQL-совместимость (PostgreSQL extension).
- Hypertables с автоматическим partitioning по времени.
- Compression (10–20x для старых данных).
- Continuous aggregates для предрассчитанных таймфреймов.

**Ключевые таблицы:**

```sql
-- OHLCV данные (hypertable, partitioned by time)
CREATE TABLE ohlcv (
    time         TIMESTAMPTZ NOT NULL,
    exchange     TEXT NOT NULL,         -- 'binance', 'bybit', 'moex', 'tinkoff'
    symbol       TEXT NOT NULL,         -- 'BTC/USDT', 'SBER', 'SU26238RMFS4'
    market_type  TEXT NOT NULL,         -- 'crypto_spot', 'crypto_futures', 'moex_stock', 'moex_bond', 'moex_futures', 'moex_option', 'moex_index'
    timeframe    TEXT NOT NULL,         -- '1m', '5m', '1h', '1d'
    open         DOUBLE PRECISION,
    high         DOUBLE PRECISION,
    low          DOUBLE PRECISION,
    close        DOUBLE PRECISION,
    volume       DOUBLE PRECISION,
    quote_volume DOUBLE PRECISION,
    trades_count INTEGER,
    -- качество
    source       TEXT,                  -- primary source name
    quality_score SMALLINT,            -- 0-100
    is_interpolated BOOLEAN DEFAULT FALSE,
    is_outlier   BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (time, exchange, symbol, market_type, timeframe)
);
SELECT create_hypertable('ohlcv', 'time');

-- Order Book Snapshots
CREATE TABLE order_book_snapshot (
    time       TIMESTAMPTZ NOT NULL,
    exchange   TEXT NOT NULL,
    symbol     TEXT NOT NULL,
    depth      SMALLINT NOT NULL,       -- top-N levels
    bids       JSONB NOT NULL,          -- [[price, qty], ...]
    asks       JSONB NOT NULL,
    bid_total  DOUBLE PRECISION,
    ask_total  DOUBLE PRECISION,
    spread     DOUBLE PRECISION,
    quality_score SMALLINT
);
SELECT create_hypertable('order_book_snapshot', 'time');

-- Funding Rate
CREATE TABLE funding_rate (
    time         TIMESTAMPTZ NOT NULL,
    exchange     TEXT NOT NULL,
    symbol       TEXT NOT NULL,
    funding_rate DOUBLE PRECISION,
    next_funding TIMESTAMPTZ,
    quality_score SMALLINT
);
SELECT create_hypertable('funding_rate', 'time');

-- Open Interest
CREATE TABLE open_interest (
    time          TIMESTAMPTZ NOT NULL,
    exchange      TEXT NOT NULL,
    symbol        TEXT NOT NULL,
    open_interest DOUBLE PRECISION,
    oi_value_usd  DOUBLE PRECISION,
    quality_score SMALLINT
);
SELECT create_hypertable('open_interest', 'time');

-- Sentiment Events
CREATE TABLE sentiment_event (
    time       TIMESTAMPTZ NOT NULL,
    source     TEXT NOT NULL,           -- 'twitter', 'reddit', 'cryptopanic'
    symbol     TEXT,                    -- NULL if general market
    title      TEXT,
    content    TEXT,
    sentiment_score DOUBLE PRECISION,   -- -1.0 to 1.0
    impact_score    DOUBLE PRECISION,   -- 0.0 to 1.0
    url        TEXT,
    raw_data   JSONB,
    quality_score SMALLINT
);
SELECT create_hypertable('sentiment_event', 'time');

-- On-chain Metrics
CREATE TABLE onchain_metric (
    time        TIMESTAMPTZ NOT NULL,
    chain       TEXT NOT NULL,          -- 'ethereum', 'bitcoin'
    metric_name TEXT NOT NULL,          -- 'exchange_inflow', 'whale_transfers'
    value       DOUBLE PRECISION,
    metadata    JSONB,
    source      TEXT,
    quality_score SMALLINT
);
SELECT create_hypertable('onchain_metric', 'time');

-- Macro Indicators
CREATE TABLE macro_indicator (
    time            TIMESTAMPTZ NOT NULL,
    indicator_name  TEXT NOT NULL,      -- 'DXY', 'US10Y', 'CPI', 'CBR_KEY_RATE', 'RU_CPI'
    value           DOUBLE PRECISION,
    previous_value  DOUBLE PRECISION,
    source          TEXT
);
SELECT create_hypertable('macro_indicator', 'time');

-- Данные по облигациям (MOEX-специфичное)
CREATE TABLE bond_data (
    time            TIMESTAMPTZ NOT NULL,
    isin            TEXT NOT NULL,
    ticker          TEXT NOT NULL,
    bond_type       TEXT NOT NULL,          -- 'ofz', 'corporate', 'municipal'
    price_pct       DOUBLE PRECISION,       -- % от номинала
    yield_to_maturity DOUBLE PRECISION,     -- YTM
    yield_current   DOUBLE PRECISION,
    duration        DOUBLE PRECISION,       -- дюрация Маколея
    modified_duration DOUBLE PRECISION,
    nkd             DOUBLE PRECISION,       -- НКД (accumulated coupon income)
    coupon_rate     DOUBLE PRECISION,
    next_coupon_date DATE,
    maturity_date   DATE,
    spread_to_ofz   DOUBLE PRECISION,       -- G-spread, рассчитываем
    volume          DOUBLE PRECISION,
    quality_score   SMALLINT
);
SELECT create_hypertable('bond_data', 'time');

-- Кривая доходности ОФЗ (snapshot)
CREATE TABLE ofz_yield_curve (
    time            TIMESTAMPTZ NOT NULL,
    tenor_years     DOUBLE PRECISION NOT NULL,  -- 0.25, 0.5, 1, 2, 3, 5, 7, 10, 15, 20
    yield           DOUBLE PRECISION NOT NULL,
    ofz_ticker      TEXT                         -- какой выпуск использовался
);
SELECT create_hypertable('ofz_yield_curve', 'time');

-- FORTS деривативы
CREATE TABLE forts_derivative (
    time            TIMESTAMPTZ NOT NULL,
    ticker          TEXT NOT NULL,              -- 'SiZ4', 'RIZ4'
    underlying      TEXT NOT NULL,              -- 'USD/RUB', 'RTS', 'BRENT'
    instrument_type TEXT NOT NULL,              -- 'futures', 'option'
    settlement_price DOUBLE PRECISION,
    open_interest   DOUBLE PRECISION,
    basis           DOUBLE PRECISION,           -- futures - spot price
    basis_pct       DOUBLE PRECISION,
    expiration_date DATE,
    -- для опционов
    strike          DOUBLE PRECISION,
    option_type     TEXT,                        -- 'call', 'put'
    implied_vol     DOUBLE PRECISION,
    volume          DOUBLE PRECISION,
    quality_score   SMALLINT
);
SELECT create_hypertable('forts_derivative', 'time');

-- Курсы валют (MOEX + CBR)
CREATE TABLE fx_rate (
    time            TIMESTAMPTZ NOT NULL,
    pair            TEXT NOT NULL,              -- 'USD/RUB', 'EUR/RUB', 'CNY/RUB'
    rate            DOUBLE PRECISION NOT NULL,
    source          TEXT NOT NULL,              -- 'moex_cets', 'cbr_official'
    volume          DOUBLE PRECISION
);
SELECT create_hypertable('fx_rate', 'time');
```

**Continuous Aggregates (предрассчитанные данные):**

```sql
-- Автоматическая агрегация 1m -> 5m, 15m, 1h, 4h, 1d
CREATE MATERIALIZED VIEW ohlcv_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS time,
    exchange, symbol, market_type,
    first(open, time) AS open,
    max(high) AS high,
    min(low) AS low,
    last(close, time) AS close,
    sum(volume) AS volume,
    sum(quote_volume) AS quote_volume,
    sum(trades_count) AS trades_count
FROM ohlcv
WHERE timeframe = '1m'
GROUP BY time_bucket('1 hour', time), exchange, symbol, market_type;
```

#### PostgreSQL (metadata & application state)

```sql
-- Гипотезы
CREATE TABLE hypothesis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now(),
    source          TEXT NOT NULL,          -- 'auto_ml', 'auto_statistical', 'manual'
    status          TEXT NOT NULL DEFAULT 'pending',  
                    -- pending → queued → running → completed → archived
    priority        INTEGER DEFAULT 5,      -- 1 (highest) - 10 (lowest)
    
    -- описание гипотезы
    name            TEXT NOT NULL,
    description     TEXT,
    strategy_type   TEXT,                   -- 'mean_reversion', 'momentum', 'sentiment', ...
    market_type     TEXT NOT NULL,          -- 'crypto', 'moex_stock', 'moex_bond', 'moex_futures'
    symbols         TEXT[],                 -- ['BTC/USDT', 'ETH/USDT']
    timeframe       TEXT,                   -- '1h', '4h', '1d'
    
    -- параметры стратегии (гибкий формат)
    params          JSONB NOT NULL,
    
    -- результаты бэктеста
    backtest_result JSONB,                 -- sharpe, max_dd, total_return, etc.
    equity_curve    BYTEA,                 -- сериализованная картинка или данные
    
    -- ML metadata
    model_id        UUID REFERENCES ml_model(id),
    feature_importance JSONB,
    
    -- пользовательский фидбэк
    user_rating     SMALLINT,              -- 1-5
    user_notes      TEXT
);

CREATE INDEX idx_hypothesis_status ON hypothesis(status);
CREATE INDEX idx_hypothesis_source ON hypothesis(source);
CREATE INDEX idx_hypothesis_created ON hypothesis(created_at DESC);

-- ML модели
CREATE TABLE ml_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ DEFAULT now(),
    name            TEXT NOT NULL,
    model_type      TEXT NOT NULL,          -- 'xgboost', 'lstm', 'transformer', ...
    version         INTEGER NOT NULL,
    status          TEXT NOT NULL,          -- 'training', 'active', 'retired'
    
    -- training metadata
    training_started  TIMESTAMPTZ,
    training_finished TIMESTAMPTZ,
    training_data_range TSTZRANGE,
    features_used   TEXT[],
    hyperparams     JSONB,
    
    -- performance
    metrics         JSONB,                 -- accuracy, precision, recall, sharpe, etc.
    artifact_path   TEXT,                  -- path to serialized model weights
    
    -- lineage
    parent_model_id UUID REFERENCES ml_model(id)
);

-- Результаты бэктестов (детальные)
CREATE TABLE backtest_run (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hypothesis_id   UUID REFERENCES hypothesis(id),
    started_at      TIMESTAMPTZ DEFAULT now(),
    finished_at     TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'running',
    
    -- параметры запуска
    data_range      TSTZRANGE,
    initial_capital DOUBLE PRECISION DEFAULT 10000,
    commission_model TEXT DEFAULT 'per_trade',  -- 'per_trade', 'maker_taker', 'tiered'
    commission_rate  DOUBLE PRECISION DEFAULT 0.003, -- 0.3% Tinkoff Investor; 0.001 for Binance
    slippage_model  TEXT DEFAULT 'fixed',
    
    -- результаты
    total_return_pct    DOUBLE PRECISION,
    sharpe_ratio        DOUBLE PRECISION,
    sortino_ratio       DOUBLE PRECISION,
    max_drawdown_pct    DOUBLE PRECISION,
    max_drawdown_duration INTERVAL,
    win_rate            DOUBLE PRECISION,
    profit_factor       DOUBLE PRECISION,
    total_trades        INTEGER,
    avg_trade_duration  INTERVAL,
    
    -- детали
    trades          JSONB,                 -- array of individual trades
    equity_curve    JSONB,                 -- time-series of portfolio value
    monthly_returns JSONB,
    
    -- validation
    walk_forward    BOOLEAN DEFAULT FALSE,
    oos_sharpe      DOUBLE PRECISION,      -- out-of-sample Sharpe
    oos_return_pct  DOUBLE PRECISION
);

-- Очередь задач
CREATE TABLE task_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ DEFAULT now(),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    task_type       TEXT NOT NULL,          -- 'backtest', 'ml_train', 'data_backfill', ...
    status          TEXT NOT NULL DEFAULT 'pending',
    priority        INTEGER DEFAULT 5,
    payload         JSONB NOT NULL,
    result          JSONB,
    error_message   TEXT,
    retries         INTEGER DEFAULT 0,
    max_retries     INTEGER DEFAULT 3
);

-- Data Quality Log
CREATE TABLE data_quality_log (
    id              BIGSERIAL PRIMARY KEY,
    time            TIMESTAMPTZ DEFAULT now(),
    source          TEXT NOT NULL,
    check_type      TEXT NOT NULL,         -- 'gap', 'outlier', 'stale', 'reconciliation'
    severity        TEXT NOT NULL,         -- 'info', 'warning', 'critical'
    symbol          TEXT,
    details         JSONB,
    resolved        BOOLEAN DEFAULT FALSE,
    resolved_at     TIMESTAMPTZ
);

-- Алерты и уведомления
CREATE TABLE notification_log (
    id              BIGSERIAL PRIMARY KEY,
    sent_at         TIMESTAMPTZ DEFAULT now(),
    channel         TEXT DEFAULT 'telegram',
    notification_type TEXT NOT NULL,       -- 'signal', 'backtest_result', 'data_alert', 'system'
    payload         JSONB NOT NULL,
    delivered       BOOLEAN DEFAULT FALSE
);
```

#### Redis

- **Task Queue**: Redis Streams для распределения задач между воркерами.
- **Cache**: Последний order book snapshot, текущие цены, сессии бота.
- **Pub/Sub**: Real-time events (new signal, backtest completed, data alert).
- **Rate Limiting**: Контроль запросов к внешним API.

---

## 7. Core Modules

### 7.1 Data Ingestion Service

**Язык:** Python (asyncio).

**Компоненты:**
- **WebSocket Connectors** (Python asyncio + `websockets`): подключение к Binance, Bybit WS для order book, trades, liquidations. Каждый коннектор — asyncio task с reconnect logic.
- **Tinkoff gRPC Stream** (Python `tinkoff-investments`): real-time candles, order book, trades для MOEX инструментов.
- **REST Pollers** (Python `asyncio` + `aiohttp`): периодический polling MOEX ISS, CryptoPanic, Reddit, Etherscan, FRED, CBR.
- **Data Validation Worker** (Python): принимает сырые данные из очереди, прогоняет через Quality Pipeline, записывает в TimescaleDB через `asyncpg`.
- **Backfill Manager** (Python): заполняет исторические данные (3 года) при первом запуске или после обнаружения гэпов.

**Конфигурация коннекторов (пример):**

```yaml
connectors:
  binance_spot_ws:
    type: websocket
    library: websockets          # Python asyncio
    endpoints:
      - wss://stream.binance.com:9443/ws
    symbols: dynamic_top_50_by_volume  # пересчёт ежедневно в 00:00 UTC
    channels: ["aggTrade", "depth20@100ms"]
    reconnect_delay: 5s
    max_reconnect_attempts: -1  # infinite

  binance_ohlcv_rest:
    type: rest_poller
    library: aiohttp             # Python asyncio
    base_url: https://api.binance.com/api/v3
    symbols: dynamic_top_50_by_volume
    timeframes: ["1m", "5m", "15m", "1h", "4h", "1d"]
    poll_interval: 60s
    rate_limit: 1200/min

  moex_iss_historical:
    type: rest_poller
    language: python
    base_url: https://iss.moex.com/iss
    instruments:
      stocks: imoex_constituents     # весь IMOEX (~40 тикеров), обновление ежеквартально
      bonds_ofz: all_liquid          # все ликвидные ОФЗ
      bonds_corp: top_20_by_volume   # top-20 корпоративных по обороту, пересчёт ежемесячно
      futures: ["SiZ4", "RIZ4", "BRZ4"]  # Si, RTS, Brent (ролл при экспирации)
      indices: ["IMOEX", "RGBI", "RTSI"]
    timeframes: ["1m", "10m", "1h", "1d"]
    history_depth: 3y               # backfill за 3 года
    poll_interval: 60s
    rate_limit: 50/sec              # generous, no API key needed

  tinkoff_streaming:
    type: grpc_stream
    language: python
    library: tinkoff-investments
    endpoint: invest-public-api.tinkoff.ru:443
    mode: production                 # real-time данные, бесплатно с брокерским счётом
    streams:
      - candles: imoex_constituents  # real-time 1m candles для всего IMOEX
      - orderbook: imoex_top_10     # 50-level order book для top-10 по ликвидности
      - trades: imoex_top_10        # tick data для top-10
      - last_price: ["*"]           # все отслеживаемые инструменты
    trading_session: main            # 10:00–18:50 MSK (вечерняя — позже)
    token_env: TINKOFF_INVEST_TOKEN  # production token из ЛК Тинькофф

  cryptopanic_news:
    type: rest_poller
    language: python
    base_url: https://cryptopanic.com/api/v1
    poll_interval: 60s
    filter: "hot"
```

### 7.2 Feature Engineering Service

**Язык:** Python (pandas, numpy, ta-lib). Polars опционально для тяжёлых агрегаций (отлично работает на Apple M4 ARM).

Вычисляет фичи из сырых данных для ML-моделей и стратегий. Фичи хранятся в отдельных таблицах TimescaleDB.

**Категории фич:**

| Category | Examples |
|---|---|
| Price-based | SMA, EMA, Bollinger Bands, ATR, RSI, MACD, Stochastic |
| Volume-based | OBV, VWAP, Volume Profile, Volume-weighted RSI |
| Order Book | Bid-ask imbalance, depth ratio, spread dynamics, wall detection |
| Microstructure | Trade flow imbalance, Kyle's lambda, Amihud illiquidity |
| On-chain (crypto) | Exchange netflow z-score, whale accumulation rate, NVT signal |
| Sentiment | Weighted sentiment score, sentiment momentum, news impact decay |
| Cross-asset | BTC-IMOEX correlation (rolling), Brent-RUB beta, MOEX-crypto divergence |
| Regime | Volatility regime (HMM), trend strength (ADX), market phase classification |
| Bonds (MOEX) | Yield curve slope, G-spread z-score, duration-adjusted carry, CBR rate delta |
| FORTS | Futures basis (annualized), OI change momentum, put-call ratio, roll yield |
| Macro (RU) | Key rate trajectory, RUB volatility, oil-RUB correlation, CPI momentum |

**Feature Store Design:**

```sql
CREATE TABLE feature_store (
    time        TIMESTAMPTZ NOT NULL,
    symbol      TEXT NOT NULL,
    timeframe   TEXT NOT NULL,
    features    JSONB NOT NULL,         -- {"rsi_14": 45.2, "bb_width": 0.05, ...}
    version     INTEGER NOT NULL DEFAULT 1
);
SELECT create_hypertable('feature_store', 'time');
```

Feature versioning — при изменении логики расчёта фичи, version инкрементируется, и старые данные пересчитываются.

### 7.3 Hypothesis Engine

**Язык:** Python.

Два режима работы:

#### Auto-generation

Система автоматически генерирует гипотезы по расписанию (например, каждые 6 часов):

**Statistical Scanner:**
- Поиск аномальных корреляций между фичами и будущим return.
- Обнаружение структурных breakpoints (regime changes).
- Поиск повторяющихся паттернов (сезонность, day-of-week effects).
- Cointegration tests для пар (pair trading candidates).

**ML-based Generator:**
- Модель обучается предсказывать direction / magnitude return на горизонте N свечей.
- Feature importance из модели → новые гипотезы ("RSI + OI divergence → reversal").
- SHAP values для интерпретации — какие комбинации фич дают сильнейший сигнал.
- Anomaly detection (Isolation Forest, Autoencoders) — поиск необычных рыночных состояний.

**Hypothesis Template System:**

```python
# Пример шаблона: крипта
class MeanReversionTemplate:
    name = "Mean Reversion on {indicator} for {symbol}"
    params_grid = {
        "indicator": ["rsi_14", "bb_percentb", "zscore_20"],
        "entry_threshold": [20, 25, 30],
        "exit_threshold": [50, 55, 60],
        "stop_loss_pct": [2, 3, 5],
        "timeframe": ["1h", "4h"],
    }
    symbols = "dynamic_top_50_binance"  # auto-resolved

# Пример шаблона: MOEX акции
class MOEXMomentumTemplate:
    name = "Momentum {indicator} for {symbol} (MOEX)"
    params_grid = {
        "indicator": ["rsi_14", "macd_cross", "ema_cross_9_21"],
        "volume_filter": [1.5, 2.0],  # volume > N * SMA(volume, 20)
        "timeframe": ["1h", "1d"],
    }
    symbols = "imoex_constituents"  # auto-resolved (~40 тикеров)

# Пример шаблона: ОФЗ / облигации
class OFZYieldCurveTemplate:
    name = "OFZ Yield Curve {strategy} trade"
    params_grid = {
        "strategy": ["steepener", "flattener", "butterfly"],
        "trigger": ["cbr_rate_change", "spread_zscore", "curve_slope_ma"],
        "holding_period_days": [30, 60, 90],
    }

# Пример шаблона: кросс-рыночный
class CryptoMOEXCorrelationTemplate:
    name = "BTC-IMOEX divergence trade"
    params_grid = {
        "correlation_window": [30, 60, 90],
        "divergence_threshold": [1.5, 2.0, 2.5],  # z-score
        "trade_side": ["crypto", "moex", "both"],
    }
```

Комбинаторный взрыв контролируется через:
- Pre-screening: быстрый грубый бэктест (только Sharpe > 0 проходит дальше).
- Deduplication: не тестируем стратегии, слишком похожие на уже протестированные.
- Budget: максимум N гипотез в очереди одновременно.

#### Manual Input (через Telegram)

Пользователь может отправить гипотезу в формате:

```
/hypothesis
name: BTC RSI divergence long
symbol: BTC/USDT
timeframe: 4h
type: mean_reversion
entry: RSI(14) < 30 AND price > SMA(200)
exit: RSI(14) > 60
stop_loss: 3%
take_profit: 8%
```

```
/hypothesis
name: SBER gap momentum
symbol: SBER
market: moex_stock
timeframe: 1h
type: momentum
entry: GAP_UP > 1% AND VOLUME > AVG(VOLUME,20) * 2
exit: TIME = 18:30 OR CLOSE < ENTRY * 0.985
```

Бот парсит, валидирует и ставит в очередь на тестирование.

### 7.4 Backtesting Engine

**Язык:** Python (numpy-vectorized core, Numba JIT для inner loops).

**Принципы бэктеста:**

- **Event-driven архитектура**: каждая свеча / тик обрабатывается последовательно.
- **Realistic simulation**: комиссии (MOEX: 0.3% Тинькофф «Инвестор»; крипта: 0.1% Binance spot), slippage, partial fills.
- **No look-ahead bias**: строгий контроль, что стратегия видит только прошлые данные.
- **Walk-forward validation**: train на 70%, validate на 15%, test на 15%.
- **MOEX-aware**: учёт торговых часов основной сессии (10:00–18:50 MSK), выходных, праздников, утренних гэпов. Вечерняя сессия (19:00–23:50) — отложена на v2.
- **Bond-aware**: расчёт P&L с учётом НКД, купонных выплат, амортизации.

**Метрики результата:**

```python
@dataclass
class BacktestResult:
    # Return metrics
    total_return_pct: float
    annualized_return_pct: float
    
    # Risk metrics
    sharpe_ratio: float
    sortino_ratio: float
    calmar_ratio: float
    max_drawdown_pct: float
    max_drawdown_duration: timedelta
    
    # Trade metrics
    total_trades: int
    win_rate: float
    profit_factor: float
    avg_win_pct: float
    avg_loss_pct: float
    avg_trade_duration: timedelta
    
    # Robustness
    oos_sharpe: float            # Out-of-sample
    monte_carlo_p5_sharpe: float # 5th percentile from Monte Carlo
    parameter_sensitivity: dict  # Sharpe variance across param grid
    
    # Data
    equity_curve: list[tuple[datetime, float]]
    monthly_returns: list[tuple[str, float]]
    trades: list[Trade]
```

**Anti-overfitting measures:**
- Walk-forward analysis (rolling window).
- Множественный тест (Bonferroni / FDR correction для p-values).
- Monte Carlo permutation test — перетасовываем returns и сравниваем.
- Parameter sensitivity: если Sharpe падает > 50% при ±10% параметра — гипотеза хрупкая.
- Minimum 100 trades для статистической значимости.

### 7.5 ML Pipeline

**Язык:** Python (scikit-learn, XGBoost, LightGBM — CPU; PyTorch MPS — inference на M4).

**Compute Strategy:**

> **CPU-first подход.** Для большинства predicting-алгоритмов (XGBoost, LightGBM, Random Forest, логистическая регрессия, HMM) CPU более чем достаточно — эти модели и так CPU-native и не выигрывают от GPU. GPU в облаке (A100/H100) есть, но использование проблематично (задержки, стоимость, DevOps overhead). Локальная машина — Apple M4, который отлично справляется с inference NLP-моделей через PyTorch MPS backend.

**Архитектура pipeline:**

```
Feature Store → Feature Selection → Train/Val/Test Split  
              → Model Training (CPU) → Hyperparameter Tuning (CPU, Optuna)  
              → Evaluation → Model Registry → Deployment (local M4)  
              → Monitoring → Retrain Trigger
```

**Модели (по приоритету):**

| Model | Use Case | Compute | Retraining |
|---|---|---|---|
| XGBoost / LightGBM | Direction prediction, feature screening | CPU (native, быстро) | Weekly |
| Random Forest / Extra Trees | Ensemble baseline, feature importance | CPU | Weekly |
| Logistic Regression + feature crosses | Fast baseline, interpretable signals | CPU (секунды) | Daily |
| FinBERT (ProsusAI/finbert, quantized) | Sentiment scoring for English news/Reddit | M4 MPS inference; CPU fine-tune | Monthly |
| ruBERT (DeepPavlov, quantized) | Sentiment scoring for Russian news/Smart-Lab/TG | M4 MPS inference; CPU fine-tune | Monthly |
| Hidden Markov Model (hmmlearn) | Market regime classification | CPU (native) | Weekly |
| Isolation Forest | Anomaly detection, regime change | CPU (native) | Daily |
| LSTM / GRU (small, <1M params) | Sequence patterns, volatility prediction | M4 MPS or CPU | Bi-weekly |
| Autoencoder (small) | Feature compression, anomaly detection | M4 MPS or CPU | Bi-weekly |

**Отложено до появления стабильного GPU-доступа:**

| Model | Use Case | Why Deferred |
|---|---|---|
| Transformer (multi-asset attention) | Cross-market signals | Требует GPU для training; inference на M4 возможен для small models |
| Large NLP fine-tuning | Domain-specific sentiment | Full fine-tune FinBERT/ruBERT требует GPU; quantized inference на CPU/MPS достаточно |

**NLP Inference на Apple M4:**

```python
import torch

# Apple MPS backend для PyTorch — ускоряет inference NLP-моделей
device = torch.device("mps") if torch.backends.mps.is_available() else torch.device("cpu")

# Quantized модели для эффективного inference
from transformers import AutoModelForSequenceClassification
model = AutoModelForSequenceClassification.from_pretrained(
    "ProsusAI/finbert", 
    torch_dtype=torch.float16  # half-precision на M4
).to(device)
```

**Важно:** FinBERT и ruBERT для inference (scoring текстов) работают быстро на CPU/MPS. Полный fine-tune — при необходимости разово в облаке, затем inference локально.

**Retraining Logic:**

```python
class RetrainingScheduler:
    def should_retrain(self, model: MLModel) -> bool:
        # Scheduled retrain
        if now() - model.last_trained > model.retrain_interval:
            return True
        
        # Performance degradation
        recent_metrics = model.get_recent_metrics(window='7d')
        if recent_metrics.sharpe < model.baseline_sharpe * 0.7:
            return True
        
        # Data drift detection (PSI > threshold)
        if model.population_stability_index() > 0.2:
            return True
        
        return False
```

### 7.6 Signal Generation Service

**Язык:** Python.

Агрегирует выходы ML-моделей, результаты активных стратегий и ad-hoc аномалии в единый поток сигналов.

**Signal Schema:**

```python
@dataclass
class Signal:
    id: str
    timestamp: datetime
    symbol: str
    direction: str          # 'long', 'short', 'neutral'
    confidence: float       # 0.0 - 1.0
    source: str             # 'ml_xgboost_v3', 'hypothesis_42', 'anomaly_detector'
    timeframe: str
    entry_price: float | None
    stop_loss: float | None
    take_profit: float | None
    reasoning: str          # Human-readable explanation
    features_snapshot: dict # Key features at signal time
    expires_at: datetime    # Signal validity window
```

**Signal Filtering:**
- Минимальный confidence threshold (configurable, default 0.65).
- Cooldown: не больше 1 сигнала на один символ за N минут.
- Conflict resolution: если long и short сигналы одновременно — notify, но не рекомендовать.
- Regime filter: не генерировать momentum-сигналы в range-bound regime.

### 7.7 Telegram Bot

**Язык:** Python (aiogram 3.x).

#### Команды

| Command | Description |
|---|---|
| `/start` | Welcome + quick setup |
| `/status` | Общий статус системы (data freshness, queue, active models) |
| `/signals` | Последние N сигналов с фильтрацией |
| `/signal <id>` | Детали конкретного сигнала с графиком |
| `/hypothesis <description>` | Отправить новую гипотезу на тестирование |
| `/queue` | Просмотр очереди гипотез |
| `/queue cancel <id>` | Отменить гипотезу |
| `/queue priority <id> <1-10>` | Изменить приоритет |
| `/results` | Последние результаты бэктестов |
| `/result <id>` | Детальный результат с equity curve |
| `/models` | Статус ML-моделей (version, metrics, next retrain) |
| `/retrain <model>` | Триггернуть переобучение модели |
| `/data` | Статус данных (freshness, quality, gaps) |
| `/data gaps` | Текущие гэпы в данных |
| `/watchlist` | Управление списком отслеживаемых инструментов |
| `/settings` | Настройки уведомлений и порогов |
| `/help` | Справка по всем командам |

#### Push-уведомления

Бот проактивно отправляет сообщения при:

- **Новый сигнал** (confidence > threshold): символ, направление, reasoning, confidence bar.
- **Backtest завершён**: краткая сводка (Sharpe, return, max DD) + ссылка на детали.
- **Data alert**: проблема с качеством данных или свежестью.
- **Model alert**: деградация модели, начало/окончание retraining.
- **System alert**: ошибки, restarted services.

#### Графики в Telegram

Генерация через `matplotlib` / `plotly` → PNG → отправка как фото.

Типы графиков:
- Equity curve с benchmark.
- P&L distribution (histogram).
- Monthly returns heatmap.
- Signal confidence timeline.
- Feature importance bar chart.
- Data quality dashboard (sparklines).

#### Inline Keyboards

Каждое уведомление о сигнале содержит кнопки:
- `📊 Details` — развёрнутая информация.
- `👍 / 👎` — фидбэк (сохраняется для улучшения моделей).
- `🔇 Mute symbol` — замьютить сигналы по этому символу на 24ч.

---

## 8. Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| **Data Ingestion (real-time)** | asyncio + websockets / aiohttp | Crypto WS, Tinkoff gRPC |
| **Data Ingestion (polling)** | asyncio + aiohttp | MOEX ISS, Reddit, RSS, Etherscan, FRED, CBR |
| **API Gateway** | FastAPI | Внутренний API между сервисами |
| **Backtest Engine** | Custom event-driven (numpy-vectorized) | Критичные циклы через numpy/numba |
| **ML Pipeline** | XGBoost, LightGBM, scikit-learn, hmmlearn | CPU-native, быстро |
| **Deep Learning (inference)** | PyTorch (MPS backend) | M4 Neural Engine для NLP inference |
| **NLP / Sentiment** | FinBERT (EN), ruBERT/DeepPavlov (RU) — quantized | HuggingFace Transformers, half-precision |
| **Feature Engineering** | pandas, numpy, ta-lib | Polars для тяжёлых агрегаций (опционально) |
| **Telegram Bot** | aiogram 3.x | Async-native |
| **Message Queue** | Redis Streams | Через redis-py / aioredis |
| **Time-series DB** | TimescaleDB | asyncpg для async доступа |
| **Metadata DB** | PostgreSQL 16 | asyncpg / SQLAlchemy 2.0 async |
| **Cache** | Redis 7 | aioredis |
| **Scheduler** | APScheduler | Async-compatible |
| **Charts** | matplotlib, plotly | PNG export для Telegram |
| **Containerization** | Docker Compose | Всё Python — единый base image |
| **Monitoring** | Prometheus client + Grafana (optional v1) | prometheus-client lib |

**Язык: 100% Python.** Весь стек на Python 3.11+ с asyncio. Для performance-критичных участков (бэктест inner loop, feature computation) используем numpy vectorization и Numba JIT. Compute strategy: CPU-first (XGBoost/LightGBM — CPU-native и быстрые), Apple M4 MPS для NLP inference, cloud GPU — только при крайней необходимости.

---

## 9. Non-Functional Requirements

| Requirement | Target |
|---|---|
| OHLCV ingestion latency (1m candle) | < 5 seconds after candle close |
| Order book snapshot latency | < 500ms |
| Backtest speed (1 year, 1h candles, 1 symbol) | < 30 seconds |
| Backtest speed (1 year, 1m candles, 1 symbol) | < 5 minutes |
| Hypothesis queue throughput | ≥ 50 backtests / hour |
| ML model retraining time | < 5 min (XGBoost/LightGBM on CPU), < 30 min (small LSTM on M4 MPS) |
| Telegram response time | < 3 seconds for commands |
| Data storage retention | Unlimited (with TimescaleDB compression for data > 6 months) |
| System uptime | 99% (допустимы maintenance windows) |
| Data quality monitoring | 100% incoming data passes validation pipeline |

---

## 10. Phased Roadmap

### Phase 1 — Foundation (Weeks 1–4)

**Goal:** Надёжный сбор данных + хранилище + базовый бот.

- [ ] Развернуть PostgreSQL + TimescaleDB + Redis (Docker Compose).
- [ ] Реализовать OHLCV ingestion для Binance (spot + futures), MOEX ISS и Tinkoff streaming.
- [ ] Tinkoff Invest API: подключить gRPC streaming (candles, order book, trades).
- [ ] Data validation pipeline (schema, gaps, outliers).
- [ ] Quality score calculation.
- [ ] Backfill исторических данных: 3 года (1d для всех; 1h для top-20; 1m — последние 6 месяцев).
- [ ] Telegram bot: `/start`, `/status`, `/data`, базовые уведомления.
- [ ] Data freshness monitoring + alerts.

**Deliverable:** Система собирает и валидирует OHLCV для 20+ крипто-пар + 15+ тикеров MOEX (акции, ОФЗ, фьючерсы) с QS > 80.

### Phase 2 — Backtest + Manual Hypotheses (Weeks 5–8)

**Goal:** Пользователь может отправлять и тестировать свои гипотезы.

- [ ] Backtesting engine (event-driven, walk-forward).
- [ ] Hypothesis queue (submit, cancel, prioritize).
- [ ] Telegram: `/hypothesis`, `/queue`, `/results`, `/result <id>`.
- [ ] Chart generation (equity curve, P&L distribution).
- [ ] Anti-overfitting measures (Monte Carlo, parameter sensitivity).
- [ ] Feature engineering service (основные технические индикаторы).

**Deliverable:** Полный цикл: идея → бэктест → результат в Telegram за < 5 минут.

### Phase 3 — Extended Data + Feature Store (Weeks 9–12)

**Goal:** Максимум данных для ML.

- [ ] Order book ingestion (asyncio WebSocket для крипты; Tinkoff gRPC для MOEX).
- [ ] Crypto: funding rate, OI, liquidations.
- [ ] MOEX: облигации pipeline (yield curve, НКД, duration, G-spread).
- [ ] MOEX: FORTS деривативы (OI, базис, roll yield).
- [ ] Macro: CBR ключевая ставка, USD/RUB, нефть Brent.
- [ ] News/sentiment pipeline (CryptoPanic, Reddit, RSS, Smart-Lab scraping).
- [ ] On-chain data (Etherscan + DeFiLlama).
- [ ] Feature store с versioning.
- [ ] Cross-source reconciliation (MOEX ISS vs Tinkoff).

**Deliverable:** 60+ фич в feature store (включая bond-specific и FORTS-specific), все с quality scoring.

### Phase 4 — ML + Auto-generation (Weeks 13–18)

**Goal:** Система сама находит идеи.

- [ ] ML pipeline: training (CPU), evaluation, model registry.
- [ ] XGBoost/LightGBM для direction prediction (CPU-native, быстрый retrain).
- [ ] Statistical scanner (correlation anomalies, cointegration).
- [ ] Auto-hypothesis generation с pre-screening.
- [ ] Retraining scheduler (time-based + drift-based).
- [ ] NLP inference pipeline: quantized FinBERT + ruBERT на M4 MPS (или CPU в Docker).
- [ ] Signal generation service.
- [ ] Telegram: `/signals`, `/models`, `/retrain`.
- [ ] User feedback loop (👍/👎 → model improvement).

**Deliverable:** Автоматическая генерация ≥ 10 гипотез/день, ≥ 3 сигнала/день с confidence > 0.65. Все модели работают на CPU/M4 без GPU.

### Phase 5 — Advanced Models + Polish (Weeks 19–24)

**Goal:** Продвинутые модели (CPU-compatible) + оптимизация.

- [ ] Small LSTM/GRU для sequence prediction (M4 MPS inference, CPU training).
- [ ] Anomaly detection (Isolation Forest, small Autoencoders).
- [ ] Regime detection (HMM — hmmlearn, CPU-native).
- [ ] Ensemble methods: stacking XGBoost + LightGBM + LR.
- [ ] Performance optimization (Numba JIT для backtest hot paths, Polars для тяжёлых агрегаций).
- [ ] Prometheus + Grafana monitoring (optional).
- [ ] Comprehensive documentation.
- [ ] **Deferred (если появится стабильный GPU-доступ):** Transformer для multi-asset attention, full NLP fine-tuning.

**Deliverable:** Полноценная research-платформа с 5+ ML-моделями, все работают на CPU/M4.

---

## 11. Risks & Mitigations

| Risk | Impact | Probability | Mitigation |
|---|---|---|---|
| API rate limits / bans | Нет данных | Medium | Несколько API keys, fallback sources, respectful polling, exponential backoff |
| MOEX ISS API изменения | Потеря исторических данных | Low | ISS стабилен годами; fallback на Tinkoff API; кэширование исторических данных |
| Tinkoff API breaking changes | Потеря real-time стриминга | Low | Официальный SDK, обновляется; fallback на MOEX ISS polling |
| Overfitting ML models | Ложные сигналы | High | Walk-forward, Monte Carlo, minimum trade count, parameter sensitivity |
| Etherscan rate limit (5/s) | Медленный on-chain pipeline | Medium | Batching requests, caching, опционально собственная нода |
| Низкая ликвидность (MOEX 2nd tier) | Нереалистичные бэктесты | Medium | Slippage model с учётом стакана; фильтр по ликвидности (min avg volume) |
| Survivorship bias | Искажённые результаты | Medium | Включить делистинг в исторические данные (MOEX ISS хранит) |
| Look-ahead bias | Невалидные бэктесты | Medium | Strict point-in-time data access, code review |
| MOEX trading hours (10:00–18:50 MSK) | Гэпы на открытии, ночной риск | Medium | Gap-aware стратегии; overnight risk модель; учёт в бэктесте |
| Санкционные риски (брокер/MOEX) | Потеря доступа к торгам | Low | Мониторинг новостей; данные уже в локальной БД |
| Cloud GPU доступ проблематичен | Не обучить большие Transformer-модели | Medium | CPU-first стратегия (XGBoost/LightGBM); quantized NLP inference на M4 MPS; cloud GPU разово для fine-tune если нужно |
| TimescaleDB performance | Медленные запросы | Low | Proper indexing, compression, continuous aggregates |
| Telegram API limits | Задержки уведомлений | Low | Message batching, priority queue for alerts |

---

## 12. Success Metrics

| Metric | Target (Month 3) | Target (Month 6) |
|---|---|---|
| Data sources connected | ≥ 10 | ≥ 20 |
| Average Data Quality Score | > 80 | > 85 |
| Data freshness SLA compliance | > 95% | > 99% |
| Hypotheses tested per week | ≥ 50 | ≥ 200 |
| ML models in production | 2 (CPU-native) | 5+ (CPU-native + small LSTM on MPS) |
| Signals generated per day | ≥ 3 | ≥ 10 |
| Signal precision (user-rated) | > 50% | > 60% |
| Backtest queue latency (p95) | < 10 min | < 5 min |
| System uptime | > 95% | > 99% |

---

## 13. Finalized Decisions

Все открытые вопросы закрыты:

| # | Question | Decision |
|---|---|---|
| 1 | Symbols universe (крипта) | Динамический top-50 by volume на Binance. Пересчёт списка ежедневно; вход/выход из universe логируется. |
| 2 | Symbols universe (MOEX) | Весь индекс IMOEX (~40 тикеров). Состав обновляется при ребалансировке индекса (ежеквартально). |
| 3 | Облигации scope | ОФЗ (все ликвидные выпуски) + корпоративные top-20 по обороту. Пересчёт top-20 ежемесячно. |
| 4 | GPU / Compute | Локальная машина Apple M4. GPU (A100/H100) есть в облаке, но использование проблематично. Для большинства predicting-алгоритмов CPU достаточно. M4 Neural Engine / MPS — для inference NLP-моделей. |
| 5 | Глубина истории | 3 года (MOEX с ~2023, крипта с ~2023). При необходимости — backfill глубже. |
| 6 | Tinkoff API токен | Production (real-time данные, бесплатно). Получить в ЛК Тинькофф → Настройки → Токен для API. |
| 7 | Вечерняя сессия MOEX | Отложена. Архитектура поддерживает, но v1 — только основная сессия (10:00–18:50 MSK). |
| 8 | Комиссии в бэктесте | MOEX: 0.3% за сделку (тариф «Инвестор» Тинькофф). Крипта: Binance spot 0.1% maker/taker (default). |
| 9 | TG-каналы для sentiment | Рекомендованные: @raborynokda, @cbrstocks, @markettwits, @AK47pfl. Добавление/удаление через конфиг. |

---

## 14. Key Parameters Summary

Быстрая справка по всем финализированным параметрам:

```yaml
# === MARKETS ===
crypto:
  exchange: Binance (primary), Bybit (secondary)
  universe: dynamic top-50 by 24h volume (recalculated daily 00:00 UTC)
  commission: 0.1% maker/taker (Binance spot default)

moex_stocks:
  source_realtime: Tinkoff Invest API (production token)
  source_historical: MOEX ISS API
  universe: IMOEX constituents (~40 tickers, updated quarterly)
  commission: 0.3% per trade (Тинькофф «Инвестор»)
  session: main only (10:00–18:50 MSK), evening deferred to v2

moex_bonds:
  scope: all liquid OFZ + top-20 corporate by turnover (monthly rebalance)
  commission: 0.3% per trade

moex_forts:
  scope: Si (USD/RUB), RI (RTS), BR (Brent) futures + liquid options
  commission: exchange fee only (~2-5 RUB per contract)

# === DATA ===
history_depth: 3 years
  1d_candles: full 3 years for all instruments
  1h_candles: full 3 years for top-20 (crypto + MOEX)
  1m_candles: last 6 months (storage optimization)

# === INFRASTRUCTURE ===
compute:
  local: Apple M4 (CPU + Neural Engine / MPS for inference)
  cloud_gpu: A100/H100 available but problematic to use regularly
  strategy: CPU-first for training (XGBoost/LightGBM), M4 MPS for NLP inference
deployment: local server (M4 Mac), Docker Compose
database: TimescaleDB (time-series) + PostgreSQL 16 (metadata) + Redis 7 (queue/cache)

# === SENTIMENT ===
telegram_channels: [@raborynokda, @cbrstocks, @markettwits, @AK47pfl]
nlp_models: FinBERT (EN), ruBERT/DeepPavlov (RU)

# === BUDGET ===
data_subscriptions: $0 (all free APIs)
```

---

## Appendix A: Deployment (Docker Compose)

> **Note:** Deployment на Apple M4 Mac. Docker Desktop for Mac поддерживает ARM-native контейнеры. PyTorch MPS backend доступен **только** на хосте macOS (не внутри Docker). Для NLP inference с MPS-ускорением — запускать ml-pipeline вне Docker напрямую на хосте, или использовать CPU mode внутри контейнера (достаточно быстро для batch inference).

```yaml
version: "3.8"
services:
  timescaledb:
    image: timescale/timescaledb:latest-pg16
    volumes:
      - timescale_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  data-ingestion-ws:
    build: ./services/data-ingestion-ws
    depends_on: [timescaledb, redis]
    restart: always
    # Python asyncio WebSocket connectors (Binance, Bybit)

  data-ingestion-rest:
    build: ./services/data-ingestion-rest
    depends_on: [timescaledb, redis]
    restart: always
    # Python asyncio REST pollers: MOEX ISS, CryptoPanic, Reddit, RSS, Etherscan, FRED, CBR

  data-ingestion-tinkoff:
    build: ./services/data-ingestion-tinkoff
    depends_on: [timescaledb, redis]
    environment:
      TINKOFF_INVEST_TOKEN: ${TINKOFF_INVEST_TOKEN}
    restart: always
    # Python gRPC streaming: candles, order book, trades for MOEX

  data-validator:
    build: ./services/data-validator
    depends_on: [timescaledb, redis]
    restart: always

  feature-engine:
    build: ./services/feature-engine
    depends_on: [timescaledb]
    restart: always

  backtest-engine:
    build: ./services/backtest-engine
    depends_on: [timescaledb, redis]
    restart: always

  hypothesis-engine:
    build: ./services/hypothesis-engine
    depends_on: [timescaledb, redis, feature-engine]
    restart: always

  ml-pipeline:
    build: ./services/ml-pipeline
    depends_on: [timescaledb, redis]
    # CPU-first: XGBoost/LightGBM native; PyTorch MPS for NLP inference on macOS host
    # На Linux (Docker) — PyTorch CPU mode для NLP inference
    restart: always

  signal-service:
    build: ./services/signal-service
    depends_on: [timescaledb, redis]
    restart: always

  telegram-bot:
    build: ./services/telegram-bot
    depends_on: [redis, timescaledb]
    environment:
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
    restart: always

  api-gateway:
    build: ./services/api-gateway
    depends_on: [timescaledb, redis]
    ports:
      - "8000:8000"
    restart: always

volumes:
  timescale_data:
```

---

## Appendix B: Hypothesis DSL (Domain-Specific Language)

Для удобного описания стратегий через Telegram:

```
# Крипта: Long entry when RSI oversold + price above long-term trend
STRATEGY mean_reversion_btc
  MARKET crypto
  SYMBOL BTC/USDT
  TIMEFRAME 4h
  
  ENTRY LONG WHEN
    RSI(14) < 30
    AND CLOSE > SMA(200)
    AND VOLUME > SMA(VOLUME, 20) * 1.5
  
  EXIT WHEN
    RSI(14) > 65
    OR CLOSE < ENTRY_PRICE * 0.97  # stop loss 3%
  
  RISK
    POSITION_SIZE 10%
    MAX_CONCURRENT 3

# MOEX: Сбербанк momentum после gap-up
STRATEGY sber_gap_momentum
  MARKET moex_stock
  SYMBOL SBER
  TIMEFRAME 1h
  
  ENTRY LONG WHEN
    GAP_UP_PCT > 1.0
    AND VOLUME_FIRST_HOUR > SMA(VOLUME_FIRST_HOUR, 20) * 2
    AND IMOEX_TREND = UP
  
  EXIT WHEN
    TIME = 18:30  # до закрытия основной сессии
    OR CLOSE < ENTRY_PRICE * 0.985
  
  RISK
    POSITION_SIZE 5%

# ОФЗ: ставка на steepening кривой после решения ЦБ
STRATEGY ofz_curve_steepener
  MARKET moex_bond
  TIMEFRAME 1d
  
  ENTRY WHEN
    CBR_RATE_CHANGE < 0           # ЦБ снизил ставку
    AND YIELD_CURVE_SLOPE(2Y, 10Y) < PERCENTILE(20, 90d)
  
  EXIT WHEN
    HOLDING_DAYS > 60
    OR YIELD_CURVE_SLOPE(2Y, 10Y) > PERCENTILE(60, 90d)
```

Парсер конвертирует DSL в `hypothesis.params` JSONB.