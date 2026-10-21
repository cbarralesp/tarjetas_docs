# 05 · Diagramas UML de secuencia

Muestran **quién le habla a quién y en qué orden**. Léelos de arriba hacia abajo.

## 1. Registrar una tarjeta (POST /api/tarjetas) — flujo feliz

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant F as TarjetaFormPage (Angular)
    participant AS as TarjetasStore / TarjetaApi
    participant C as TarjetaBipController
    participant S as TarjetaBipServiceImpl
    participant M as TarjetaBipMapper
    participant R as TarjetaBipRepository
    participant DB as PostgreSQL

    U->>F: Completa el formulario y presiona "Registrar"
    F->>F: Valida el formulario (Validators)
    F->>AS: store.crear(TarjetaRequest)
    AS->>C: POST /api/tarjetas (JSON)
    C->>C: @Valid valida el DTO
    C->>S: crear(CrearTarjetaRequest)
    S->>R: existsByNumero(numero)
    R->>DB: SELECT EXISTS ... WHERE numero = ?
    DB-->>R: false
    S->>M: aEntidad(request)
    M-->>S: TarjetaBip (Entity)
    S->>R: save(entity)
    R->>DB: INSERT INTO tarjeta_bip ...
    DB-->>R: fila con id (y @PrePersist puso fechas y estado ACTIVA)
    S->>M: aRespuesta(entity)
    M-->>S: TarjetaResponse
    S-->>C: TarjetaResponse
    C-->>AS: 201 Created + JSON
    AS->>AS: refresca la lista (GET) y la guarda en el signal
    AS-->>F: emite (salió bien)
    F->>U: Aviso "Tarjeta registrada" y vuelve a /tarjetas
```

## 2. Registrar una tarjeta duplicada (flujo de error 409)

```mermaid
sequenceDiagram
    autonumber
    participant AS as TarjetasStore / TarjetaApi
    participant I as errorInterceptor
    participant C as Controller
    participant S as ServiceImpl
    participant R as Repository
    participant H as GlobalExceptionHandler

    AS->>C: POST /api/tarjetas {numero: "12345678"...}
    C->>S: crear(request)
    S->>R: existsByNumero("12345678")
    R-->>S: true
    S--xC: lanza NumeroTarjetaDuplicadoException
    C--xH: la excepción sube
    H-->>I: 409 {"mensaje": "Ya existe una tarjeta..."}
    I-->>AS: AppError("Ya existe una tarjeta...")
    Note over AS: Aviso rojo con el mensaje; el formulario sigue abierto
```

## 3. Listar tarjetas (GET /api/tarjetas)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant L as TarjetasPage
    participant AS as TarjetasStore / TarjetaApi
    participant C as Controller
    participant S as ServiceImpl
    participant R as Repository
    participant DB as PostgreSQL

    U->>L: Abre /tarjetas
    L->>L: cargando = true
    L->>AS: store.cargar()
    AS->>C: GET /api/tarjetas
    C->>S: listar()
    S->>R: findAll(Sort DESC id)
    R->>DB: SELECT * FROM tarjeta_bip ORDER BY id DESC
    DB-->>R: filas
    R-->>S: List<TarjetaBip>
    S-->>C: List<TarjetaResponse> (vía Mapper)
    C-->>AS: 200 [JSON]
    AS-->>L: Tarjeta[]
    L->>L: tarjetas.set(lista) · cargando = false
    L->>U: Dibuja resumen y tarjetas (signals)
```

## 4. Editar una tarjeta (GET + PUT)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant F as TarjetaFormPage
    participant AS as TarjetasStore / TarjetaApi
    participant C as Controller
    participant S as ServiceImpl
    participant M as Mapper
    participant R as Repository

    U->>F: Presiona "Editar" (/tarjetas/5/editar)
    F->>AS: store.obtener(5)
    AS->>C: GET /api/tarjetas/5
    C->>S: obtener(5)
    S->>R: findById(5)
    R-->>S: TarjetaBip
    C-->>AS: 200 TarjetaResponse
    AS-->>F: Tarjeta
    F->>U: Muestra el formulario con datos (número bloqueado)
    U->>F: Cambia el saldo y presiona "Guardar cambios"
    F->>AS: store.actualizar(5, TarjetaEdicionRequest)
    AS->>C: PUT /api/tarjetas/5
    C->>C: @Valid valida el DTO
    C->>S: actualizar(5, request)
    S->>R: findById(5)
    R-->>S: TarjetaBip existente
    S->>M: actualizarEntidad(entity, request)
    S->>R: save(entity)
    Note right of R: @PreUpdate refresca actualizadaEn
    S-->>C: TarjetaResponse
    C-->>AS: 200 JSON
    AS->>AS: refresca la lista (GET) y la guarda en el signal
    AS-->>F: emite (salió bien)
    F->>U: Aviso "Tarjeta actualizada" y vuelve al listado
```

## 5. Eliminar una tarjeta (DELETE)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant L as TarjetasPage
    participant CD as ConfirmDialog
    participant C as Controller
    participant S as ServiceImpl
    participant R as Repository
    participant DB as PostgreSQL

    U->>L: Presiona "Eliminar" en una tarjeta
    L->>CD: muestra el diálogo de confirmación
    alt El usuario cancela
        U->>CD: Cancelar
        CD-->>L: cancelar
    else El usuario confirma
        U->>CD: Eliminar
        CD-->>L: confirmar
        L->>AS: store.eliminar(id)
        AS->>C: DELETE /api/tarjetas/{id}
        C->>S: eliminar(id)
        S->>R: existsById(id)
        R-->>S: true
        S->>R: deleteById(id)
        R->>DB: DELETE FROM tarjeta_bip WHERE id = ?
        C-->>AS: 204 No Content
        AS->>AS: refresca la lista (GET)
        AS-->>L: emite (salió bien)
        L->>U: Aviso "Tarjeta eliminada"
    end
```

## 6. Arranque del backend

```mermaid
sequenceDiagram
    participant M as main()
    participant SP as Spring Boot
    participant FW as Flyway
    participant DB as PostgreSQL
    participant HB as Hibernate

    M->>SP: SpringApplication.run
    SP->>FW: ejecuta migraciones
    FW->>DB: V1 crea tabla · V2 inserta datos
    SP->>HB: ddl-auto = validate
    HB->>DB: compara Entities con tablas
    SP->>SP: crea beans e inyecta dependencias
    SP-->>M: Tomcat escuchando en :8082
```

Anterior: [04 · UML de clases](04-uml-clases.md) · Siguiente: [06 · Modelo de base de datos](06-modelo-base-de-datos.md)
