# ====== Vault BUSINESS LOGIC ======

## Purpose
`Vault` володіє:
- Безпечним зберіганням, шифруванням та управлінням життєвим циклом конфіденційних даних (секретів).
- Валідацією політик доступу до конкретних секретів.

**NOT here:**
- Аутентифікація користувачів (це відповідальність модуля `Identity`).

## Entities

| Entity | Basic Fields | Description | Invariants |
|---|---|---|---|
| `SecretItem` *(Aggregate Root)* | id, name, encryptedPayload, status | Основна сутність для збереження секрету. | - `id` іммутабельний.<br>- `encryptedPayload` ніколи не містить відкритих даних.<br>- Статус змінюється лише за політиками (Status Lifecycle). |
| `AuditLog` *(internal Entity)* | id, secretId, action, timestamp | Запис про взаємодію із секретом. | - Абсолютно іммутабельний (Append-only). |

## Status Lifecycle 

- **Старт**: Нова сутність створюється виключно в статусі `ACTIVE`.

Дозволені переходи:
| From | Self-initiated | System-initiated |
|---|---|---|
| `ACTIVE` | → `REVOKED` | → `EXPIRED` |
| `REVOKED` | — (block ) | — ( block ) |
| `EXPIRED` | → `ACTIVE` (при оновленні) | — |

## Value Objects

| Value Object | Description | Invariants |
|---|---|---|
| `SecretNameVO` | Назва секрету для ідентифікації. | - Іммутабельний.<br>- Тільки алфавітно-цифрові символи та `_`, `_`. |
| `EncryptedDataVO` | Обгортає зашифрований пейлоад та вектор ініціалізації. | - Іммутабельний.<br>- Вектор ініціалізації має бути строго 16 байт. |

## Domain Services

| Domain Service | Operation |
|---|---|
| `StoreSecretService` | Шифрує дані через інфраструктурний криптопровайдер, валідує політики, створює `SecretItem`, емітить `SecretStoredDE`. |
| `AccessSecretService` | Перевіряє статус, дешифрує, генерує запис аудиту `AuditLog`. |

## Application Commands & Queries

**Commands (`AC`):** `StoreSecretAC`, `RevokeSecretAC`.
**Queries (`AQ`):** `GetSecretAQ`, `ListSecretsMetadataAQ` 