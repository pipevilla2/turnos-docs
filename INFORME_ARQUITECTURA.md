# Informe de Arquitectura — Sistema de Agendamiento de Turnos

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
Angular SPA (cliente/empleado)  --HTTPS/JSON+JWT-->  ASP.NET Core Web API  -->  SQL Server (Azure SQL)
```

### 2.1 Backend — Clean Architecture / capas por responsabilidad

```
Turnos.Api            (presentación: controllers, auth, middleware, DI)
      -> Turnos.Application  (casos de uso / reglas de orquestación, DTOs)
             -> Turnos.Domain      (entidades, reglas de negocio puras, contratos)
      -> Turnos.Infrastructure (EF Core, repositorios, background service, seed)
Turnos.Tests           (pruebas unitarias xUnit + Moq + FluentAssertions)
```

![Clean Architecture - Turnos](file:///C:/Users/PC%20HP/Downloads/diagrama%20arquitectura%20clean.png)

La dependencia siempre apunta hacia el **Dominio**: `Api` e `Infrastructure`
dependen de `Application`, que a su vez depende de `Domain`; `Domain` no
depende de nada. Esto permite:

- Sustituir el motor de base de datos (SQL Server, PostgreSQL, etc.) sin tocar
  reglas de negocio.
- Probar la lógica de agendamiento sin base de datos real (mocks de
  `ITurnoRepository` / `ISucursalRepository`).

### 2.2 Patrones de diseño aplicados

| Patrón | Dónde | Por qué |
|---|---|---|
| **Repository** | `ITurnoRepository`, `ISucursalRepository` + implementaciones EF Core | Aísla el acceso a datos del resto de la app; facilita pruebas con dobles. |
| **Service Layer / Application Service** | `TurnoService`, `SucursalService` | Concentra los casos de uso (crear, activar, listar, actualizar) y coordina repositorios + reglas de dominio. |
| **Rich Domain Model** | Entidad `Turno` | Las transiciones de estado (`Activar`, `Cancelar`, `MarcarAtendido`, `IntentarExpirar`) viven en la entidad, no en el service, evitando un modelo con solo datos y sin lógica y garantizando invariantes (p. ej. no se puede activar un turno ya atendido). |
| **Dependency Injection** | `Program.cs` (contenedor nativo de .NET) | Bajo acoplamiento entre capas; permite reemplazar implementaciones en pruebas. |
| **Middleware / Pipeline** | `ExceptionMiddleware` | Traduce excepciones de dominio a respuestas HTTP consistentes sin repetir try/catch en cada controller. |
| **Background Worker (Hosted Service)** | `ExpiracionTurnosService` | Expira turnos vencidos de forma proactiva (cada 30s), garantizando la regla de los 15 minutos incluso si nadie vuelve a consultar el turno. |
| **DTO / Anti-Corruption Layer** | `Turnos.Application.DTOs` | La API nunca expone las entidades de dominio directamente; controla qué información sale y desacopla el contrato REST de la persistencia. |

### 2.3 Frontend — Angular por funcionalidades

El frontend utiliza una arquitectura **feature-based** con **Standalone
Components**, sin `NgModules`. Las funcionalidades se agrupan por pantalla y
las responsabilidades compartidas se concentran en `core`:

```text
src/app/
├── core/
│   ├── guards/          (protección de rutas)
│   ├── interceptors/    (JWT en peticiones HTTP)
│   ├── models/          (modelos TypeScript)
│   └── services/        (AuthService, TurnoService, SucursalService)
└── features/
    ├── login/
    ├── agendar-turno/
    ├── lista-turnos/
    └── turno-detalle/
```

Cada funcionalidad representa una pantalla y mantiene su lógica TypeScript y
su plantilla HTML separadas. Las rutas utilizan `loadComponent` para cargar
los componentes bajo demanda, reduciendo el peso inicial de la aplicación al
abrirse.

#### Convenciones de nombres del frontend

- Archivos y carpetas en **kebab-case**: `agendar-turno` y
  `agendar-turno.component.ts`.
- Clases en **PascalCase**: `AgendarTurnoComponent` y `TurnoService`.
- Selectores de componentes con el prefijo `app-`: `app-agendar-turno`.
- Servicios con el sufijo `.service.ts`, guards con `.guard.ts` e
  interceptores con `.interceptor.ts`.
- Las pruebas se nombran con el sufijo `.spec.ts` y se ubican junto al código
  que validan.

Ejemplo de carga diferida:

```typescript
{
  path: 'agendar',
  canActivate: [authGuard],
  loadComponent: () =>
    import('./features/agendar-turno/agendar-turno.component')
      .then(m => m.AgendarTurnoComponent)
}
```

## 3. Modelo de datos

### Motor de base de datos

La solución utiliza **SQL Server** (desplegado como **Azure SQL Database**) a
través de Entity Framework Core (`Microsoft.EntityFrameworkCore.SqlServer`),
configurado con `UseSqlServer(...)` en `Program.cs`. SQL Server ofrece soporte
real de escrituras concurrentes, bloqueos a nivel de fila y transacciones, lo
que resulta clave para generar los códigos de turno de forma segura bajo
concurrencia (ver sección 6).

El acceso a datos está aislado en `Turnos.Infrastructure`, por lo que la
aplicación no queda acoplada a un motor concreto: migrar a otro proveedor
(p. ej. PostgreSQL con `UseNpgsql(...)`) implicaría cambiar solo la
configuración del `DbContext` y la cadena de conexión; las entidades,
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
| Máximo 5 turnos/día por cédula | `TurnoService.CrearTurnoAsync` consulta `CountByCedulaBetweenAsync` (turnos no cancelados del día colombiano) antes de crear; si es ≥5 lanza `LimiteTurnosDiariosException` (HTTP 409). El contador se reinicia naturalmente al cambiar de día porque el filtro usa el rango `[hoy 00:00, mañana 00:00)` de Colombia. |
| Fecha y hora de negocio | `ColombiaClock` genera la hora local de Colombia para creación, expiración y activación. El backend compara todas las fechas de turnos con el mismo reloj para evitar diferencias de zona horaria. |
| Código consecutivo del turno | `TurnoService` obtiene el siguiente consecutivo por sucursal mediante `GetNextConsecutivoAsync`, que incrementa de forma atómica un contador dedicado (`TurnosConsecutivos`) con `MERGE ... WITH (HOLDLOCK)`, garantizando códigos únicos incluso ante solicitudes concurrentes. |
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

- **API sin estado + JWT**: el servidor no guarda sesión en memoria; cada petición
  llega con su token y puede ser atendida por cualquier instancia del backend.
  Esto permite subir varias copias de la API detrás de un balanceador y enviar
  tráfico entre ellas sin que el usuario tenga que ir siempre a la misma máquina.
- **Concurrencia en la base de datos**: se usa **SQL Server** (Azure SQL), con
  soporte real de escrituras concurrentes. El consecutivo del código de turno
  se genera mediante una operación atómica (`MERGE ... WITH (HOLDLOCK)`) dentro
  de una transacción, que serializa el incremento por sucursal y evita códigos
  de turno duplicados bajo carga concurrente; el índice único
  `UX_Turnos_CodigoTurno` actúa como garantía final (ver
  `CONCURRENCIA_Y_ESCALABILIDAD.md`).
- **Índices** en las columnas más consultadas (ver sección 3) para mantener
  O(log n) las validaciones de límite diario y el barrido de expiración
  incluso con el crecimiento de la tabla `Turnos`.
- El **worker de expiración** corre en un scope propio y en lote (batch),
  evitando bloquear el hilo de peticiones HTTP.
- Cada creación de turno es independiente (cada llamada produce un turno nuevo
  con su propio identificador único GUID), lo que facilita reintentos seguros
  desde el cliente ante fallos de red.

## 7. Front-end (Angular)

- **Standalone components** (Angular 17+), sin `NgModules`, con *lazy
  loading* por ruta (`loadComponent`) para reducir el peso inicial al cargar la
  aplicación.
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
- Se usa **SQL Server** (Azure SQL Database) como motor de persistencia; el
  acceso a datos aislado en `Turnos.Infrastructure` permite migrar a otro
  proveedor cambiando solo la configuración del `DbContext` (ver sección 6).
- El conteo de "turnos solicitados en el día" excluye los `Cancelado`
  (el usuario se retractó explícitamente) pero incluye `Expirado` (sí fue una
  solicitud real que no se materializó), conforme al enunciado.

## 10. Despliegue en Azure y arquitectura de la solución

La prueba técnica fue desplegada en Azure para validar el flujo real de una
aplicación cliente-servidor con frontend estático, API en un plan de App Service
y base de datos SQL Server gestionada.

### 10.1 URLs desplegadas

- **Azure Static Web App (frontend)**: https://gray-field-03d5bbe0f.3.azurestaticapps.net/login
- **Azure App Service (API - Swagger)**: https://turnos-backend-hpgygthvgpcdcbbf.westus3-01.azurewebsites.net/swagger/index.html

### 10.2 Arquitectura desplegada

La solución se compone de estos componentes:

- **Azure Static Web Apps**: aloja el frontend Angular (`turnos-frontend`) y
  sirve la aplicación web de forma estática.
- **Azure App Service**: hospeda la API REST (`turnos-backend`), que expone los
  endpoints de autenticación, gestión de turnos y validaciones de negocio.
- **Azure SQL Server / Azure SQL Database**: almacena sucursales, turnos,
  consecutivos y toda la información transaccional requerida por la aplicación.

La comunicación sigue este flujo:

1. El usuario accede a la aplicación web desde Internet.
2. El frontend consume la API a través de HTTPS con JWT.
3. La API valida el token, aplica reglas de negocio y consulta la base de
   datos.
4. Azure SQL guarda y devuelve la información persistente de turnos y sucursales.

### 10.3 Descripción de la imagen

La imagen representa la infraestructura publicada en Azure y la relación entre
los componentes principales:

- **Internet / Usuario**: es el punto de entrada desde donde el cliente accede a
  la aplicación.
- **Azure Static Web Apps**: contiene la interfaz web del sistema; es el sitio
  público que entrega la experiencia frontend sin necesidad de un servidor de
  aplicación para renderizar páginas dinámicas.
- **Azure App Service**: alberga el backend `.NET`, con la API REST y la lógica
  de negocio. Se conecta con la base de datos y aplica la autenticación y la
  autorización del sistema.
- **Azure SQL Server**: gestiona la persistencia de datos. El contenedor de la
  base de datos `dbturnos` guarda la información de los turnos y las sucursales.
- **Plan de App Service**: indica que la API está desplegada en un entorno de
  hosting administrado por Azure, pudiendo escalar y mantener la aplicación sin
  gestionar infraestructura a nivel de sistema operativo.

### 10.4 Beneficios de este despliegue

- **Escalabilidad**: el frontend y la API pueden escalar independientemente
  según la carga.
- **Mantenimiento simplificado**: Azure se encarga del hosting y de la
  infraestructura base.
- **Seguridad**: la API se expone como servicio y la autenticación se maneja con
  JWT, con CORS y reglas de autorización para proteger los recursos.
- **Disponibilidad**: la separación de responsabilidades entre frontend, API y
  base de datos facilita despliegues y mantenimiento más seguros y predecibles.
