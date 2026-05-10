# Stayke — Esquema de Base de Datos

## DEBO MODIFICARLO Y ESTUDIAR LA ESTRUCTURA MAS APROPIADA

```mermaid
erDiagram

    users {
        uuid id PK
        varchar pubkey UK
        varchar email UK
        varchar name
        varchar lastname
        text profile_url
        varchar phone
        varchar identity_pda
        varchar profile_pda
        boolean is_active
        boolean is_banned
        boolean is_verified
        timestamptz banned_at
        text ban_reason
        timestamptz created_at
        timestamptz updated_at
    }

    sessions {
        uuid id PK
        uuid user_id FK
        varchar jwt_jti UK
        inet ip_address
        text user_agent
        timestamptz expires_at
        timestamptz created_at
    }

    kyc_verifications {
        uuid id PK
        uuid user_id FK
        varchar sumsub_applicant_id UK
        varchar status
        char country_code
        varchar doc_type
        varchar commitment_hash
        timestamptz reviewed_at
        text reject_reason
        timestamptz created_at
        timestamptz updated_at
    }

    properties {
        uuid id PK
        uuid host_id FK
        varchar title
        text description
        varchar property_type
        char country_code
        varchar city
        varchar state
        text address
        text address_hint
        decimal latitude
        decimal longitude
        smallint max_guests
        smallint bedrooms
        smallint bathrooms
        decimal price_per_night
        smallint min_nights
        smallint max_nights
        time checkin_time
        time checkout_time
        text house_rules
        varchar content_hash
        varchar content_hash_pda
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    property_images {
        uuid id PK
        uuid property_id FK
        text url
        text storage_key
        boolean is_cover
        smallint sort_order
        timestamptz created_at
    }

    property_amenities {
        uuid property_id FK
        varchar amenity
    }

    user_deposits {
        uuid id PK
        uuid user_id FK
        decimal total_amount
        decimal minimum_required
        decimal lending_amount
        varchar kamino_vault_pda
        decimal staking_amount
        varchar staking_protocol
        boolean staking_authorized
        varchar staking_pda
        varchar status
        timestamptz created_at
        timestamptz updated_at
    }

    deposit_slashes {
        uuid id PK
        uuid deposit_id FK
        uuid dispute_id FK
        decimal amount
        text reason
        varchar executed_by
        timestamptz created_at
    }

    bookings {
        uuid id PK
        varchar booking_pda UK
        uuid property_id FK
        uuid guest_id FK
        uuid host_id FK
        date checkin_date
        date checkout_date
        smallint nights
        decimal price_per_night
        decimal total_amount
        decimal stayke_fee
        decimal host_payout
        varchar escrow_pda
        varchar status
        text guest_message
        text cancellation_reason
        uuid cancelled_by FK
        timestamptz cancelled_at
        timestamptz created_at
        timestamptz updated_at
    }

    reviews {
        uuid id PK
        uuid booking_id FK
        uuid reviewer_id FK
        uuid reviewed_id FK
        uuid property_id FK
        smallint rating
        text comment
        varchar review_type
        timestamptz created_at
    }

    conversations {
        uuid id PK
        uuid booking_id FK
        uuid participant_a FK
        uuid participant_b FK
        timestamptz created_at
    }

    messages {
        uuid id PK
        uuid conversation_id FK
        uuid sender_id FK
        text content
        boolean is_read
        timestamptz created_at
    }

    disputes {
        uuid id PK
        varchar dispute_pda UK
        uuid booking_id FK
        uuid opened_by FK
        uuid against FK
        text reason
        varchar status
        uuid resolver_id FK
        text resolution
        timestamptz resolved_at
        decimal stake_amount
        timestamptz created_at
        timestamptz updated_at
    }

    dispute_evidence {
        uuid id PK
        uuid dispute_id FK
        uuid uploaded_by FK
        text file_url
        text storage_key
        varchar evidence_type
        text description
        timestamptz created_at
    }

    %% Relaciones

    users ||--o{ sessions : "tiene"
    users ||--o{ kyc_verifications : "verifica con"
    users ||--o{ properties : "publica"
    users ||--o{ user_deposits : "deposita"
    users ||--o{ bookings : "reserva como guest"
    users ||--o{ bookings : "recibe como host"
    users ||--o{ reviews : "escribe"
    users ||--o{ reviews : "recibe"
    users ||--o{ disputes : "abre"
    users ||--o{ disputes : "resuelve"
    users ||--o{ conversations : "participa"
    users ||--o{ messages : "envía"

    properties ||--o{ property_images : "tiene"
    properties ||--o{ property_amenities : "tiene"
    properties ||--o{ bookings : "es reservada en"
    properties ||--o{ reviews : "recibe"

    bookings ||--o| disputes : "genera"
    bookings ||--o{ reviews : "origina"
    bookings ||--o| conversations : "tiene"

    disputes ||--o{ dispute_evidence : "adjunta"
    disputes ||--o{ deposit_slashes : "genera"

    user_deposits ||--o{ deposit_slashes : "recibe"

    conversations ||--o{ messages : "contiene"
```

---

## Notas del esquema

### Separación on-chain / off-chain

Las columnas `*_pda` son **referencias**, no réplicas. El backend no duplica lógica del contrato — solo guarda la dirección del account on-chain para poder consultarlo cuando lo necesite.

| Tabla | PDA referenciada |
|---|---|
| `users` | `identity_pda`, `profile_pda`, `reputation_pda` |
| `user_deposits` | `kamino_vault_pda`, `staking_pda` |
| `bookings` | `booking_pda`, `escrow_pda` |
| `disputes` | `dispute_pda` |
| `properties` | `content_hash_pda` |

### Depósito inicial vs escrow de reserva

Son fondos completamente separados. `user_deposits` maneja el bond de comportamiento permanente del usuario. El `escrow_pda` en `bookings` es específico de cada transacción y no se mezcla con el depósito.

### Autenticación

`auth_nonces` tiene TTL corto (5-15 min) y los registros se eliminan al usarse (nonce de uso único). `sessions` es opcional si se usa JWT stateless — solo necesario si se requiere revocación de sesiones.
