# Informe de Arquitectura — Sistema de Agendamiento de Turnos (Amaris)

## 1. Objetivo

Permitir que los clientes de la entidad bancaria agenden turnos de atención en
cualquier sucursal desde una app móvil o la web, usando su número de cédula,
con un límite de 15 minutos para activar el turno al llegar físicamente, un
máximo de 5 turnos solicitados por día por cédula, y expiración automática de
los turnos no activados.

## 2. Visión general de la arquitectura

La solución se divide en dos proyectos independientes que se comunican por
HTTP/REST:

```
Angular SPA (cliente/empleado)  --HTTPS/JSON+JWT-->  ASP.NET Core Web API  -->  SQLite
```

### 2.1 Backend — Clean Architecture / capas por responsabilidad

```
Turnos.Api            (presentación: controllers, auth, middleware, DI)
      -> Turnos.Application  (casos de uso / reglas de orquestación, DTOs)
             -> Turnos.Domain      (entidades, reglas de negocio puras, contratos)
      -> Turnos.Infrastructure (EF Core, repositorios, background service, seed)
Turnos.Tests           (pruebas unitarias xUnit + Moq + FluentAssertions)
```

La dependencia siempre apunta hacia el **Dominio**: `Api` e `Infrastructure`
dependen de `Application`, que a su vez depende de `Domain`; `Domain` no
depende de nada. Esto permite:

- Sustituir SQLite por SQL Server/PostgreSQL sin tocar reglas de negocio.
- Probar la lógica de agendamiento sin base de datos real (mocks de
  `ITurnoRepository` / `ISucursalRepository`).

### 2.2 Patrones de diseño aplicados

| Patrón | Dónde | Por qué |
|---|---|---|
| **Repository** | `ITurnoRepository`, `ISucursalRepository` + implementaciones EF Core | Aísla el acceso a datos del resto de la app; facilita pruebas con dobles. |
| **Service Layer / Application Service** | `TurnoService`, `SucursalService` | Concentra los casos de uso (crear, activar, listar, actualizar) y coordina repositorios + reglas de dominio. |
| **Rich Domain Model** | Entidad `Turno` | Las transiciones de estado (`Activar`, `Cancelar`, `MarcarAtendido`, `IntentarExpirar`) viven en la entidad, no en el service, evitando un "modelo anémico" y garantizando invariantes (p. ej. no se puede activar un turno ya atendido). |
| **Dependency Injection** | `Program.cs` (contenedor nativo de .NET) | Bajo acoplamiento entre capas; permite reemplazar implementaciones en pruebas. |
| **Middleware / Pipeline** | `ExceptionMiddleware` | Traduce excepciones de dominio a respuestas HTTP consistentes sin repetir try/catch en cada controller. |
| **Background Worker (Hosted Service)** | `ExpiracionTurnosService` | Expira turnos vencidos de forma proactiva (cada 30s), garantizando la regla de los 15 minutos incluso si nadie vuelve a consultar el turno. |
| **DTO / Anti-Corruption Layer** | `Turnos.Application.DTOs` | La API nunca expone las entidades de dominio directamente; controla qué información sale y desacopla el contrato REST de la persistencia. |

## 3. Modelo de datos

### Motor de base de datos

La solución utiliza **SQLite** por practicidad: es una base de datos embebida,
no requiere instalar ni administrar un servidor externo y permite ejecutar la
prueba técnica rápidamente con una configuración mínima. No obstante, la
aplicación no queda acoplada a SQLite, porque el acceso a datos está aislado en
`Turnos.Infrastructure` mediante Entity Framework Core.

Para cambiar a **SQL Server** bastan unos pocos ajustes: instalar el paquete
`Microsoft.EntityFrameworkCore.SqlServer`, reemplazar `UseSqlite(...)` por
`UseSqlServer(...)` y actualizar la cadena de conexión. Las entidades,
repositorios, servicios y reglas de negocio permanecen iguales.

**Sucursal**: Id, Nombre, Dirección, Ciudad, Activa.

**Turno**: Id (GUID), CodigoTurno, Cédula, SucursalId (FK), FechaHoraCreación,
FechaHoraExpiración (Creación + 15 min), FechaHoraActivación (nullable),
Estado (`Pendiente`, `Activado`, `Atendido`, `Expirado`, `Cancelado`).

Se agregaron índices compuestos `(Cedula, FechaHoraCreacion)` y
`(Estado, FechaHoraExpiracion)` porque son exactamente los filtros usados por
las dos consultas más frecuentes: contar turnos del día por cédula y barrer
turnos pendientes vencidos.

## 4. Reglas de negocio y dónde se implementan

| Regla | Implementación |
|---|---|
| Límite de 15 minutos para activar | `Turno.Crear` fija `FechaHoraExpiracion = ahora + 15min`; `Turno.Activar` valida el tiempo y lanza `TurnoExpiradoException` si ya venció. |
| Expiración automática | `ExpiracionTurnosService` (BackgroundService) + `Turno.IntentarExpirar`, corre cada 30s. |
| Máximo 5 turnos/día por cédula | `TurnoService.CrearTurnoAsync` consulta `CountByCedulaOnDateAsync` (turnos no cancelados del día) antes de crear; si es ≥5 lanza `LimiteTurnosDiariosException` (HTTP 409). El contador se reinicia naturalmente al cambiar de día porque el filtro usa el rango `[hoy 00:00, mañana 00:00)`. |
| Reintentar tras expirar | El cliente simplemente vuelve a llamar `POST /api/turnos`; como el turno expirado no cuenta distinto de uno vigente, solo se bloquea si ya llegó a 5 turnos "vivos" (no cancelados) ese día — incluyendo expirados, que sí cuentan como intento, tal como lo especifica el enunciado ("más de 5 turnos solicitados en el día"). |
| Solo sucursales activas | Se valida `Sucursal.Activa` en `CrearTurnoAsync`. |

## 5. Seguridad

- **Autenticación JWT** (`Microsoft.AspNetCore.Authentication.JwtBearer`), con
  dos flujos:
  - `POST /api/auth/token-cliente { cedula }` → token con rol `Cliente` y el
    claim `cedula`. Representa que el cliente ya pasó la autenticación fuerte
    del banco (PIN/biometría/OTP) en su app y solo necesita una sesión para
    consumir la API de turnos.
  - `POST /api/auth/login { usuario, password }` → token con rol `Empleado`,
    para el personal de sucursal (credenciales de demostración en
    `appsettings.json`; en producción irían contra el directorio corporativo).
- **Autorización por rol y por dueño del recurso**: todos los endpoints de
  `TurnosController` exigen `[Authorize]`; un cliente solo puede crear/ver/
  activar **sus propios** turnos (se compara el claim `cedula` del token con
  la cédula del recurso); `PUT /api/turnos/{id}` (marcar Atendido/Cancelado)
  está restringido a `[Authorize(Roles = "Empleado")]`.
- **CORS** restringido a los orígenes configurados (`http://localhost:4200`
  en desarrollo).
- La llave de firma JWT, credenciales y cadenas de conexión están en
  `appsettings.json` solo como ejemplo de prueba técnica; en un entorno real
  irían en un vault / variables de entorno / `dotnet user-secrets`.

## 6. Eficiencia y escalabilidad

- **Stateless API + JWT**: no hay sesión en memoria del servidor, por lo que
  la API puede escalar horizontalmente detrás de un balanceador sin
  "sticky sessions".
- **Concurrencia en la base de datos**: SQLite se usa aquí por simplicidad de
  la prueba técnica (cero configuración); el código de acceso a datos está
  aislado en `Turnos.Infrastructure` para migrar a SQL Server/PostgreSQL con
  soporte real de escrituras concurrentes solo cambiando el `UseSqlite` por
  `UseSqlServer`/`UseNpgsql` y la cadena de conexión — el resto de capas no
  cambia.
- **Índices** en las columnas más consultadas (ver sección 3) para mantener
  O(log n) las validaciones de límite diario y el barrido de expiración
  incluso con el crecimiento de la tabla `Turnos`.
- El **worker de expiración** corre en un scope propio y en lote (batch),
  evitando bloquear el hilo de peticiones HTTP.
- La creación de turnos es idempotente a nivel de negocio (cada llamada
  produce un turno nuevo con GUID), lo que facilita reintentos seguros desde
  el cliente ante fallos de red.

## 7. Front-end (Angular)

- **Standalone components** (Angular 17+), sin `NgModules`, con *lazy
  loading* por ruta (`loadComponent`) para minimizar el bundle inicial.
- **Capa `core`**: `AuthService` (sesión con signals), `TurnoService`,
  `SucursalService` (HTTP), `authInterceptor` (adjunta el JWT a cada
  petición) y `authGuard` (protege rutas autenticadas).
- **Capa `features`**: un componente por pantalla (`login`,
  `agendar-turno`, `lista-turnos`, `turno-detalle`), cada uno con su HTML
  separado, siguiendo separación de responsabilidades.
- El componente de agendamiento muestra un **cronómetro en vivo** de los 15
  minutos y deshabilita "Activar turno" cuando llega a cero, reflejando en
  UI la misma regla de negocio que protege el backend.

## 8. Pruebas

- **Backend** (`Turnos.Tests`, xUnit + Moq + FluentAssertions):
  reglas de creación (límite diario, sucursal inactiva/inexistente),
  activación dentro/fuera de tiempo, transición de estados inválida, y
  reglas puras de la entidad `Turno`.
- **Frontend** (Jasmine/Karma, generado por Angular CLI):
  `AuthService`, `TurnoService` (peticiones HTTP simuladas con
  `HttpTestingController`) y el componente `AgendarTurnoComponent`.

## 9. Supuestos y alcance

- No se implementó registro/gestión de usuarios administradores; el login de
  empleado usa credenciales de demostración en configuración, suficiente para
  demostrar autorización basada en roles en el alcance de la prueba.
- Se usa SQLite embebido para que el evaluador pueda ejecutar el proyecto sin
  instalar un motor de base de datos aparte; la migración a un motor
  productivo es un cambio de una línea (ver sección 6).
- El conteo de "turnos solicitados en el día" excluye los `Cancelado`
  (el usuario se retractó explícitamente) pero incluye `Expirado` (sí fue una
  solicitud real que no se materializó), conforme al enunciado.
