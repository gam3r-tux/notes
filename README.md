# Documento Técnico de Análisis de Cambios
## Módulo de Programa de Lealtad — Microservicio de Reportes SmartPOS

---

**Proyecto:** Microservicio de Reportes de Transacciones Bancarias SmartPOS  
**Versión del Documento:** 1.0.0  
**Fecha:** 27 de marzo de 2026  
**Estado:** En revisión  
**Tecnología Base:** Java 11 / Spring Boot 2.7.10  

---

## Tabla de Contenidos

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)
2. [Contexto del Proyecto Actual](#2-contexto-del-proyecto-actual)
3. [Descripción del Cambio](#3-descripción-del-cambio)
4. [Análisis de Impacto](#4-análisis-de-impacto)
5. [Diseño Técnico de la Solución](#5-diseño-técnico-de-la-solución)
6. [Contratos de API — Nuevos Endpoints](#6-contratos-de-api--nuevos-endpoints)
7. [Modelo de Datos](#7-modelo-de-datos)
8. [Integración con los Reportes Existentes](#8-integración-con-los-reportes-existentes)
9. [Manejo de Errores](#9-manejo-de-errores)
10. [Consideraciones de Seguridad](#10-consideraciones-de-seguridad)
11. [Estrategia de Pruebas](#11-estrategia-de-pruebas)
12. [Plan de Implementación](#12-plan-de-implementación)
13. [Riesgos y Mitigaciones](#13-riesgos-y-mitigaciones)
14. [Glosario](#14-glosario)

---

## 1. Resumen Ejecutivo

Este documento describe el análisis técnico y el diseño de los cambios requeridos para incorporar las operaciones del **Programa de Lealtad** (Consulta, Redención y Cancelación de puntos) dentro del microservicio existente de reportes de transacciones de clientes bancarios realizadas a través de dispositivos SmartPOS en comercios afiliados.

La nueva funcionalidad amplía la información disponible en los reportes transaccionales, enriqueciendo los datos del cliente con el estado de su saldo de puntos y el historial de movimientos de lealtad asociados a cada transacción de pago.

---

## 2. Contexto del Proyecto Actual

### 2.1 Descripción General

El microservicio de reportes procesa y expone información sobre las transacciones de pago realizadas por clientes bancarios en terminales SmartPOS de comercios afiliados. Sus responsabilidades principales son:

- Consolidar datos transaccionales provenientes del ecosistema de pagos SmartPOS.
- Generar reportes detallados y resumidos para clientes bancarios.
- Exponer endpoints REST consumidos por canales digitales (app móvil, banca en línea, back-office).

### 2.2 Stack Tecnológico

| Componente | Versión |
|---|---|
| Java | 11 (LTS) |
| Spring Boot | 2.7.10 |
| Spring Web MVC | 5.3.x (transitivo) |
| Spring Data JPA | 2.7.x |
| Maven / Gradle | (según configuración del proyecto) |
| Servidor de aplicaciones | Tomcat embebido |

### 2.3 Estructura Actual de Controllers

El proyecto cuenta con tres controllers principales:

#### `TransactionReportController`
Gestiona los reportes de transacciones individuales y por lote. Expone endpoints para consultar el detalle de una transacción específica, listar transacciones por rango de fecha, y exportar reportes en diferentes formatos.

#### `SummaryReportController`
Genera reportes consolidados y resúmenes estadísticos. Provee vistas agrupadas por comercio, por fecha, y por tipo de operación, orientadas a la auditoría y reconciliación bancaria.

#### `ClientProfileReportController`
Concentra la información del perfil transaccional del cliente bancario. Provee el historial de uso del SmartPOS, frecuencia de pagos, y datos de comportamiento financiero relevantes para el reporte personalizado.

---

## 3. Descripción del Cambio

### 3.1 Objetivo

Incorporar al microservicio de reportes tres nuevas operaciones que interactúan con el **Programa de Lealtad** del banco, de modo que la información de puntos quede registrada y disponible dentro de los reportes transaccionales:

| # | Operación | Descripción |
|---|---|---|
| 1 | **Consulta de Puntos** | Permite obtener el saldo actual de puntos acumulados por el cliente en el programa de lealtad, asociado a su perfil bancario. |
| 2 | **Redención de Puntos** | Registra y reporta el uso (canje) de puntos de lealtad aplicados como beneficio en una transacción de pago SmartPOS. |
| 3 | **Cancelación de Puntos** | Registra la reversión o anulación de puntos previamente canjeados o acreditados, reflejando el ajuste en el reporte de la transacción correspondiente. |

### 3.2 Alcance del Cambio

**Incluido en este cambio:**
- Nuevo controller `LoyaltyReportController` con tres endpoints REST.
- Nuevos DTOs de request/response para las operaciones de lealtad.
- Capa de servicio `LoyaltyReportService` con su interfaz y su implementación.
- Cliente HTTP (Feign Client o RestTemplate) para integración con el servicio externo de lealtad.
- Enriquecimiento de los reportes existentes con campos de lealtad opcionales.
- Manejo de errores específico para las operaciones de lealtad.
- Pruebas unitarias y de integración.

**Fuera del alcance:**
- Modificación de la lógica de negocio del motor de lealtad (sistema externo).
- Cambios en la base de datos transaccional del SmartPOS.
- Modificaciones a los otros tres controllers existentes más allá de la incorporación de los campos de lealtad en las respuestas.

---

## 4. Análisis de Impacto

### 4.1 Componentes Afectados

| Componente | Tipo de Cambio | Nivel de Impacto |
|---|---|---|
| `TransactionReportController` | Adición de campos opcionales en response | Bajo |
| `SummaryReportController` | Adición de sección de resumen de lealtad en response | Bajo |
| `ClientProfileReportController` | Adición de historial de puntos en el perfil del cliente | Medio |
| `LoyaltyReportController` | **Nuevo componente** | Alto |
| `LoyaltyReportService` | **Nuevo componente** | Alto |
| `LoyaltyFeignClient` | **Nuevo componente** | Alto |
| DTOs de respuesta existentes | Adición de campos opcionales (`@JsonInclude(NON_NULL)`) | Bajo |
| Configuración Spring (`application.yml`) | Nuevas propiedades del cliente de lealtad | Bajo |
| Dependencias (`pom.xml`) | Adición de Spring Cloud OpenFeign (si aplica) | Bajo |

### 4.2 Riesgos de Regresión

- La incorporación de campos opcionales en los DTOs existentes es retrocompatible siempre que se use `@JsonInclude(JsonInclude.Include.NON_NULL)`.
- No se modifican rutas ni verbos HTTP existentes.
- El nuevo controller opera en un contexto de ruta `/loyalty` completamente nuevo.

---

## 5. Diseño Técnico de la Solución

### 5.1 Arquitectura de Capas

```
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (REST)                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           LoyaltyReportController                      │  │
│  │   POST /api/v1/loyalty/points/query                   │  │
│  │   POST /api/v1/loyalty/points/redeem                  │  │
│  │   POST /api/v1/loyalty/points/cancel                  │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│                    Service Layer                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  LoyaltyReportService (Interface + Implementation)     │ │
│  │  - Orquesta llamada al cliente externo                 │ │
│  │  - Aplica reglas de negocio y validaciones             │ │
│  │  - Construye el modelo de reporte enriquecido          │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│                  Integration Layer                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         LoyaltyExternalClient (Feign / RestTemplate)   │ │
│  │  - Consumo del API externo del Programa de Lealtad     │ │
│  │  - Circuit Breaker / Retry con Resilience4j            │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Estructura de Paquetes Propuesta

```
com.banco.reportes
├── controller
│   ├── TransactionReportController.java       ← Existente (modificación menor)
│   ├── SummaryReportController.java           ← Existente (modificación menor)
│   ├── ClientProfileReportController.java     ← Existente (modificación menor)
│   └── LoyaltyReportController.java           ← NUEVO
│
├── service
│   ├── loyalty
│   │   ├── LoyaltyReportService.java          ← NUEVO (Interface)
│   │   └── impl
│   │       └── LoyaltyReportServiceImpl.java  ← NUEVO (Implementación)
│   └── ... (servicios existentes)
│
├── client
│   └── loyalty
│       ├── LoyaltyExternalClient.java         ← NUEVO (Feign Client)
│       └── dto
│           ├── LoyaltyQueryRequest.java       ← NUEVO
│           ├── LoyaltyQueryResponse.java      ← NUEVO
│           ├── LoyaltyRedeemRequest.java      ← NUEVO
│           ├── LoyaltyRedeemResponse.java     ← NUEVO
│           ├── LoyaltyCancelRequest.java      ← NUEVO
│           └── LoyaltyCancelResponse.java     ← NUEVO
│
├── dto
│   ├── loyalty
│   │   ├── PointsQueryRequestDTO.java         ← NUEVO
│   │   ├── PointsQueryResponseDTO.java        ← NUEVO
│   │   ├── PointsRedeemRequestDTO.java        ← NUEVO
│   │   ├── PointsRedeemResponseDTO.java       ← NUEVO
│   │   ├── PointsCancelRequestDTO.java        ← NUEVO
│   │   └── PointsCancelResponseDTO.java       ← NUEVO
│   └── ... (DTOs existentes — se enriquecen con campo LoyaltySummaryDTO opcional)
│
├── exception
│   ├── LoyaltyServiceException.java           ← NUEVO
│   ├── InsufficientPointsException.java       ← NUEVO
│   └── ... (excepciones existentes)
│
└── config
    ├── LoyaltyClientConfig.java               ← NUEVO
    └── ... (configuraciones existentes)
```

### 5.3 Nuevo Controller: `LoyaltyReportController`

```java
@RestController
@RequestMapping("/api/v1/loyalty")
@Validated
@Slf4j
public class LoyaltyReportController {

    private final LoyaltyReportService loyaltyReportService;

    public LoyaltyReportController(LoyaltyReportService loyaltyReportService) {
        this.loyaltyReportService = loyaltyReportService;
    }

    /**
     * Consulta el saldo de puntos de lealtad del cliente.
     */
    @PostMapping("/points/query")
    public ResponseEntity<PointsQueryResponseDTO> queryPoints(
            @Valid @RequestBody PointsQueryRequestDTO request) {
        log.info("Consulta de puntos para clienteId: {}", request.getClientId());
        PointsQueryResponseDTO response = loyaltyReportService.queryPoints(request);
        return ResponseEntity.ok(response);
    }

    /**
     * Registra la redención de puntos de lealtad asociada a una transacción SmartPOS.
     */
    @PostMapping("/points/redeem")
    public ResponseEntity<PointsRedeemResponseDTO> redeemPoints(
            @Valid @RequestBody PointsRedeemRequestDTO request) {
        log.info("Redención de puntos para clienteId: {}, transaccionId: {}",
                request.getClientId(), request.getTransactionId());
        PointsRedeemResponseDTO response = loyaltyReportService.redeemPoints(request);
        return ResponseEntity.ok(response);
    }

    /**
     * Registra la cancelación de puntos de lealtad previamente aplicados.
     */
    @PostMapping("/points/cancel")
    public ResponseEntity<PointsCancelResponseDTO> cancelPoints(
            @Valid @RequestBody PointsCancelRequestDTO request) {
        log.info("Cancelación de puntos para clienteId: {}, transaccionId: {}",
                request.getClientId(), request.getTransactionId());
        PointsCancelResponseDTO response = loyaltyReportService.cancelPoints(request);
        return ResponseEntity.ok(response);
    }
}
```

### 5.4 Interfaz del Servicio: `LoyaltyReportService`

```java
public interface LoyaltyReportService {
    PointsQueryResponseDTO queryPoints(PointsQueryRequestDTO request);
    PointsRedeemResponseDTO redeemPoints(PointsRedeemRequestDTO request);
    PointsCancelResponseDTO cancelPoints(PointsCancelRequestDTO request);
}
```

### 5.5 Feign Client de Integración: `LoyaltyExternalClient`

```java
@FeignClient(
    name = "loyalty-service",
    url = "${loyalty.service.base-url}",
    configuration = LoyaltyClientConfig.class
)
public interface LoyaltyExternalClient {

    @PostMapping("/external/loyalty/query")
    LoyaltyQueryResponse queryPoints(@RequestBody LoyaltyQueryRequest request);

    @PostMapping("/external/loyalty/redeem")
    LoyaltyRedeemResponse redeemPoints(@RequestBody LoyaltyRedeemRequest request);

    @PostMapping("/external/loyalty/cancel")
    LoyaltyCancelResponse cancelPoints(@RequestBody LoyaltyCancelRequest request);
}
```

---

## 6. Contratos de API — Nuevos Endpoints

### 6.1 Consulta de Puntos

**Endpoint:** `POST /api/v1/loyalty/points/query`

**Request Body:**

```json
{
  "clientId": "CLI-00123456",
  "accountNumber": "4012001234567890",
  "requestId": "REQ-20260327-001"
}
```

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `clientId` | String | Sí | Identificador único del cliente bancario |
| `accountNumber` | String | Sí | Número de cuenta o tarjeta asociada |
| `requestId` | String | Sí | Identificador único de la solicitud (idempotencia) |

**Response Body (HTTP 200):**

```json
{
  "clientId": "CLI-00123456",
  "availablePoints": 15420,
  "expiringPoints": 1200,
  "expirationDate": "2026-06-30",
  "loyaltyTier": "GOLD",
  "lastMovementDate": "2026-03-20",
  "requestId": "REQ-20260327-001",
  "timestamp": "2026-03-27T10:35:00Z"
}
```

---

### 6.2 Redención de Puntos

**Endpoint:** `POST /api/v1/loyalty/points/redeem`

**Request Body:**

```json
{
  "clientId": "CLI-00123456",
  "accountNumber": "4012001234567890",
  "transactionId": "TXN-20260327-789456",
  "pointsToRedeem": 500,
  "merchantId": "COM-00789",
  "posTerminalId": "POS-001234",
  "requestId": "REQ-20260327-002"
}
```

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `clientId` | String | Sí | Identificador único del cliente bancario |
| `accountNumber` | String | Sí | Número de cuenta o tarjeta asociada |
| `transactionId` | String | Sí | ID de la transacción SmartPOS a la que se asocia el canje |
| `pointsToRedeem` | Integer | Sí | Cantidad de puntos a canjear (mínimo 1) |
| `merchantId` | String | Sí | Identificador del comercio donde se realiza la transacción |
| `posTerminalId` | String | Sí | Identificador del terminal SmartPOS |
| `requestId` | String | Sí | Identificador único de la solicitud (idempotencia) |

**Response Body (HTTP 200):**

```json
{
  "redemptionId": "RED-20260327-00112",
  "clientId": "CLI-00123456",
  "transactionId": "TXN-20260327-789456",
  "pointsRedeemed": 500,
  "remainingPoints": 14920,
  "monetaryEquivalent": 50.00,
  "currency": "MXN",
  "status": "APPROVED",
  "requestId": "REQ-20260327-002",
  "timestamp": "2026-03-27T10:36:00Z"
}
```

---

### 6.3 Cancelación de Puntos

**Endpoint:** `POST /api/v1/loyalty/points/cancel`

**Request Body:**

```json
{
  "clientId": "CLI-00123456",
  "accountNumber": "4012001234567890",
  "transactionId": "TXN-20260327-789456",
  "redemptionId": "RED-20260327-00112",
  "cancellationReason": "TRANSACTION_REVERSED",
  "requestId": "REQ-20260327-003"
}
```

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `clientId` | String | Sí | Identificador único del cliente bancario |
| `accountNumber` | String | Sí | Número de cuenta o tarjeta asociada |
| `transactionId` | String | Sí | ID de la transacción SmartPOS original |
| `redemptionId` | String | Sí | ID de la redención a cancelar |
| `cancellationReason` | Enum | Sí | Motivo de cancelación: `TRANSACTION_REVERSED`, `CLIENT_REQUEST`, `FRAUD_DETECTION`, `SYSTEM_ERROR` |
| `requestId` | String | Sí | Identificador único de la solicitud (idempotencia) |

**Response Body (HTTP 200):**

```json
{
  "cancellationId": "CAN-20260327-00045",
  "clientId": "CLI-00123456",
  "transactionId": "TXN-20260327-789456",
  "redemptionId": "RED-20260327-00112",
  "pointsRestored": 500,
  "updatedBalance": 15420,
  "status": "CANCELLED",
  "requestId": "REQ-20260327-003",
  "timestamp": "2026-03-27T10:40:00Z"
}
```

---

## 7. Modelo de Datos

### 7.1 DTOs Principales (Capa API)

#### `PointsQueryRequestDTO`

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PointsQueryRequestDTO {

    @NotBlank(message = "El clientId es obligatorio")
    private String clientId;

    @NotBlank(message = "El número de cuenta es obligatorio")
    private String accountNumber;

    @NotBlank(message = "El requestId es obligatorio")
    private String requestId;
}
```

#### `PointsRedeemRequestDTO`

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PointsRedeemRequestDTO {

    @NotBlank
    private String clientId;

    @NotBlank
    private String accountNumber;

    @NotBlank
    private String transactionId;

    @NotNull
    @Min(value = 1, message = "La cantidad mínima de puntos a canjear es 1")
    private Integer pointsToRedeem;

    @NotBlank
    private String merchantId;

    @NotBlank
    private String posTerminalId;

    @NotBlank
    private String requestId;
}
```

#### `PointsCancelRequestDTO`

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PointsCancelRequestDTO {

    @NotBlank
    private String clientId;

    @NotBlank
    private String accountNumber;

    @NotBlank
    private String transactionId;

    @NotBlank
    private String redemptionId;

    @NotNull
    private CancellationReason cancellationReason;

    @NotBlank
    private String requestId;
}
```

#### `CancellationReason` (Enum)

```java
public enum CancellationReason {
    TRANSACTION_REVERSED,
    CLIENT_REQUEST,
    FRAUD_DETECTION,
    SYSTEM_ERROR
}
```

### 7.2 Enriquecimiento de DTOs Existentes

Para integrar la información de lealtad en los reportes existentes sin romper contratos actuales, se agrega un campo opcional en los DTOs de respuesta de los tres controllers existentes:

```java
// Fragmento aplicable a TransactionReportResponseDTO,
// SummaryReportResponseDTO y ClientProfileReportResponseDTO

@JsonInclude(JsonInclude.Include.NON_NULL)
private LoyaltySummaryDTO loyaltyInfo;
```

#### `LoyaltySummaryDTO`

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class LoyaltySummaryDTO {

    private Integer pointsEarned;
    private Integer pointsRedeemed;
    private Integer pointsCancelled;
    private Integer currentBalance;
    private String loyaltyTier;
    private String lastOperationType;    // QUERY | REDEEM | CANCEL
    private String lastOperationDate;
}
```

---

## 8. Integración con los Reportes Existentes

### 8.1 `TransactionReportController`

Cuando se consulta el detalle de una transacción específica (`GET /api/v1/reports/transactions/{transactionId}`), la respuesta incluirá opcionalmente el objeto `loyaltyInfo` con el resumen de puntos asociados a dicha transacción.

**Cambio en la lógica del servicio existente:**

```java
// En TransactionReportServiceImpl, al construir la respuesta:
LoyaltySummaryDTO loyaltySummary = loyaltyReportService
    .getLoyaltySummaryForTransaction(transaction.getClientId(),
                                     transaction.getTransactionId());
response.setLoyaltyInfo(loyaltySummary); // Null si no hay movimiento de lealtad
```

### 8.2 `ClientProfileReportController`

El perfil del cliente en `GET /api/v1/reports/clients/{clientId}/profile` incluirá el historial resumido del programa de lealtad mediante el campo `loyaltyInfo` que muestra saldo actual, nivel y último movimiento.

### 8.3 `SummaryReportController`

Los reportes consolidados incluirán estadísticas de lealtad agrupadas por periodo (total de puntos emitidos, canjeados y cancelados) como sección adicional en la respuesta.

---

## 9. Manejo de Errores

### 9.1 Excepciones Nuevas

| Excepción | Código HTTP | Código de Error | Escenario |
|---|---|---|---|
| `LoyaltyServiceException` | 503 | `LOYALTY_SERVICE_UNAVAILABLE` | El servicio externo de lealtad no responde |
| `InsufficientPointsException` | 422 | `INSUFFICIENT_POINTS` | El cliente no tiene suficientes puntos para la redención solicitada |
| `RedemptionNotFoundException` | 404 | `REDEMPTION_NOT_FOUND` | No se encuentra la redención a cancelar |
| `DuplicateRequestException` | 409 | `DUPLICATE_REQUEST` | El `requestId` ya fue procesado anteriormente |

### 9.2 Formato de Respuesta de Error

Todas las excepciones devuelven el siguiente formato estandarizado (coherente con el manejo de errores existente del microservicio):

```json
{
  "timestamp": "2026-03-27T10:45:00Z",
  "status": 422,
  "errorCode": "INSUFFICIENT_POINTS",
  "message": "El cliente no cuenta con puntos suficientes para completar la operación.",
  "path": "/api/v1/loyalty/points/redeem",
  "requestId": "REQ-20260327-002"
}
```

### 9.3 `GlobalExceptionHandler` — Adiciones

```java
// Agregar a la clase @ControllerAdvice existente:

@ExceptionHandler(LoyaltyServiceException.class)
@ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
public ErrorResponseDTO handleLoyaltyServiceException(LoyaltyServiceException ex,
                                                       HttpServletRequest request) {
    return buildErrorResponse(HttpStatus.SERVICE_UNAVAILABLE,
            "LOYALTY_SERVICE_UNAVAILABLE", ex.getMessage(), request.getRequestURI());
}

@ExceptionHandler(InsufficientPointsException.class)
@ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
public ErrorResponseDTO handleInsufficientPoints(InsufficientPointsException ex,
                                                   HttpServletRequest request) {
    return buildErrorResponse(HttpStatus.UNPROCESSABLE_ENTITY,
            "INSUFFICIENT_POINTS", ex.getMessage(), request.getRequestURI());
}
```

---

## 10. Consideraciones de Seguridad

### 10.1 Autenticación y Autorización

- Los nuevos endpoints heredan el mecanismo de seguridad vigente en el microservicio (JWT Bearer Token).
- Se recomienda definir un rol específico `LOYALTY_OPERATOR` para las operaciones de Redención y Cancelación, dado su impacto transaccional.
- La Consulta de puntos puede operar con el rol estándar `REPORT_VIEWER`.

```java
// Ejemplo en SecurityConfig (Spring Security)
.antMatchers(HttpMethod.POST, "/api/v1/loyalty/points/query").hasRole("REPORT_VIEWER")
.antMatchers(HttpMethod.POST, "/api/v1/loyalty/points/redeem").hasRole("LOYALTY_OPERATOR")
.antMatchers(HttpMethod.POST, "/api/v1/loyalty/points/cancel").hasRole("LOYALTY_OPERATOR")
```

### 10.2 Validación de Datos

- Todos los campos de entrada son validados con Bean Validation (`@Valid`, `@NotBlank`, `@Min`, etc.).
- Los identificadores (`clientId`, `transactionId`, `redemptionId`) deben validarse contra un patrón conocido para prevenir inyecciones.
- Los valores de `pointsToRedeem` tienen límites mínimos y se recomienda establecer un máximo configurable.

### 10.3 Idempotencia

- El campo `requestId` en todos los endpoints de escritura (Redención y Cancelación) garantiza idempotencia.
- Se debe implementar un caché de `requestId` procesados (Redis o tabla de control) con TTL de 24 horas.

### 10.4 Logging y Auditoría

- Las operaciones de Redención y Cancelación deben registrarse en el log de auditoría del microservicio con nivel `INFO`, incluyendo: `clientId`, `transactionId`, `requestId`, usuario autenticado, y resultado de la operación.
- No se deben loggear datos sensibles (número de cuenta completo, tokens).

---

## 11. Estrategia de Pruebas

### 11.1 Pruebas Unitarias

| Clase a Probar | Framework | Cobertura Mínima |
|---|---|---|
| `LoyaltyReportServiceImpl` | JUnit 5 + Mockito | 80% |
| `LoyaltyReportController` | JUnit 5 + MockMvc | 80% |
| `LoyaltyExternalClient` | WireMock | 70% |
| Mappers de DTO | JUnit 5 | 90% |

**Escenarios clave para pruebas unitarias:**
- Consulta exitosa con saldo mayor a cero.
- Consulta de cliente sin puntos acumulados.
- Redención exitosa con puntos suficientes.
- Redención fallida por puntos insuficientes (`InsufficientPointsException`).
- Cancelación exitosa de una redención existente.
- Cancelación de una redención inexistente (`RedemptionNotFoundException`).
- Reintento con `requestId` duplicado (`DuplicateRequestException`).
- Timeout / indisponibilidad del servicio externo de lealtad (`LoyaltyServiceException`).

### 11.2 Pruebas de Integración

- Pruebas con Spring Boot Test (`@SpringBootTest`) que validen el flujo completo desde el controller hasta el cliente externo mockeado con WireMock.
- Validación de los campos de lealtad enriquecidos en las respuestas de los tres controllers existentes.

### 11.3 Pruebas de Contrato (Contract Testing)

Se recomienda usar **Spring Cloud Contract** o **Pact** para definir contratos entre este microservicio y el servicio externo de lealtad, garantizando la estabilidad de la integración ante cambios en ambos lados.

### 11.4 Pruebas en Ambiente QA

- Verificar el flujo Consulta → Redención → Cancelación sobre un cliente de prueba con datos controlados.
- Validar que los reportes existentes (`TransactionReport`, `SummaryReport`, `ClientProfileReport`) muestran correctamente el campo `loyaltyInfo` cuando existe información de lealtad y lo omiten (`null`) cuando no.

---

## 12. Plan de Implementación

### 12.1 Fases de Desarrollo

| Fase | Actividad | Responsable | Estimado |
|---|---|---|---|
| 1 | Configuración de dependencias y estructura de paquetes | Backend Dev | 0.5 día |
| 2 | Implementación de DTOs y enums | Backend Dev | 0.5 día |
| 3 | Implementación de `LoyaltyExternalClient` (Feign) | Backend Dev | 1 día |
| 4 | Implementación de `LoyaltyReportService` | Backend Dev | 1 día |
| 5 | Implementación de `LoyaltyReportController` | Backend Dev | 0.5 día |
| 6 | Enriquecimiento de DTOs y servicios existentes | Backend Dev | 1 día |
| 7 | Manejo de errores y auditoría | Backend Dev | 0.5 día |
| 8 | Pruebas unitarias e integración | Backend Dev / QA | 2 días |
| 9 | Revisión de código (Code Review) | Tech Lead | 0.5 día |
| 10 | Despliegue en QA y pruebas funcionales | QA | 1 día |
| 11 | Aprobación y despliegue a Producción | DevOps / Tech Lead | 0.5 día |

**Estimado total:** ~9 días hábiles

### 12.2 Propiedades de Configuración Requeridas

```yaml
# application.yml — Nuevas propiedades a agregar

loyalty:
  service:
    base-url: ${LOYALTY_SERVICE_URL:https://loyalty-api.banco.internal}
    connect-timeout: 3000      # ms
    read-timeout: 5000         # ms
    retry:
      max-attempts: 2
      wait-duration: 1000      # ms
  cache:
    request-id-ttl: 86400      # segundos (24 horas)
  points:
    max-redeem-per-transaction: 10000
```

### 12.3 Dependencias a Agregar en `pom.xml`

```xml
<!-- Spring Cloud OpenFeign -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<!-- Resilience4j para Circuit Breaker (si no está incluido) -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
</dependency>

<!-- WireMock para pruebas (scope test) -->
<dependency>
    <groupId>com.github.tomakehurst</groupId>
    <artifactId>wiremock-jre8</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 13. Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Indisponibilidad del servicio externo de lealtad | Media | Alto | Implementar Circuit Breaker con Resilience4j. Los reportes devolverán `loyaltyInfo: null` ante falla, sin interrumpir el flujo principal. |
| Latencia alta en respuestas del servicio de lealtad | Media | Medio | Timeouts configurables y llamadas asíncronas para el enriquecimiento de reportes (CompletableFuture). |
| Cambios en el contrato del API externo de lealtad | Baja | Alto | Definir contratos con Spring Cloud Contract / Pact. |
| Regresión en controllers existentes | Baja | Alto | Los campos de lealtad son opcionales (`NON_NULL`). Suite de pruebas de regresión automatizada antes del despliegue. |
| Procesamiento duplicado de reembolsos/cancelaciones | Baja | Alto | Control de idempotencia mediante `requestId` con caché distribuido (Redis). |

---

## 14. Glosario

| Término | Definición |
|---|---|
| **SmartPOS** | Terminal punto de venta inteligente utilizado por comercios afiliados para procesar pagos bancarios. |
| **Programa de Lealtad** | Sistema de acumulación y canje de puntos ofrecido por el banco a sus clientes por transacciones realizadas. |
| **Redención** | Acción de canjear puntos acumulados por un beneficio o descuento en una transacción. |
| **Cancelación de Puntos** | Reversión de una redención previamente aplicada, devolviendo los puntos al saldo del cliente. |
| **Feign Client** | Componente de Spring Cloud que permite declarar clientes HTTP de forma declarativa mediante interfaces. |
| **Circuit Breaker** | Patrón de diseño que evita llamadas a un servicio externo no disponible, protegiendo la estabilidad del sistema. |
| **Idempotencia** | Propiedad de una operación que garantiza el mismo resultado aunque sea ejecutada múltiples veces con los mismos parámetros. |
| **DTO** | Data Transfer Object — objeto utilizado para transportar datos entre capas de la aplicación. |
| **JWT** | JSON Web Token — estándar de autenticación basado en tokens firmados digitalmente. |

---

*Documento generado para uso interno del equipo de desarrollo. Versión 1.0.0 — Marzo 2026.*
