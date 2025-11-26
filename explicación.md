# Explicación Completa del Sistema - Obligatorio 3

## 📋 Índice
1. [Arquitectura General](#arquitectura-general)
2. [Protocolos y Tecnologías](#protocolos-y-tecnologías)
3. [Componentes del Sistema](#componentes-del-sistema)
4. [Flujos de Comunicación](#flujos-de-comunicación)
5. [Justificación de Decisiones](#justificación-de-decisiones)

---

## 🏗️ Arquitectura General

### Visión General
El sistema está compuesto por **5 módulos principales** que se comunican entre sí usando diferentes protocolos según sus necesidades:

```
┌─────────────┐
│   Cliente   │──TCP (puerto 30000)──┐
└─────────────┘                      │
                                     ▼
┌─────────────┐              ┌──────────────┐
│ ClienteChat │──WebSocket───▶│ ServidorChat │──gRPC──▶┌──────────┐
└─────────────┘   (puerto    └──────────────┘ (puerto  │ Servidor │
                         5002)             5001)        └──────────┘
                                                              │
                                                              │ RabbitMQ
                                                              │ (puerto 5672)
                                                              ▼
                                                       ┌──────────────┐
                                                       │ ServidorLogs│
                                                       └──────────────┘
                                                              │
                                                              │ REST API
                                                              │ (puerto 5000)
                                                              ▼
                                                       [Consultas remotas]
```

### Módulos del Sistema

1. **Servidor Principal** (puerto 30000)
   - Gestiona usuarios, clases, inscripciones
   - Protocolo TCP personalizado
   - Publica logs a RabbitMQ
   - Expone servicio gRPC para autenticación
   - Gestiona webhooks

2. **Cliente** 
   - Aplicación de consola
   - Se conecta al servidor vía TCP
   - Realiza todas las operaciones CRUD de clases

3. **Servidor de Chat** (puerto 5002)
   - Gestiona salas de chat en tiempo real
   - WebSocket para comunicación bidireccional
   - Valida usuarios y clases vía gRPC

4. **Cliente de Chat**
   - Aplicación de consola
   - Se conecta al servidor de chat vía WebSocket
   - Permite chatear en clases online

5. **Servidor de Logs** (puerto 5000)
   - Consume logs de RabbitMQ
   - Almacena logs en memoria
   - Expone REST API para consultas

---

## 🔌 Protocolos y Tecnologías

### 1. **TCP (Protocolo Personalizado)**
**Dónde se usa:** Cliente ↔ Servidor Principal

**Por qué:**
- **Control total** sobre el protocolo
- **Eficiencia** para operaciones síncronas request-response
- **Simplicidad** para operaciones CRUD tradicionales
- **Bajo overhead** comparado con HTTP para este caso de uso

**Cómo funciona:**
- Protocolo basado en **frames** con estructura:
  ```
  [HEADER: 3 bytes][CMD: 2 bytes][LEN: 4 bytes][DATA: variable]
  ```
- HEADER: `"REQ"` (request) o `"RES"` (response)
- CMD: Código numérico del comando (01=Login, 02=Register, etc.)
- LEN: Longitud del payload en bytes
- DATA: Payload en texto plano

**Ejemplo de flujo:**
```
Cliente → Servidor: REQ|01|0005|user1
Servidor → Cliente: RES|01|0008|ASK_PASS
Cliente → Servidor: REQ|01|0008|password
Servidor → Cliente: RES|01|0002|OK
```

**Ventajas:**
- ✅ Comunicación directa y eficiente
- ✅ Sin overhead de HTTP/JSON
- ✅ Protocolo binario compacto
- ✅ Ideal para aplicaciones cliente-servidor tradicionales

---

### 2. **RabbitMQ (Message-Oriented Middleware - MOM)**
**Dónde se usa:** Servidor Principal → Servidor de Logs

**Por qué:**
- **Desacoplamiento**: El servidor principal no necesita saber si el servidor de logs está disponible
- **Asincronía**: Los logs se publican sin bloquear operaciones principales
- **Escalabilidad**: Múltiples consumidores pueden procesar logs
- **Confiabilidad**: RabbitMQ garantiza la entrega de mensajes
- **Persistencia opcional**: Los mensajes pueden persistirse si el consumidor está offline

**Cómo funciona:**
1. **Servidor Principal** (Productor):
   ```csharp
   // En LogPublisher.cs
   LogPublisher.Publish(new LogEvent {
       TimestampUtc = DateTime.UtcNow,
       Action = "LOGIN",
       User = "usuario1",
       Level = "INFO"
   });
   ```
   - Serializa el log a JSON
   - Publica en la cola `server-logs` de RabbitMQ
   - No espera confirmación (fire-and-forget)

2. **Servidor de Logs** (Consumidor):
   ```csharp
   // En RabbitLogConsumer.cs
   // BackgroundService que escucha continuamente
   consumer.Received += async (sender, ea) => {
       var log = Deserialize(ea.Body);
       _store.Add(log);
       channel.BasicAck(ea.DeliveryTag);
   };
   ```
   - Escucha la cola `server-logs`
   - Procesa mensajes asincrónicamente
   - Almacena en memoria

**Ventajas:**
- ✅ El servidor principal no se bloquea esperando que se procesen logs
- ✅ Si el servidor de logs cae, los mensajes se acumulan en RabbitMQ
- ✅ Fácil agregar más consumidores (ej: análisis, alertas)
- ✅ Patrón Producer-Consumer estándar

---

### 3. **gRPC (Remote Procedure Call)**
**Dónde se usa:** ServidorChat ↔ Servidor Principal

**Por qué:**
- **Performance**: Protocolo binario (Protocol Buffers) más rápido que JSON
- **Tipado fuerte**: Contratos definidos en `.proto` files
- **Streaming**: Soporta comunicación bidireccional (aunque no lo usamos aquí)
- **HTTP/2**: Usa HTTP/2 que es más eficiente que HTTP/1.1
- **Multiplataforma**: Funciona en cualquier lenguaje

**Cómo funciona:**
1. **Definición del contrato** (`auth.proto`):
   ```protobuf
   service AuthService {
     rpc ValidateUser(ValidateUserRequest) returns (ValidateUserResponse);
     rpc ValidateClassLink(ValidateClassLinkRequest) returns (ValidateClassLinkResponse);
   }
   ```

2. **Servidor Principal** (Servidor gRPC):
   ```csharp
   // En Program.cs del Servidor
   builder.Services.AddGrpc();
   app.MapGrpcService<GrpcAuthService>();
   ```
   - Escucha en puerto 5001
   - Implementa `GrpcAuthService` que valida usuarios y clases

3. **ServidorChat** (Cliente gRPC):
   ```csharp
   // En ServidorChat/Program.cs
   using var channel = GrpcChannel.ForAddress("http://servidor:5001");
   var client = new AuthService.AuthServiceClient(channel);
   var response = await client.ValidateUserAsync(new ValidateUserRequest {
       Username = username,
       Password = password
   });
   ```

**Flujo de autenticación en chat:**
```
ClienteChat → ServidorChat (WebSocket): {"Type": "AUTH", "Username": "...", "Password": "..."}
ServidorChat → Servidor (gRPC): ValidateUser(username, password)
Servidor → ServidorChat (gRPC): {Valid: true}
ServidorChat → ClienteChat (WebSocket): {"Type": "AUTH_RESPONSE", "Success": true}
```

**Ventajas:**
- ✅ Muy rápido (binario, HTTP/2)
- ✅ Contratos explícitos (menos errores)
- ✅ Ideal para comunicación entre servicios
- ✅ Soporte nativo en .NET

---

### 4. **WebSocket**
**Dónde se usa:** ClienteChat ↔ ServidorChat

**Por qué:**
- **Tiempo real**: Comunicación bidireccional instantánea
- **Bajo latency**: Sin overhead de HTTP request/response
- **Persistente**: Conexión abierta permite push de mensajes
- **Ideal para chat**: Los mensajes llegan inmediatamente a todos los participantes

**Cómo funciona:**
1. **Conexión inicial:**
   ```csharp
   // ClienteChat se conecta
   var ws = new ClientWebSocket();
   await ws.ConnectAsync(new Uri("ws://servidor-chat:5002/ws"), ...);
   ```

2. **Mensajes JSON:**
   ```json
   // Autenticación
   {"Type": "AUTH", "Username": "user1", "Password": "pass1"}
   
   // Unirse a clase
   {"Type": "JOIN", "ClassLink": "https://clases/abc123"}
   
   // Enviar mensaje
   {"Type": "MESSAGE", "Message": "Hola a todos!"}
   ```

3. **Broadcast en sala:**
   ```csharp
   // ServidorChat mantiene salas por classId
   var room = _rooms[classId]; // ConcurrentDictionary<WebSocket, string>
   foreach (var socket in room.Keys) {
       await socket.SendAsync(messageBytes, ...);
   }
   ```

**Flujo de chat:**
```
Usuario A → ServidorChat: {"Type": "MESSAGE", "Message": "Hola"}
ServidorChat → Usuario A: (echo, opcional)
ServidorChat → Usuario B: {"Type": "MESSAGE", "Username": "A", "Message": "Hola"}
ServidorChat → Usuario C: {"Type": "MESSAGE", "Username": "A", "Message": "Hola"}
```

**Ventajas:**
- ✅ Tiempo real verdadero (sin polling)
- ✅ Eficiente para chat (una conexión, múltiples mensajes)
- ✅ Soporte nativo en navegadores y .NET
- ✅ Menor overhead que HTTP para comunicación continua

---

### 5. **REST API**
**Dónde se usa:** Acceso remoto al Servidor de Logs

**Por qué:**
- **Estándar universal**: Cualquier cliente HTTP puede consumirlo
- **Fácil de probar**: Postman, curl, navegador
- **Stateless**: Cada request es independiente
- **Filtrado flexible**: Query parameters para filtros complejos

**Cómo funciona:**
1. **Endpoints disponibles:**
   ```csharp
   // GET /logs?user=usuario1&action=LOGIN&from=2025-01-01&to=2025-01-31
   app.MapGet("/logs", ([AsParameters] LogQuery query, InMemoryLogStore store) => {
       var result = store.Query(query);
       return Results.Ok(result);
   });
   ```

2. **Filtros soportados:**
   - `user`: Filtrar por usuario
   - `classId`: Filtrar por clase
   - `action`: Filtrar por acción (LOGIN, ENROLL, etc.)
   - `level`: Filtrar por nivel (INFO, WARN, ERROR)
   - `from` / `to`: Rango de fechas
   - `text`: Búsqueda de texto libre
   - `limit`: Límite de resultados

3. **Ejemplo de uso:**
   ```bash
   # Obtener logs de un usuario
   curl "http://localhost:5000/logs?user=usuario1&limit=50"
   
   # Logs de una clase específica
   curl "http://localhost:5000/logs?classId=abc123"
   
   # Logs de errores en un rango de fechas
   curl "http://localhost:5000/logs?level=ERROR&from=2025-01-01&to=2025-01-31"
   ```

**Ventajas:**
- ✅ Accesible desde cualquier herramienta HTTP
- ✅ Fácil de documentar y probar
- ✅ Flexible para consultas complejas
- ✅ Estándar de la industria

---

## 🧩 Componentes del Sistema

### Servidor Principal

**Responsabilidades:**
- Gestión de usuarios (registro, login)
- CRUD de clases online
- Inscripciones y cancelaciones
- Subida/descarga de imágenes
- Reportes diarios
- Publicación de logs a RabbitMQ
- Servicio gRPC para autenticación
- Notificación de webhooks

**Tecnologías:**
- **TCP** (puerto 30000): Comunicación con clientes
- **gRPC** (puerto 5001): Servicio de autenticación para ServidorChat
- **RabbitMQ**: Publicación de logs

**Archivos clave:**
- `Program.cs`: Loop principal, handlers de comandos TCP, servidor gRPC
- `LogPublisher.cs`: Publicación asíncrona de logs a RabbitMQ
- `GrpcAuthService.cs`: Implementación del servicio gRPC
- `WebhookNotifier.cs`: Verificación y llamada de webhooks

---

### Cliente

**Responsabilidades:**
- Interfaz de usuario (consola)
- Comunicación con servidor vía TCP
- Operaciones CRUD de clases
- Subida/descarga de imágenes
- Heartbeat para detectar caídas del servidor

**Tecnologías:**
- **TCP**: Comunicación con servidor

**Características:**
- Protocolo frame-based
- Manejo de errores de red
- Heartbeat cada 1 segundo para verificar disponibilidad

---

### Servidor de Chat

**Responsabilidades:**
- Gestión de salas de chat por clase
- Autenticación de usuarios (vía gRPC)
- Validación de links de clases (vía gRPC)
- Broadcast de mensajes en tiempo real
- Gestión de conexiones WebSocket

**Tecnologías:**
- **WebSocket** (puerto 5002): Comunicación con clientes de chat
- **gRPC** (cliente): Validación con servidor principal

**Arquitectura:**
```csharp
// Estructura de salas
ConcurrentDictionary<string, ConcurrentDictionary<WebSocket, string>> _rooms;
// classId → { WebSocket → Username }
```

**Flujo de mensajes:**
1. Cliente se autentica (WebSocket → gRPC → WebSocket)
2. Cliente se une a clase (WebSocket → gRPC → WebSocket)
3. Cliente envía mensaje → Broadcast a todos en la sala

---

### Cliente de Chat

**Responsabilidades:**
- Interfaz de usuario (consola)
- Conexión WebSocket al servidor de chat
- Autenticación
- Envío/recepción de mensajes en tiempo real

**Tecnologías:**
- **WebSocket**: Comunicación con servidor de chat

**Características:**
- Thread separado para recibir mensajes
- Manejo de desconexiones
- Interfaz de consola interactiva

---

### Servidor de Logs

**Responsabilidades:**
- Consumir logs de RabbitMQ
- Almacenar logs en memoria (con límite)
- Exponer REST API para consultas
- Filtrar logs por múltiples criterios

**Tecnologías:**
- **RabbitMQ** (consumidor): Recibe logs del servidor principal
- **REST API** (puerto 5000): Expone consultas

**Almacenamiento:**
- In-memory con límite configurable (default: 5000)
- FIFO: cuando se alcanza el límite, se eliminan los más antiguos
- Thread-safe con `ReaderWriterLockSlim`

**Filtros soportados:**
- Usuario, Clase, Acción, Nivel, Fechas, Texto libre
- Se pueden combinar múltiples filtros

---

## 🔄 Flujos de Comunicación

### Flujo 1: Login de Usuario
```
Cliente → [TCP] → Servidor: REQ|01|0005|user1
Servidor → [TCP] → Cliente: RES|01|0008|ASK_PASS
Cliente → [TCP] → Servidor: REQ|01|0008|password
Servidor → [TCP] → Cliente: RES|01|0002|OK
Servidor → [RabbitMQ] → Log: {"Action": "LOGIN", "User": "user1"}
RabbitMQ → ServidorLogs: (consume log)
```

### Flujo 2: Inscripción a Clase con Webhook
```
Cliente → [TCP] → Servidor: REQ|05|0000| (ENROLL)
Servidor → [TCP] → Cliente: RES|05|000A|ASK_CLASSID
Cliente → [TCP] → Servidor: REQ|05|0006|abc123
Servidor → [TCP] → Cliente: RES|05|0009|ASK_WEBHOOK
Cliente → [TCP] → Servidor: REQ|05|0030|https://webhook.site/xyz
Servidor → [TCP] → Cliente: RES|05|0002|OK
Servidor → [RabbitMQ] → Log: {"Action": "ENROLL", "ClassId": "abc123"}
```

**1 minuto antes de la clase:**
```
Servidor (WebhookNotifier) → [HTTP POST] → https://webhook.site/xyz
Payload: {"classId": "abc123", "className": "...", "message": "..."}
```

### Flujo 3: Chat en Clase Online
```
ClienteChat → [WebSocket] → ServidorChat: {"Type": "AUTH", "Username": "user1", "Password": "pass1"}
ServidorChat → [gRPC] → Servidor: ValidateUser("user1", "pass1")
Servidor → [gRPC] → ServidorChat: {Valid: true}
ServidorChat → [WebSocket] → ClienteChat: {"Type": "AUTH_RESPONSE", "Success": true}

ClienteChat → [WebSocket] → ServidorChat: {"Type": "JOIN", "ClassLink": "https://clases/abc123"}
ServidorChat → [gRPC] → Servidor: ValidateClassLink("abc123")
Servidor → [gRPC] → ServidorChat: {Valid: true, ClassName: "Matemáticas"}
ServidorChat → [WebSocket] → ClienteChat: {"Type": "JOINED", "ClassId": "abc123"}

ClienteChat → [WebSocket] → ServidorChat: {"Type": "MESSAGE", "Message": "Hola!"}
ServidorChat → [WebSocket] → Todos en la sala: {"Type": "MESSAGE", "Username": "user1", "Message": "Hola!"}
```

### Flujo 4: Consulta de Logs
```
Cliente HTTP → [REST GET] → ServidorLogs: /logs?user=usuario1&action=LOGIN&limit=10
ServidorLogs → Consulta en memoria → Filtra logs
ServidorLogs → [REST JSON] → Cliente HTTP: [{...}, {...}, ...]
```

---

## 💡 Justificación de Decisiones

### ¿Por qué TCP para Cliente-Servidor Principal?

**Razones:**
1. **Protocolo personalizado**: Necesitamos control total sobre el formato
2. **Eficiencia**: Operaciones síncronas request-response no necesitan HTTP
3. **Simplicidad**: Frame-based es más simple que REST para este caso
4. **Legacy**: Probablemente viene de obligatorios anteriores

**Alternativas consideradas:**
- ❌ REST API: Overhead innecesario, más complejo para este caso
- ❌ gRPC: Mejor para servicios, pero TCP es más directo para cliente-servidor
- ✅ TCP personalizado: Perfecto para este caso de uso

---

### ¿Por qué RabbitMQ para Logs?

**Razones:**
1. **Desacoplamiento**: El servidor principal no depende del servidor de logs
2. **Asincronía**: No bloquea operaciones principales
3. **Confiabilidad**: Mensajes no se pierden si el consumidor está offline
4. **Escalabilidad**: Fácil agregar más consumidores

**Alternativas consideradas:**
- ❌ TCP directo: Acopla los servicios, si el servidor de logs cae, el principal falla
- ❌ REST API: El servidor principal tendría que esperar respuesta
- ✅ RabbitMQ: Patrón estándar MOM, desacoplado y confiable

---

### ¿Por qué gRPC para ServidorChat-Servidor?

**Razones:**
1. **Performance**: Binario, HTTP/2, más rápido que REST
2. **Tipado fuerte**: Contratos explícitos en `.proto`
3. **Inter-servicio**: Comunicación entre servicios backend
4. **Streaming**: Futuro soporte para streaming si se necesita

**Alternativas consideradas:**
- ❌ REST API: Más lento, más overhead
- ❌ TCP directo: Tendríamos que implementar otro protocolo
- ✅ gRPC: Estándar moderno para comunicación entre servicios

---

### ¿Por qué WebSocket para Chat?

**Razones:**
1. **Tiempo real**: Comunicación bidireccional instantánea
2. **Push**: El servidor puede enviar mensajes sin que el cliente pregunte
3. **Eficiencia**: Una conexión para múltiples mensajes
4. **Estándar**: Soporte nativo en navegadores y .NET

**Alternativas consideradas:**
- ❌ Polling HTTP: Latencia alta, ineficiente
- ❌ Server-Sent Events (SSE): Solo unidireccional (servidor→cliente)
- ✅ WebSocket: Bidireccional, tiempo real, estándar

---

### ¿Por qué REST API para Servidor de Logs?

**Razones:**
1. **Accesibilidad**: Cualquier herramienta HTTP puede consumirlo
2. **Fácil de probar**: Postman, curl, navegador
3. **Filtrado flexible**: Query parameters estándar
4. **Estándar universal**: Todos conocen REST

**Alternativas consideradas:**
- ❌ gRPC: Requeriría cliente gRPC, menos accesible
- ❌ TCP: Tendríamos que implementar protocolo personalizado
- ✅ REST API: Estándar, accesible, flexible

---

## 🐳 Docker y Despliegue

### Arquitectura Docker

**docker-compose.yml** define todos los servicios:
- `rabbitmq`: Message broker (puerto 5672, 15672)
- `servidor`: Servidor principal (puerto 30000, 5001)
- `servidor-logs`: Servidor de logs (puerto 5000)
- `servidor-chat`: Servidor de chat (puerto 5002)
- `cliente`: Cliente (opcional, con profile)
- `cliente-chat`: Cliente de chat (opcional, con profile)

**Red Docker:**
- Todos los servicios en la red `clases_online_network`
- Se comunican por nombre de servicio (ej: `servidor`, `rabbitmq`)

### Multi-Stage Build

**Servidor Principal** usa multi-stage build:
- **Stage 1 (build)**: SDK de .NET para compilar
- **Stage 2 (runtime)**: Solo runtime de .NET

**Ventajas:**
- Imagen final **60-70% más pequeña** (~280MB vs ~850MB)
- Menor superficie de ataque (sin herramientas de desarrollo)
- Transferencias más rápidas
- Mejores prácticas de Docker

Ver `DOCKER-MULTISTAGE-JUSTIFICATION.md` para detalles.

---

## 📊 Resumen de Tecnologías

| Tecnología | Dónde se usa | Por qué | Puerto |
|------------|-------------|---------|--------|
| **TCP** | Cliente ↔ Servidor | Protocolo personalizado, eficiente | 30000 |
| **RabbitMQ** | Servidor → ServidorLogs | MOM, desacoplamiento, asincronía | 5672 |
| **gRPC** | ServidorChat ↔ Servidor | Inter-servicio, performance, tipado | 5001 |
| **WebSocket** | ClienteChat ↔ ServidorChat | Tiempo real, bidireccional | 5002 |
| **REST API** | Acceso remoto a logs | Estándar, accesible, flexible | 5000 |

---

## 🎯 Conclusión

El sistema utiliza **4 tecnologías obligatorias** (gRPC, RabbitMQ, REST API, WebSocket) distribuidas estratégicamente según las necesidades de cada componente:

- **TCP personalizado**: Para el cliente principal (legacy, eficiente)
- **RabbitMQ**: Para logs (desacoplamiento, asincronía)
- **gRPC**: Para comunicación entre servicios (performance, tipado)
- **WebSocket**: Para chat en tiempo real (bidireccional, push)
- **REST API**: Para acceso remoto a logs (estándar, accesible)

Cada tecnología fue elegida porque es la **mejor opción** para su caso de uso específico, siguiendo principios de arquitectura de software modernos.

