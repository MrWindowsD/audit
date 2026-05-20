# Codemap — Project Viski (Network Optimizer)

Актуальная структура проекта и описание архитектурных решений после завершения **Epic 3 (Фрагментация)**.

---

## 1. Назначение продукта

Локальный «оптимизатор сети» на базе `VpnService` с Go-ядром. Использует **gVisor Netstack** для обработки трафика в userspace и **Transparent Proxy** для управления соединениями с поддержкой фрагментации (DPI Bypass).

---

## 2. Идентификаторы и версии

| Параметр | Значение |
|----------|----------|
| `applicationId` | `com.mrwindowsd.optimizer` |
| compile / target SDK | 35 |
| min SDK | 24 |
| Go / gVisor | 1.26.3 / v1.25+ API |

---

## 3. Ключевые архитектурные решения

### 3.0. Принципы (не нарушать)

| Принцип | Смысл |
|---------|--------|
| **Фрагментация ≠ поломка** | DPI-обход не должен отрезать часть трафика. Профиль применяется ко всему TCP с `type ≠ 0` (кроме 127.x/::1). Если ломает — баг в стратегии/пайплайне. |
| **test_domains ≠ маршрутизация** | Список только для **Auto-Tuner** (Prober). Никакого split-tunnel через этот список. |
| **Split-tunnel → Epic 5** | Домены + приложения — отдельный эпик (rule-sets, Fake-IP+route, `addDisallowedApplication`). До этого — глобальный VPN + глобальный профиль фрагментации. |

### 3.1. Фрагментация и Auto-Tuner (Epic 3 Expansion)
- **Live-фрагментация:** При активном профиле (`StrategyNone` исключён) весь TCP через `BuildPipeline` в `proxyTCP`. `test_domains` — **только** Auto-Tuner.
- **Surgical Shredder:** Крошение только байт доменного имени (SNI) с микро-задержками для десинхронизации DPI.
- **IPv4 Only:** Отключение IPv6 в TUN для исключения побочных попыток соединения по недоступным протоколам.
- **Архитектура Pipeline:** Использование паттерна Middleware для сборки цепочки обработки трафика. 
- **Критический порядок слоев:** 
    1.  **BypassMiddleware:** Только loopback (127.x / ::1) без DPI-слоя.
    2.  **FakePacketMiddleware:** Отправляет "камикадзе"-пакет.
    3.  **SNIMixedCaseMiddleware:** Мутирует регистр SNI.
    4.  **FragmentationMiddleware:** Осуществляет финальное дробление (L4/L7/Shredder).

### 3.2. Локальный Bypass
- **Silent Bypass:** Мгновенное отклонение (TCP RST) локальных пакетов прямо в Forwarder-ах gVisor.
- **DNS Compatibility:** UDP-трафик на локальные адреса разрешен для корректной работы системных DNS-резолверов.

### 3.3. Управление конфигурацией (ConfigManager)
- **Data Flow:** Kotlin (`ConfigManager`) -> JSON -> Go (`EngineConfig`).
- **Синхронизация:** Поля `mixed_case` и `fake_packets` в JSON являются обязательными для активации новых методов.

---

## 4. Структура репозитория

```
ProjectViski/
├── app/
│   └── src/main/
│       ├── java/com/mrwindowsd/optimizer/
│       │   ├── MainActivity.kt           # UI: кнопки Start/Stop, Reset Config (Test)
│       │   ├── core/
│       │   │   ├── CoreBridge.kt         # JNI мост, обработка callback-ов тюнера
│       │   │   └── ConfigManager.kt      # Персистентность (SharedPreferences)
│       │   └── service/
│       │       └── OptimizerVpnService.kt # Жизненный цикл VPN, инъекция конфига
├── go-core/
│   ├── main.go                 # Точка входа, JNI LogWriter
│   ├── bridge.c                # JNI Registration, callbacks
│   ├── build.ps1               # Кросс-сборка под все архитектуры
│   └── stack/
│       ├── netstack.go         # Стек gVisor, Silent Bypass logic
│       ├── proxy.go            # TCP/UDP проксирование, Pipeline execution
│       ├── pipeline.go         # Абстракция Dialer/Middleware, сборка цепочек
│       ├── mixed_case.go       # SNI Mutation (Deterministic "First Upper")
│       ├── fake_packets.go     # Kamikaze Strategy (Protected dummy sockets)
│       ├── hex_debug.go        # Утилиты логирования (limit 512 bytes)
│       ├── smart_split.go          # SmartSplit mid-SNI / legacy modes
│       ├── tuner_profiles_archive.go # Архив Disorder #6–#8 (вне тюнера)
│       └── fragmentation.go    # Матрица стратегий (7 профилей), Auto-Tuner
└── docs/                       # Единый центр документации
```

---

## 5. Статус по эпикам

| Эпик | Статус             | Комментарий |
|------|--------------------|-------------|
| **Epic 1** | ✅ **Done**         | Инфраструктура и логирование. |
| **Epic 2** | ✅ **Done**         | gVisor Netstack и стабильная остановка. |
| **Epic 3** | 🔄 **In progress** | **Фрагментация, SNI Mutation и Fake Packets.** |
| **Epic 4** | 🔄 *Next*          | DNS-маршрутизация (Fake-IP) и каскадный фильтр. |
| **Epic 5** | 📋 *Planned*       | Split-tunnel: домены (rule-sets) + приложения (Android API). |
