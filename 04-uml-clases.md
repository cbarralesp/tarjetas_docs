# 04 · Diagramas UML de clases

> Para ver los diagramas en VS Code instala la extensión **"Markdown Preview Mermaid Support"** y abre la vista previa (`Ctrl+Shift+V`). GitHub también los muestra.

## 1. Backend completo (MVC en capas)

```mermaid
classDiagram
    direction TB

    class TarjetaBipController {
        -TarjetaBipService service
        +listar() List~TarjetaResponse~
        +obtener(id) TarjetaResponse
        +crear(request) ResponseEntity
        +actualizar(id, request) TarjetaResponse
        +eliminar(id) ResponseEntity
    }

    class TarjetaBipService {
        <<interface>>
        +listar() List~TarjetaResponse~
        +obtener(id) TarjetaResponse
        +crear(request) TarjetaResponse
        +actualizar(id, request) TarjetaResponse
        +eliminar(id) void
    }

    class TarjetaBipServiceImpl {
        -TarjetaBipRepository repository
        -TarjetaBipMapper mapper
        -buscar(id) TarjetaBip
    }

    class TarjetaBipMapper {
        +aEntidad(CrearTarjetaRequest) TarjetaBip
        +actualizarEntidad(entidad, ActualizarTarjetaRequest) void
        +aRespuesta(TarjetaBip) TarjetaResponse
    }

    class TarjetaBipRepository {
        <<interface · Spring Data>>
        +existsByNumero(numero) boolean
        +save(t) TarjetaBip
        +findById(id) Optional
        +findAll(sort) List
        +existsById(id) boolean
        +deleteById(id) void
    }

    class TarjetaBip {
        <<Entity>>
        -Long id
        -String numero
        -String titular
        -String alias
        -TipoTarjeta tipo
        -BigDecimal saldo
        -EstadoTarjeta estado
        -LocalDateTime creadaEn
        -LocalDateTime actualizadaEn
        +antesDeGuardar() void
        +antesDeActualizar() void
    }

    class TipoTarjeta {
        <<enumeration>>
        NORMAL
        TNE_ESCOLAR
        TNE_SUPERIOR
        ADULTO_MAYOR
    }
    class EstadoTarjeta {
        <<enumeration>>
        ACTIVA
        BLOQUEADA
    }

    class CrearTarjetaRequest {
        <<record · DTO>>
        numero
        titular
        alias
        tipo
        saldo
    }
    class ActualizarTarjetaRequest {
        <<record · DTO>>
        titular
        alias
        tipo
        saldo
        estado
    }
    class TarjetaResponse {
        <<record · DTO>>
        id
        numero
        titular
        alias
        tipo
        saldo
        estado
        creadaEn
        actualizadaEn
    }

    class GlobalExceptionHandler {
        <<RestControllerAdvice>>
        +noEncontrada()
        +duplicada()
        +integridad()
        +validacion()
        +solicitudMalFormada()
        +inesperado()
    }
    class ApiError {
        <<record>>
        fecha
        status
        error
        mensaje
        detalles
    }
    class TarjetaNoEncontradaException
    class NumeroTarjetaDuplicadoException
    class CorsConfig
    class SwaggerConfig

    TarjetaBipController --> TarjetaBipService : usa
    TarjetaBipController ..> CrearTarjetaRequest
    TarjetaBipController ..> ActualizarTarjetaRequest
    TarjetaBipController ..> TarjetaResponse
    TarjetaBipServiceImpl ..|> TarjetaBipService
    TarjetaBipServiceImpl --> TarjetaBipRepository
    TarjetaBipServiceImpl --> TarjetaBipMapper
    TarjetaBipServiceImpl ..> TarjetaNoEncontradaException
    TarjetaBipServiceImpl ..> NumeroTarjetaDuplicadoException
    TarjetaBipMapper ..> TarjetaBip
    TarjetaBipMapper ..> TarjetaResponse
    TarjetaBipRepository ..> TarjetaBip
    TarjetaBip --> TipoTarjeta
    TarjetaBip --> EstadoTarjeta
    GlobalExceptionHandler ..> ApiError
    GlobalExceptionHandler ..> TarjetaNoEncontradaException
    GlobalExceptionHandler ..> NumeroTarjetaDuplicadoException
```

### Cómo leerlo

- Línea punteada con triángulo (`..|>`) = **implementa** una interfaz.
- Flecha continua (`-->`) = **tiene / usa** una referencia (inyectada por constructor).
- Flecha punteada (`..>`) = **depende** (la usa en algún método).
- El flujo siempre baja: **Controller → Service → Repository → Entity**. Nunca al revés.

## 2. La Entity y sus enums

```mermaid
classDiagram
    class TarjetaBip {
        <<Entity>>
        -Long id
        -String numero
        -String titular
        -String alias
        -TipoTarjeta tipo
        -BigDecimal saldo
        -EstadoTarjeta estado
        -LocalDateTime creadaEn
        -LocalDateTime actualizadaEn
        +antesDeGuardar() void
        +antesDeActualizar() void
    }
    class TipoTarjeta {
        <<enumeration>>
        NORMAL
        TNE_ESCOLAR
        TNE_SUPERIOR
        ADULTO_MAYOR
    }
    class EstadoTarjeta {
        <<enumeration>>
        ACTIVA
        BLOQUEADA
    }
    TarjetaBip "1" --> "1" TipoTarjeta
    TarjetaBip "1" --> "1" EstadoTarjeta
```

## 3. Excepciones y su traducción a HTTP

```mermaid
classDiagram
    class RuntimeException
    class TarjetaNoEncontradaException { HTTP 404 }
    class NumeroTarjetaDuplicadoException { HTTP 409 }
    RuntimeException <|-- TarjetaNoEncontradaException
    RuntimeException <|-- NumeroTarjetaDuplicadoException
```

Además: validaciones de `@Valid` → **400**, JSON mal formado → **400**, violación de integridad en la BD → **409**, cualquier otro error → **500**.

## 4. Frontend Angular (MVC por feature)

```mermaid
classDiagram
    direction TB
    class TarjetasPage {
        <<página · Controller>>
        -porEliminar signal
        +buscar() filtrar()
        +pedirConfirmacion(t)
        +confirmarEliminacion()
    }
    class TarjetaFormPage {
        <<página · Controller>>
        +id input
        -form FormGroup
        +guardar()
    }
    class TarjetaBipCard {
        <<componente>>
        +tarjeta input
        +eliminar output
    }
    class EstadoBadge
    class ConfirmDialog
    class ClpPipe
    class Toast
    class ToastService
    class TarjetasStore {
        <<Store · signals>>
        +tarjetas signal
        +busqueda signal
        +filtroEstado signal
        +tarjetasFiltradas computed
        +resumen computed
        +cargar()
        +obtener(id) Observable
        +crear(r) Observable
        +actualizar(id, r) Observable
        +eliminar(id) Observable
    }
    class TarjetaApi {
        <<Api>>
        -HttpClient http
        +listar() Observable
        +obtener(id) Observable
        +crear(r) Observable
        +actualizar(id, r) Observable
        +eliminar(id) Observable
    }
    class errorInterceptor
    class ApiError
    class Tarjeta {
        <<Model · interface>>
    }
    class TarjetaRequest {
        <<Model · interface>>
    }
    class TarjetaEdicionRequest {
        <<Model · interface>>
    }

    TarjetasPage --> TarjetasStore
    TarjetaFormPage --> TarjetasStore
    TarjetasPage --> TarjetaBipCard
    TarjetasPage --> ConfirmDialog
    TarjetaBipCard --> EstadoBadge
    TarjetaBipCard --> ClpPipe
    TarjetasPage --> ClpPipe
    TarjetasStore --> TarjetaApi
    TarjetasStore --> ToastService
    Toast --> ToastService
    TarjetaApi ..> Tarjeta
    TarjetaApi ..> TarjetaRequest
    TarjetaApi ..> TarjetaEdicionRequest
    errorInterceptor ..> ApiError
```

## 5. Diagrama de despliegue lógico

```mermaid
flowchart LR
    subgraph Navegador
        NG["Angular 21<br/>localhost:4200<br/>(proxy /api → :8082)"]
    end
    subgraph Servidor
        API["Spring Boot · Java 21<br/>localhost:8082<br/>+ Swagger UI"]
        FW["Flyway<br/>(migraciones)"]
    end
    DB[("PostgreSQL 18<br/>tarjetas_db")]
    NG -- "HTTP · JSON<br/>/api/tarjetas (vía proxy)" --> API
    API -- "JDBC" --> DB
    FW -- "crea tablas al arrancar" --> DB
```

Anterior: [03 · Patrones](03-patrones-de-diseno.md) · Siguiente: [05 · UML de secuencia](05-uml-secuencia.md)
