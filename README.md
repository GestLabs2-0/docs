# Stayke — Documento de Diseño del Proyecto

> Estado: borrador de trabajo. Revisión manual pendiente.

---

## Visión general

Stayke es una plataforma descentralizada de alquileres a corto plazo construida sobre Solana. Resuelve el problema de confianza en el corazón de cualquier marketplace de hosting usando primitivos de blockchain.

**El argumento central:** La reputación, identidad y garantías pertenecen al usuario, no a la plataforma. Si Stayke cierra mañana, el historial del usuario sigue existiendo on-chain y cualquier otra plataforma puede leerlo.

**Público objetivo primario:** Usuarios no-crypto. La estrategia de onboarding es migrar gente del mundo Web2 al mundo Web3 de forma gradual y sin fricción.

---

## Arquitectura

### Modelo híbrido

| Capa | Responsabilidad | Tecnología |
|---|---|---|
| On-chain | Garantías, escrow, identidad, reputación, baneos | Anchor (Rust) en Solana |
| Backend | Listados, búsqueda, chat, KYC, imágenes | Tradicional (a definir) |
| Frontend | Interfaz de usuario, integración wallet | Next.js |

La `pubkey` del usuario es la **foreign key universal** que une ambas capas.

### Principio de separación

El backend **no replica lógica de negocio** que vive on-chain. Solo almacena:
- Referencias a PDAs (direcciones de accounts on-chain)
- Datos que el contrato no necesita: imágenes, texto libre, chat, estado de KYC
- Caché de estado on-chain para búsquedas (eventualmente consistente)

---

## Stack técnico

| Componente | Decisión | Alternativas consideradas |
|---|---|---|
| Blockchain | Solana | — |
| Smart contracts | Anchor (Rust) | — |
| Frontend | Next.js | — |
| Wallet / Auth | Privy | Dynamic, Magic, Turnkey, Civic |
| KYC | Didit | Sumsub, Civic Pass, Stripe Identity |
| Lending / Yield | Kamino Finance | MarginFi, Drift |
| Token de pago | USDC | — |
| Liquid staking (opcional) | Marinade (mSOL) / Jito (jitoSOL) | — |

---

## Autenticación

### Onboarding con Privy

Privy abstrae completamente la wallet para el usuario no-crypto. El usuario se registra con email; Privy genera una wallet Solana real en background. Las firmas para SIWS y para transacciones on-chain suceden de forma transparente.

### Flujo Sign-In with Privy

Usamos la verificación con Privy.
1.- Se obtiene el access token de Privy al iniciar sesión en el frontend. 
2.- Se envía el access token al backend como un bearer token.
3.- Se valida el access token con la SDK de Privy. 

---

## Identidad y KYC

### Tres cuentas por usuario

| Account | Propósito | Estado |
|---|---|---|
| `IdentityAccount` (on-chain) | Vinculado al documento real. Se congela al verificarse. | Inmutable post-verificación |
| `UserProfile` (on-chain) | Cuenta de uso normal. Linked al `IdentityAccount`. | Mutable |
| `ReputationProfile` (on-chain) | Cuenta que almacena todos los datos relativos a la reputacion del usuario. | Mutable |

### Por qué tres cuentas

Si un usuario es baneado, el `IdentityAccount` se congela. El programa rechaza crear un nuevo `UserProfile` y un `ReputationProfile` que lo referencie. No es infalible (podría usar documento de tercero), pero encarece significativamente la evasión.

### Formato de identidad

No existe un estándar universal. La representación canónica es **off-chain** únicamente:

```
{ISO_3166_A2}:{DOC_TYPE}:{DOC_NUMBER_NORMALIZADO}

Ejemplos:
  VE:NID:V12345678
  CO:NID:1234567890
  BR:CPF:00000000000
  MX:CURP:BADD110313HCMLNS09
  US:PP:123456789       ← pasaporte como fallback universal
```

**DocType:**
- `NID` — National ID (cédula, DNI, Aadhaar...)
- `CPF` — Brasil
- `CURP` — México
- `PP` — Pasaporte (fallback universal)
- `RES` — Residency permit
- `DL` — Driver's license (último recurso)

### El número nunca toca la chain

```rust
// Off-chain: el oracle computa el commitment
let canonical = format!("{}:{}:{}", country_code, doc_type, doc_number_normalized);
let commitment = hash(canonical);

// On-chain: solo el hash como seed del PDA
let (identity_pda, bump) = Pubkey::find_program_address(
    &[b"identity", commitment.as_bytes()],
    &program_id,
);
```

Si el PDA ya existe → `AccountAlreadyInitialized` → unicidad garantizada sin exponer datos.

### Flujo de verificación (MVP)

```
Usuario → sube documento al frontend
Frontend → envía al backend (Sumsub SDK)
Sumsub → valida el documento real
Backend → computa hash(canonical), firma verify_identity(commitment)
Programa Anchor → crea/actualiza IdentityAccount
```

### Roadmap de verificación

| Versión | Modelo |
|---|---|
| V1 (hackathon) | Admin/oracle firma verificaciones |
| V2 | Ban automático por patrón, apelación a jurado |
| V3 | ZK proof o integración Civic/Worldcoin con volumen suficiente |

---

## Modelo económico

### Tokens de pago

**USDC** es el token principal para todas las transacciones. Justificación: ampliamente adoptado en Solana, fondos auditados, más legal que USDT.

### Dos fondos completamente separados

```
┌─────────────────────────────────────────────────────────┐
│ DEPÓSITO INICIAL (bond de comportamiento)               │
│                                                         │
│ • Permanente en la plataforma                           │
│ • Puede ser slasheado por mal comportamiento            │
│ • Determina tier de confianza del usuario               │
│ • OPCIONAL: el usuario decide si deposita               │
│                                                         │
│ Porción mínima → Kamino lending (obligatoria)           │
│ Excedente → mSOL/jSOL si el usuario lo autoriza         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ ESCROW DE RESERVA                                       │
│                                                         │
│ • Específico de cada booking                            │
│ • Se crea al confirmar reserva                          │
│ • Se paga por transferencia directa de wallet           │
│ • Se paga por dinero sobrante depositado                │
│ • Se libera al host al completar                        │
│ • Va a disputa si hay problema                          │
│ • OBLIGATORIO para cualquier transacción                │
└─────────────────────────────────────────────────────────┘
```

### Fondo de reservas

Al igual que las millas de viaje por la distancia recorrida en una aerolínea, Stayke debe considerar crear una promoción del mismo estilo por la cantidad de reservas que haga el usuario funcionando como un tipo de fondo de ahorro por cada reserva que haga.

### Yield

*Es siempre opcional hacer lending.*

| Estrategia | APY aprox | Activo | Volatilidad |
|---|---|---|---|
| Kamino USDC lending | ~4-10% | USDC | Ninguna |
| mSOL (Marinade) | ~5.95% | SOL | Expuesto a precio SOL |
| jitoSOL (Jito) | ~6-7% | SOL | Expuesto a precio SOL |

---

## Sistema de reputación

La reputación acumulada on-chain es el activo más valioso del usuario — no puede transferirse a otra identidad aunque consiga otro documento. Un usuario con historial extenso tiene algo irreproducible.

`ReputationAccount` (on-chain) se actualiza con cada booking completado, review recibido y disputa resuelta.

**Pendiente:** estructura completa del `ReputationAccount`.

---

## Anti-falsa publicidad

Cada vez que se modifica un dato vital de una propiedad (título, descripción, dirección, imágenes), se genera un `sha256` de esos datos que se almacena on-chain. Sirve como punto de comparación en disputas.

Durante un booking activo, la modificación de la propiedad está bloqueada a nivel de contrato.

---

## Sistema de disputas y baneo

### Postura sobre el baneo

Ningún sistema puede banear de forma 100% certera. El objetivo es **encarecer la evasión**, no eliminarla. Si alguien usa el documento de un tercero para registrarse, eso ya no está en el control del protocolo.

### Flujo de disputa

```
1. Usuario abre disputa → deposita stake para activarla
2. Se crea DisputeAccount on-chain
3. Evidencia off-chain se adjunta al backend
4. Admin (MVP) o jurado (V2) revisa
5. Resolución:
   - Si el denunciante tenía razón → recupera stake + compensación
   - Si el denunciante estaba equivocado → stake va al pool de resolución
```

### Slash progresivo

| Evento | Consecuencia |
|---|---|
| 1 disputa perdida | Warning |
| 2 disputas perdidas | Restricción temporal automática |
| 3 disputas perdidas | Ban automático, apelación posible |
| Infracción muy grave | Slash total + ban inmediato |

El slash siempre se ejecuta primero sobre la porción en lending (más líquida), luego sobre la porción en staking si la hubiera.

### Modelo de resolución

**MVP (hackathon):** Administradores centralizados e imparciales provistos por Stayke. Solo pueden juzgar disputas — no pueden remover baneos arbitrariamente ni modificar el protocolo.

**Roadmap V2:** Sistema de jurado con poder de voto ligado a reputación alta. Staking económico cuando la plataforma tenga volumen suficiente.

---

## Onboarding no-crypto (estrategia de Privy)

```
Nivel 0 — Usuario nuevo
  Registro con email. Todo sucede on-chain en background.
  Nunca ve una wallet, nunca firma manualmente, nunca paga gas.

Nivel 1 — Usuario con historial
  El sistema le muestra: "Tu reputación está en Solana, es tuya para siempre."
  Puede ver su ReputationAccount en un explorador si quiere.

Nivel 2 — Usuario avanzado
  Puede exportar su private key desde Privy.
  Conecta su propia wallet (Phantom, Backpack).
  Entiende el valor de la soberanía de sus datos.
```

---

## Por definir

### Tiers de confianza por stake

| Tier | Perfil | Beneficios |
|---|---|---|
| Sin depósito | Usuario casual | Acceso básico, fee estándar |
| Tier 1 | Usuario con depósito mínimo | Fee reducido, badge de confianza, propiedades premium |
| Tier 2 | Usuario con depósito alto | Fee mínimo, poder de voto en disputas, yield compartido |

*Los montos exactos por tier están pendientes de definir.*

### Tokens de reputación de Stayke

Token interno de la plataforma. Difícil de conseguir, no especulativo.

**Usos definidos:**
- Reduce el fee de Stayke por reserva (aumenta ganancia del host)
- Descuento para huéspedes por cantidad de reservas
- Porcentaje adicional del yield de lending

**Pendiente de definir:**
- Cómo se obtienen exactamente
- Si hay un marketplace de intercambio entre usuarios
- Máximo de tokens en circulación
- Requisitos para acceder a cada beneficio

---


### Ideas por integrar

- Sistema de tokens con marketplace de intercambio
- Opción de liquid staking para usuarios avanzados
- Jurado descentralizado con staking económico
- Agentes de IA + integración x402
- Chat en tiempo real (para el hackathon, mensajería básica)
- Integrar sistema de verificación de propiedades con app para evitar imágenes falseadas con IA

## Preguntas abiertas

- [ ] **Flujo de reserva completo paso a paso** — prerequisito crítico para todo el trabajo de contratos restante
- [ ] **¿Quién paga el gas?** — política de gasless transactions o fee delegation para usuarios no-crypto con Privy
- [ ] **Estructura del `ReputationAccount`** — cómo escala con bookings y reviews
- [ ] **Montos mínimos de depósito por tier** — valores concretos
- [ ] **Cómo se obtienen los tokens de reputación** — por tiempo, por stake, por actividad, o venta limitada
- [ ] **Estructura completa del `DisputeAccount`** on-chain
- [ ] **Política de contenido de propiedades** — qué tipos de propiedades se aceptan, qué se rechaza

---

## Equipo

| Persona | Rol principal | Areas de apoyo | Rol en el registro |
|---|---|---|---|
| Derek Rojo | Designer / Marketing | Trainee Dev frontend | Marketing y diseño |
| Gallbers Gallardo | Tech Lead / Blockchain Engineer | Frontend, Backend | CTO | 
| Gabriel Torres | Smart contracts / Anchor | Frontend, Backend | Smart contracts / Anchor / CEO & Co-founder |
| José Medina | Frontend / Integraciones | Frontend | Co-founder & Frontend Dev |
| Kevin Araujo | Backend Dev | Frontend integraciones y maquetación | Backend Dev |
| Raúl González | Backend / Integraciones | Frontend Integraciones / Organizacion | Co-founder & Backend Lead |
