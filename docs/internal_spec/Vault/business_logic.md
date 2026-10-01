# ====== Vault BUSINESS LOGIC ======

## Purpose

`Vault` володіє:
- Безпечним збереженням, структуруванням та контролем життєвого циклу конфіденційних зашифрованих записів (секретів).
- Валідацією правил та доменних політик доступу / стану конкретних секретів.
- Реєстрацією фактів аудиту над захищеними сутностями.
- Наданням атомарних операцій для збереження, дешифрування, відкликання та оновлення секретів.

**NOT here:**
- Аутентифікація користувачів та управління обліковими записами — відповідальність модуля `Identity`.
- Пряма апаратна або низькорівнева реалізація алгоритмів шифрування — делегується в `Infrastructure` через контракт `ICryptoProvider`.
- Безпосередній транспорт (HTTP контролери, передача сокетів) — шар `Presentation`.

---

## Entities

| Entity | Basic Fields | Description | Invariants |
|---|---|---|---|
| `SecretItem` *(Aggregate Root)* | `SecretIdVO id`, `SecretNameVO name`, `EncryptedDataVO encryptedPayload`, `SecretStatusVO status`, `DateTime createdAtUtc`, `DateTime? expiresAtUtc` | Головна сутність для збереження і контролю життєвого циклу секрету. | - `id` іммутабельний.<br>- `encryptedPayload` обов'язковий і ніколи не містить відкритих даних.<br>- Створюється виключно у статусі `ACTIVE`.<br>- Переходи між статусами можливі виключно за матрицею переходів через виклик доменних політик.<br>- Якщо `expiresAtUtc` задано, воно повинно бути строго пізніше `createdAtUtc`. |
| `AuditLog` *(internal Entity — owned by Vault aggregate flow)* | `AuditIdVO id`, `SecretIdVO secretId`, `AuditActionVO action`, `DateTime performedAtUtc` | Незмінний запис про факт виконання операції над секретом. | - Іммутабельний: поля ініціалізуються один раз і не змінюються.<br>- `performedAtUtc` фіксується за системним UTC-часом події.<br>- Заборонено видаляти або модифікувати існуючі записи (append-only). |

---

## Status Lifecycle

- **Старт**: Нова сутність `SecretItem` створюється виключно в статусі `ACTIVE`.

Дозволені переходи:

| From | Self-initiated (User) | System-initiated |
|---|---|---|
| `ACTIVE` | → `REVOKED` | → `EXPIRED` |
| `REVOKED` | — *(terminal block)* | — *(terminal block)* |
| `EXPIRED` | → `ACTIVE` *(при оновленні вмісту / терміну)* | — |

- **REVOKED** є термінальним статусом: відкликаний секрет не може бути активований знову або прочитаний (потребує створення нового запису).
- **EXPIRED** виставляється автоматично системою перевірки часу життя; читання блокується, доки секрет не буде оновлений користувачем.

---

## Value Objects

| Value Object | Description | Invariants |
|---|---|---|
| `SecretIdVO` / `AuditIdVO` | Типізовані UUID-ідентифікатори сутностей (розширюють `AEntityIdVO`). | - Іммутабельний.<br>- Валідний GUID (не `Guid.Empty`). |
| `SecretNameVO` | Читабельна назва секрету для ідентифікації користувачем. | - Іммутабельний.<br>- Довжина: від 3 до 100 символів.<br>- Дозволені тільки символи `a-z`, `A-Z`, `0-9`, `-`, `_`, `.`. |
| `EncryptedDataVO` | Інкапсулює шифротекст, ініціалізаційний вектор (IV) та алгоритм. | - Іммутабельний.<br>- `Ciphertext` не може бути порожнім.<br>- `IV` (Initialization Vector) має точний фіксований розмір (напр., 16 байт для AES). |
| `SecretStatusVO` | Обгортає енум `SecretStatus` (`ACTIVE`, `REVOKED`, `EXPIRED`). | - Іммутабельний.<br>- Стан відповідає валідному значенню з доменного енума. |
| `AuditActionVO` | Типізоване позначення виконаної дії (`STORED`, `ACCESSED`, `REVOKED`, `ROTATED`). | - Іммутабельний.<br>- Значення обмежене допустимим переліком доменних дій аудиту. |

---

## Domain Policies

| Domain Policy | Description |
|---|---|
| `RevokeSecretDPolicy` | Перевіряє можливість відкликання: блокує операцію, якщо секрет уже знаходиться у статусі `REVOKED`. |
| `AccessSecretDPolicy` | Перевіряє валідність надання доступу до дешифрування: блокує читання, якщо статус `REVOKED` або `EXPIRED`. |
| `ReactivateSecretDPolicy` | Перевіряє, чи дозволено повернення в `ACTIVE`: дозволяє перехід тільки зі статусу `EXPIRED` за умови передачі валідного нового payload. |

---

## Domain Services

Кожен доменний сервіс виконує одну атомарну бізнес-операцію та звертається до зовнішніх механізмів лише через інвертовані інтерфейси (`OutsourceContract`).

| Domain Service | Operation |
|---|---|
| `StoreSecretService` | Приймає сирі дані, звертається до `ICryptoProvider` для шифрування, формує `EncryptedDataVO`, створює `SecretItem`, фіксує івент `SecretStoredDE`. |
| `AccessSecretService` | Валідує стан секрету через `AccessSecretDPolicy`, викликає `ICryptoProvider` для розшифрування, формує сутність `AuditLog` про факт доступу, емітить `SecretAccessedDE`. |
| `RevokeSecretService` | Застосовує `RevokeSecretDPolicy`, переводить `SecretItem` у стан `REVOKED`, емітить `SecretRevokedDE`. |

---

## Domain Events

Усі івенти несуть лише метадані. Пейлоади відкритих/зашифрованих даних ніколи не передаються в івентах.

| Event | Carries | Notes |
|---|---|---|
| `SecretStoredDE` | `Guid eventId`, `SecretIdVO secretId`, `DateTime createdAtUtc` | Емітиться при успішному початковому збереженні. |
| `SecretAccessedDE` | `Guid eventId`, `SecretIdVO secretId`, `DateTime accessedAtUtc` | Емітиться при успішній видачі секрету користувачу. |
| `SecretRevokedDE` | `Guid eventId`, `SecretIdVO secretId`, `DateTime revokedAtUtc` | Слухається фоновими службами для інвалідації кешів. |
| `SecretStatusChangedDE`| `Guid eventId`, `SecretIdVO secretId`, `SecretStatusVO oldStatus`, `SecretStatusVO newStatus` | Слухається компонентами аудиту. |

---

## Application Commands & Queries

Тонкий шар Application: приймає CQRS-об'єкти та оркеструє доменні сутності, сервіси й репозиторії.

**Commands (`AC`):**

| Area | Commands |
|---|---|
| Управління секретами | `StoreSecretAC`, `RevokeSecretAC`, `UpdateSecretPayloadAC` |
| Технічні / Системні | `MarkSecretsAsExpiredAC` |

**Queries (`AQ`):**

| Query | Purpose |
|---|---|
| `GetSecretAQ` | Оркеструє перевірку доступу, дешифрування та повертає секрет з відкритим значенням (разом із записом в аудит). |
| `ListSecretsMetadataAQ` | Повертає список метаданих секретів (ID, Name, Status, CreatedAt, ExpiresAt) без зашифрованого вмісту. |

---

## Infrastructure

### Outsource Contracts (Domain Interfaces)
- `ICryptoProvider`: абстракція над алгоритмом шифрування (напр., AES-256-GCM). Реалізується в `Infrastructure/Cryptography`.
- `ISecretRepository`: контракт персистенсу для збереження та завантаження сутності `SecretItem`.
- `IAuditLogRepository`: контракт збереження аудит-записів (append-only).

### Persistence (EF Core)
- `SecretItemConfiguration` — мапінг `SecretItem`:
    - `Id` конвертується у первинний ключ GUID.
    - `EncryptedDataVO` розбивається на окремі стовпці (`Ciphertext`, `IV`).
    - Статус зберігається рядком або типізованим int.
- `AuditLogConfiguration` — мапінг таблиці журналу подій. 
