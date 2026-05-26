        # kafka — Page cache, zero-copy, retention.bytes

        Homework-шаблон для урока **l3_storage_tuning** (Page cache, zero-copy, retention.bytes) на платформе Vibe Learn.

        ## Что делать

        Измерь разницу throughput между warm и cold consumer на Go.
1. Producer пишет 1 GB данных в топик `storage-bench` (6 партиций).
2. Warm consumer: стартует сразу после producer — читает из page cache.
   Измерь throughput (MB/s) за первые 10 секунд чтения.
3. Cold consumer: подожди 10 минут после записи (page cache вытеснился из-за
   других операций в docker-окружении), затем читай. Измерь throughput.
4. CI assert: warm_throughput > 1.5 × cold_throughput.
Используй segmentio/kafka-go. Логируй MB/s каждые 2 секунды.
Конфигурация consumer: MinBytes=1 MiB, MaxBytes=10 MiB.

## Контекст (из transfer-задачи урока)

Платформа жалуется: consumer lag растёт всё больше каждую неделю, throughput consumer'а
падает с 100 MB/s в начале месяца до 30 MB/s через месяц. `iostat` на брокере показывает
disk read utilization 90%. Топик имеет 6 партиций, `retention.ms=30d`, `retention.bytes=-1`
(не ограничен по объёму). Consumer группа читает данные с задержкой ~3 недели
(обрабатывает исторические данные). `security.protocol=PLAINTEXT`.

**Вопрос:** Что происходит? Объясни механизм деградации throughput. Что менять, в каком порядке?
Предложи конкретные параметры.

## Recap из урока

- **Linux page cache** — первичный storage layer Kafka. Hot consumer читает из RAM (page cache), cold consumer — с диска. Разница в throughput может быть 5–10×.
- **Zero-copy через sendfile()** устраняет два memcpy и context switch: данные идут page cache → socket buffer → NIC без копии в JVM heap. Отключается при SSL, compression-перекодировании и isolation.level=read_committed.
- **retention.bytes применяется к партиции**, не к топику. Топик с 12 партициями и retention.bytes=10 GiB может занять до 120 GiB.
- **flush.messages=1 — антипаттерн**: durability обеспечивает replication (replication.factor + min.insync.replicas + acks=all), не per-message fsync. fsync на каждое сообщение убивает throughput.
- Тюнинг параметров брокера без профилирования вреден. Менять num.network.threads и num.io.threads только при явном bottleneck в JMX-метриках.

        ## Как работать

        1. Платформа Vibe Learn создаёт копию этого репо в твоём GitHub-аккаунте по клику «Начать домашку» на странице урока (через GitHub `/generate`, codecrafters-pattern).
        2. Склонируй копию локально, реализуй TODO в `main.go`, прогони тесты, запушь.
        3. CI (`.github/workflows/ci.yml`) запускает `go vet` + `go test ./...` на каждый push. Платформа слушает результат через webhook от GitHub Actions и обновляет статус домашки на странице урока.

        ## Локальное окружение

        - Go 1.22+
        - Docker + docker-compose — `docker compose -f docker-compose.yml up -d` поднимает 3-нодовый Kafka cluster на портах 9092/9093/9094, использовать в тестах через bootstrap `localhost:9092,localhost:9093,localhost:9094`.

        ## Запуск

        ```bash
        # Поднять локальный Kafka
        docker compose up -d

        # Прогнать тесты (часть из них стартует свой ephemeral testcontainers cluster, часть использует docker-compose выше)
        go test ./...

        # Запустить main (печатает marker; замени stub на реализацию)
        go run .
        ```

        ## Заметка автора

        Это baseline-шаблон, сгенерированный платформой. Бизнес-сущность задачи (что конкретно реализовать в `main.go`, какие тесты сделать строгими) расширяется по ходу итераций — параллельно с углублением теории урока.
