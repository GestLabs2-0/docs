# Stayke — Documento de Diseño del Proyecto

---

## Visión general

Stayke es una plataforma descentralizada de alquileres a corto plazo construida sobre Solana. Resuelve el problema de confianza en el corazón de cualquier marketplace de hosting usando primitivos de blockchain.

**El argumento central:** La reputación, identidad y garantías pertenecen al usuario, no a la plataforma. Si Stayke cierra mañana, el historial del usuario sigue existiendo on-chain y cualquier otra plataforma puede leerlo.

**Público objetivo primario:** Usuarios no-crypto. La estrategia de onboarding es migrar gente del mundo Web2 al mundo Web3 de forma gradual y sin fricción.

---

### Misión

Garantizamos la seguridad, reputación, confianza e identidad de todos los usuarios en cada reserva que realicen dándoles la propiedad de sus datos a ellos y no a Stayke.

### Visión

Queremos ser precursores en una infraestructura que cree una reputación de confianza online para reservas y otras múltiples actividades que puedan realizar nuestros clientes.

---

### Modelo de negocios:

El nucleo del modelo de negocios usa la reservación de propiedades como fuente de ingresos principal, sin embargo, esto provoca que Stayke compita contra negocios con más confianza y viejos en este mercado, por eso nosotros decidimos dejar de lado la competencia y enfocarnos en mercados pequeños a los que las empresas grandes no pueden llegar por fricciones bancarias y falta de confianza. 

Stayke opera dentro de la blockchain de Solana para mantener un registro público de la reputación de un usuario sin tener que almacenar datos públicos delicados y evadir las fricciones bancarias de distintos países sin complicar la comunicación con la blockchain usando una wallet embebida.

Para que Stayke pueda ofrecer seguridad, este exigirá un depósito inicial luego de la primera reserva o hosting realizado que permanacerá en una tesorería controlada por el contrato inteligente. El objetivo de la tesorería es que se penalice a los usuarios que cometan alguna infracción y sirva de recompensación a los usuarios que sufrieron el percance. Esto aumenta la fricción de los usuarios que ingresan a la plataforma, pero garantiza que aquellos que ingresen son personas de confianza.

Más allá de que Stayke ofrezca como servicio base la renta de propiedades, decidimos crear un par de beneficios alrededor de este para aumentar su atractivo y crear un mercado nuevo: 
- Ofrecer lending y staking opcional para los usuarios que tengan dinero depositado para que, incluso en momentos sin reservas, tengan ingresos pasivos por el dinero en protocolos de liquidez.
- Crear una modalidad de re-venta de reservas autorizada por el cliente y el host con limitaciones reales.
- Ofrecer beneficios a los usuarios con bastante dinero depositado puesto que esto es sinónimo de confianza en el protocolo.

#### Ideas para expandir el negocio
- Creación de tokens por tiempo en la plataforma otorgando beneficios a los usuarios.
- Ampliar el negocio de reservas a distintos mercados donde la reputación sea de relevancia como la renta de carros, servicios, etc.
- Implementar protocolo x402 con inteligencia artificial para que usuarios ocupados puedan simplemente asignar a su modelo de confianza el lugar que esté más adaptado a su personalidad.
- Implementar búsqueda de propiedades para reservar con fundamento en la personalidad y necesidades del usuario utilizando modelos de IA recientes.
- Pagos para aumentar la visibilidad de la propiedad en la plataforma.
- El dinero bloqueado para una reserva puede ser enviado a un fondo de liquidez por elección del host y el cliente para obtener beneficios durante el tiempo que dure la reserva.
- Ofrecer descuentos por una determinada cantidad de reservas culminadas exitosamente.
- Habilitar una red de recomendaciones que visibilice lugares basado en: países con escaso turismo, gustos personales y antiguos lugares visitados

---

## Arquitectura

### Modelo híbrido

| Capa | Responsabilidad | Tecnología |
|---|---|---|
| On-chain | Garantías, escrow, identidad, reputación, baneos | Anchor (Rust) en Solana |
| Backend | Listados, búsqueda, chat, KYC, imágenes | Tradicional (a definir) |
| Frontend | Interfaz de usuario, integración wallet | Next.js |

La `pubkey` del usuario es la **foreign key universal** que une ambas capas.

Stayke sigue una arquitectura híbrida usando la blockchain como la fuente de la verdad y un backend como referencias a información de los usuarios en la blockchain.

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
| Wallet / Auth | Web3Auth o Custom | Dynamic, Magic, Turnkey |
| KYC | Didit o Sumsub | Sumsub, Civic Pass, Stripe Identity |
| Lending / Yield | Kamino Finance | MarginFi, Drift |
| Token de pago | USDC | — |
| Liquid staking (opcional) | Marinade (mSOL) / Jito (jitoSOL) | — |

---

## Autenticación

### Onboarding con Web3Auth

Web3Auth abstrae completamente la wallet para el usuario no-crypto. El usuario se registra con email o proveedor social (Google, Apple); Web3Auth genera una wallet Solana real en background usando MPC (Multi-Party Computation), de modo que ninguna entidad tiene la clave privada completa. Las firmas para SIWS y para transacciones on-chain suceden de forma transparente.

### Flujo Sign-In with Web3Auth

Usamos la verificación con Web3Auth.
1.- El usuario se autentica con email o proveedor social en el frontend.
2.- Web3Auth devuelve un JWT y reconstruye la wallet del usuario de forma descentralizada.
3.- Se envía el JWT al backend como bearer token para validar la sesión.
4.- El backend verifica el JWT con la SDK de Web3Auth y asocia la `pubkey` resultante al perfil del usuario.

---

## Identidad y KYC

### Tres cuentas por usuario

| Account | Propósito | Estado |
|---|---|---|
| `IdentityAccount` (on-chain) | Vinculado al documento real. Se congela al verificarse. | Inmutable post-verificación |
| `UserProfile` (on-chain) | Cuenta de uso normal. Linked al `IdentityAccount`. | Mutable |
| `ReputationProfile` (on-chain) | Cuenta que almacena todos los datos relativos a la reputacion del usuario. | Mutable |

**Estructura de cuentas:**

```
pub struct ReputationProfile {
    pub owner: Pubkey,
    pub host_reviews: u32,     // Number of reviews received as host
    pub total_score_host: u64, // Total score from reviews (e.g., sum of ratings)
    pub client_reviews: u32,     // Number of reviews received as client
    pub total_score_client: u64, // Total score from reviews (e.g., sum of ratings)
    pub hosted_stays: u32,    // Number of stays hosted
    pub completed_stays: u32, // Number of stays completed as a guest
    pub host_cancellations: u32,   // Number of cancellations as host
    pub client_cancellations: u32, // Number of cancellations as client
    pub host_cancellations_within_48h: u32, // Number of cancellations as host within 24 hours of the stay
    pub client_cancellations_within_48h: u32, // Number of cancellations as host within 24 hours of the stay
    pub low_infractions: u8,
    pub medium_infractions: u8,
    pub high_infractions: u8,
    pub last_updated: i64, // unix timestamp of the last update to the reputation profile
    pub bump: u8,
}

pub struct UserProfile {
    pub owner: Pubkey,
    pub identity: Pubkey,
    pub active_booking: Option<Pubkey>,
    pub active_stay: Option<Pubkey>,

    // This field represents the total amount of tokens that the user has deposited in the platform, excluding the ones that are currently being used for lending and staking.
    pub deposited: u64,
    pub deposit_timestamp: i64,
    pub lending: u64, // This field represents the total amount of tokens that the user has lent
    pub staked: u64, // This field represents the amount of liquid staked tokens that the user has.
    pub is_verified: bool,

    // Counter for the amount of listings that the user has created, this is used to generate the listing_id for each listing created by the user.
    pub listings: u16,
    pub bump: u8,
}

pub struct Identity {
    pub owner: Pubkey,
    pub country_code: [u8; 2],
    pub id: [u8; 32],
    pub verified_at: i64,         // unix timestamp
    pub verifier: Option<Pubkey>, // who verified (oracle, admin, or the very program)
    pub doc_type: DocType,
    pub is_frozen: bool,
    pub is_banned: bool,
    pub banned_at: i64 
    pub bump: u8,
}

```

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
Frontend → envía al backend (Didit SDK)
Sumsub → valida el documento real
Backend → computa hash(canonical), firma verify_identity(commitment)
Programa Anchor → crea/actualiza IdentityAccount
```
---

## Modelo económico

### Tokens de pago

**USDC** es el token principal para todas las transacciones. Justificación: ampliamente adoptado en Solana, fondos auditados, más legal que USDT.

### Dos fondos completamente separados

Para reducir la fricción por el uso de un depósito inicial, la primera operación en la plataforma puede ser sin un fondo
de depósito inicial. El objetivo es que los usuarios perciban que la plataforma realmente no intercede en el dinero que ellos poseen.


```
┌─────────────────────────────────────────────────────────┐
│ DEPÓSITO INICIAL (bond de comportamiento)               │
│                                                         │
│ • Permanente en la plataforma                           │
│ • Puede ser slasheado por mal comportamiento            │
│ • Determina tier de confianza del usuario               │
│ • OPCIONAL: el usuario decide si deposita               │
│                                                         │
│ Lending → Kamino (100% opt-in, decisión del usuario)    │
│ Staking → mSOL/jSOL (100% opt-in, decisión del usuario) │
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

### Yield

*Es siempre opcional hacer lending.*

| Estrategia | APY aprox | Activo | Volatilidad |
|---|---|---|---|
| Kamino USDC lending | ~4-10% | USDC | Ninguna |
| mSOL (Marinade) | ~5.95% | SOL | Expuesto a precio SOL |
| jitoSOL (Jito) | ~6-7% | SOL | Expuesto a precio SOL |

---

## Mercado de reservación

**Flujo de reservas:** 
1. Solicitud — El huésped selecciona una propiedad e inicia la reserva.
2. Creación de cuentas — El sistema genera automáticamente una cuenta que registra los días de la reserva y una cuenta de reserva con toda la información relevante, pendiente de aprobación del host.
3. Aprobación doble — Si el host acepta, el huésped debe confirmar la reserva y realizar un depósito a la cuenta escrow.
4. Fondos en escrow — A partir de este momento el dinero queda congelado. Si el host lo tiene habilitado, puede iniciarse la operación de reventa.
5. Estado "En proceso" — Al llegar la fecha de ingreso, debe realizarse una llamada al contrato para activar este estado.
6. Período de disputas — Solo durante el estado "En proceso" es posible abrir una disputa. Para cerrar la reserva, ambas partes deben estar de acuerdo.
7. Reseñas obligatorias — Antes de finalizar, tanto el huésped como el host deben dejar su calificación. Sin reseñas pendientes no es posible iniciar una nueva reserva.

**Reglas importantes**

- Cancelación tardía — Cancelar dentro de los dos días previos al ingreso genera un slash tanto al host como a Stayke.
- Comportamiento reiterativo — Si un usuario inicia y cancela reservas con frecuencia, recibirá una amonestación que afectará su reputación en la plataforma.

**Propuesta: fondo de liquidez**
Mientras los fondos permanecen en escrow durante la reserva, podrían redirigirse a un fondo de liquidez para generar rendimientos adicionales al host y a Stayke. Condiciones por considerar:

- Requiere aprobación del host.
- Si la opción aplica durante todo el período en que exista la cuenta escrow (incluyendo la posibilidad de cancelación), también necesitaría aprobación del huésped.
- Solo activo durante la vigencia de la reserva confirmada.

---

## Mercado de re-venta de reservas

Un mercado de re-venta de reservas crea una economía dentro de la plataforma permitiendo que un usuario haga una o varias reservas para su venta a distintos usuarios.
Aunque suena como una idea atractiva, debe estar limitada en los diferentes aspectos puesto que el mal uso de distintos usuarios reduciría el movimiento
en lugar de aumentarlo. Los aspectos son los siguientes: 
- La plataforma no debe permitir la re-venta de reservas sin un acuerdo previo entre el usuario y el host solicitado en cada reserva que realicen ambos.
- La re-venta solo se puede completar si el host acepta al nuevo usuario al que se le desea transferir la reserva puesto que se está poniendo en riesgo al host.
- Si se cancela una reserva que se puede vender en el plazo de dos días antes de la misma reserva, se hará un slash del depósito que se le dará la host y a Stayke.

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
1. Usuario abre disputa
2. Se crea DisputeAccount on-
3. Se establece un límite para que los usuarios suban sus evidencias y la disputa sea resuelta por Stayke
4. Evidencia off-chain se adjunta al backend
5. Usuarios resuelven entre ellos la disputa determinando si hubo realmente algún problema y determinando el slash si es que hubo un problema.
6. En el caso de que la disputa no se haya resuelto en la ventana de tiempo establecida, se escalará a Stayke.
7. Admin (MVP) o jurado (V2) revisa
8. Resolución:
   - Si el denunciante tenía razón → recupera stake + compensación
   - Si el denunciante estaba equivocado → stake va al pool de resolución
```

### Slash progresivo

La cantidad de disputas mínimas para un baneo debe ser consideraba por la gravedad del problema y no solo por la cantidad total.

El slash siempre se ejecuta primero sobre la porción en lending (más líquida), luego sobre la porción en staking si la hubiera.

El porcentaje del slash debe variar de acuerdo a la gravedad de la disputa y es algo que aún está por definir: 

**Slash por gravedad de disputa:**

### Modelo de resolución escalado

**MVP (Stage 1):** Administradores centralizados e imparciales provistos por Stayke. Solo pueden juzgar disputas — no pueden remover baneos arbitrariamente ni modificar el protocolo.

**Stage 4:** Sistema de jurado descentralizado con poder de voto ligado a reputación alta. Staking económico cuando la plataforma tenga volumen suficiente de disputas para alimentarlo.

---

## Onboarding no-crypto (estrategia de Web3Auth)

```
Nivel 0 — Usuario nuevo
  Registro con email o proveedor social. Todo sucede on-chain en background.
  Nunca ve una wallet, nunca firma manualmente, nunca paga gas.

Nivel 1 — Usuario con historial
  El sistema le muestra: "Tu reputación está en Solana, es tuya para siempre."
  Puede ver su ReputationAccount en un explorador si quiere.

Nivel 2 — Usuario avanzado
  Puede exportar su private key desde Web3Auth (MPC key reconstruction).
  Conecta su propia wallet (Phantom, Backpack).
  Entiende el valor de la soberanía de sus datos.
```

---

## Ideas experimentales por definir para los siguientes stages.

### Fondo de reservas

Al igual que las millas de viaje por la distancia recorrida en una aerolínea, Stayke debe considerar crear una promoción del mismo estilo por la cantidad de reservas que haga el usuario funcionando como un tipo de fondo de ahorro por cada reserva que haga.

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

### Jurado descentralizado para disputas

Ver Stage 4 del Roadmap para la descripción completa del mecanismo.

---

### Ideas por considerar

- Implementación de agente de IA que analice la personalidad, gustos y necesidades del usuario para recomendarle localizaciones o buscarlas por él en el caso de que no quiera decidir + integración x402.
- Chat en tiempo real (para el hackathon, mensajería básica)
- Integrar sistema de verificación de propiedades con app para evitar imágenes falseadas con IA
- La suma de reseñas y puntos dentro de la plataforma genera descuentos en la plataforma cada X cantidad de reservas, aumentando el dinero que gana el host y reduciendo los costos del usuario.
- Los dueños de las propiedades pueden pagar una tarifa (en la criptomoneda nativa de tu plataforma o en dólares) para que su alojamiento aparezca fijado en la página principal o en la parte superior de los resultados de búsqueda durante un fin de semana.


## Roadmap

### Stage 1 — MVP

El objetivo del Stage 1 es demostrar el loop completo de una reserva end-to-end con identidad, reputación y escrow funcionando on-chain. Todo lo que no sea esencial para ese loop queda fuera de este stage.

**Contratos Anchor (on-chain):**
- `IdentityAccount` — creación, verificación y congelamiento por baneo
- `UserProfile` — perfil de usuario vinculado a la identidad
- `ReputationProfile` — actualización por booking completado, review y disputa resuelta
- Escrow de reserva — flujo completo: solicitud → aprobación doble → confirmación → en proceso → cierre → reseñas
- Sistema de disputas semi-centralizado con admins de Stayke como árbitros

**Integraciones:**
- KYC con Didit o Sumsub. El backend actúa como oracle centralizado: computa el hash del documento y firma la instrucción `verify_identity` hacia el programa Anchor
- Autenticación con Web3Auth — onboarding sin fricción para usuarios no-crypto

**Fuera de scope en Stage 1:**
- Lending y staking (ambos son 100% opcionales y se implementan en Stage 2)
- Tokens de reputación
- Mercado de re-ventas
- Jurado descentralizado

---

### Stage 2 — Economía de confianza

El objetivo del Stage 2 es monetizar la confianza acumulada en el Stage 1 y crear incentivos de retención para hosts y huéspedes.

- **Lending opcional con Kamino Finance** — 100% opt-in por parte del usuario. El usuario autoriza explícitamente que una porción de su depósito sea enviada a Kamino. Stayke no mueve fondos sin aprobación activa.
- **Liquid staking opcional** — mSOL (Marinade) y jitoSOL (Jito) como alternativa para usuarios avanzados que prefieran exposición a SOL.
- **Fondo de liquidez en escrow** — opción para que el host (y el huésped, si aplica durante el período de cancelación) autoricen que los fondos en escrow generen yield durante la reserva activa.
- **Tiers de confianza por stake** — beneficios diferenciados según el monto depositado en la plataforma.
- **Tokens de reputación de Stayke** — token interno no especulativo, difícil de conseguir, que reduce fees y otorga beneficios por uso frecuente.

---

### Stage 3 — Expansión del mercado

El objetivo del Stage 3 es ampliar la economía de la plataforma con nuevas modalidades de transacción y visibilidad.

- **Mercado de re-venta de reservas** — piloto controlado con hosts seleccionados. La re-venta requiere acuerdo previo entre el usuario original y el host; el host debe aceptar al nuevo huésped antes de completar la transferencia.
- **Visibilidad pagada** — los hosts pueden pagar para destacar su propiedad en resultados de búsqueda o en la página principal durante períodos definidos.
- **Fondo de reservas (programa de fidelidad)** — acumulación de beneficios por cantidad de reservas completadas, similar a un programa de millas.
- **Expansión geográfica** — incorporación de mercados adicionales fuera del piloto inicial.

---

### Stage 4 — Descentralización

El objetivo del Stage 4 es reducir la dependencia de Stayke como árbitro central y transferir poder de gobernanza a la comunidad. Este stage se activa cuando el volumen de la plataforma sea suficiente para sostener los mecanismos descritos.

- **Jurado descentralizado para disputas** — implementación similar a Kleros. Cuando se abre una disputa, el caso y la evidencia se publican en IPFS. Cualquier usuario verificado por KYC puede participar como jurado apostando voluntariamente USDC. El sistema de votación es de commit-reveal (48 horas de ventana); todos los votos valen igual independientemente de la apuesta. Votar con la mayoría recupera la apuesta más una ganancia proporcional; votar contra la mayoría implica perder lo apostado.
- **Descentralización del oracle de KYC** — evaluación de alternativas al oracle centralizado del Stage 1.
- **Agente de IA + protocolo x402** — integración de un agente que analice preferencias del usuario y gestione búsqueda y reserva de forma autónoma.

---

## Preguntas abiertas

- [ ] **Flujo de reserva completo paso a paso** — prerequisito crítico para todo el trabajo de contratos restante
- [ ] **¿Quién paga el gas?** — política de gasless transactions o fee delegation para usuarios no-crypto con Web3Auth
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
| Gabriel Torres | Smart contracts / Anchor | Frontend, Backend | Smart contracts / CEO & Co-founder |
| José Medina | Frontend / Integraciones | Frontend | Co-founder & Frontend Dev |
| Kevin Araujo | Backend Dev | Frontend integraciones y maquetación | Backend Dev |
| Raúl González | Backend / Integraciones | Frontend Integraciones / Organizacion | Co-founder & Backend Lead |