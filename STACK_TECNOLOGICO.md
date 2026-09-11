# Stack tecnológico

## Resumen

La solución está compuesta por un backend desarrollado con ASP.NET Core Web API y un frontend desarrollado con Angular. Ambos proyectos se comunican mediante una API REST utilizando JSON y autenticación JWT.

## Backend

| Tecnología | Versión / detalle |
|---|---|
| Lenguaje | C# |
| Framework | .NET 8 / ASP.NET Core 8 |
| Target framework | `net8.0` |
| Tipo de aplicación | ASP.NET Core Web API RESTful |
| Arquitectura | Clean Architecture por capas |
| Autenticación | JWT Bearer |
| Acceso a datos | Entity Framework Core 8 |
| Base de datos | SQLite (seleccionada por practicidad y facilidad de ejecución) |
| Documentación API | Swagger / OpenAPI mediante Swashbuckle |
| Procesos en segundo plano | `BackgroundService` / Hosted Service |
| Inyección de dependencias | Contenedor nativo de ASP.NET Core |

### Proyectos del backend

- `Turnos.Api`: controllers, autenticación, middleware y configuración de la aplicación.
- `Turnos.Application`: casos de uso, servicios, DTOs e interfaces de aplicación.
- `Turnos.Domain`: entidades, enumeraciones, excepciones y reglas de negocio.
- `Turnos.Infrastructure`: Entity Framework Core, contexto de datos, repositorios, seed y procesos en segundo plano.
- `Turnos.Tests`: pruebas unitarias del dominio y la aplicación.

### Paquetes principales del backend

- `Microsoft.AspNetCore.Authentication.JwtBearer` `8.0.8`
- `Microsoft.EntityFrameworkCore.Sqlite` `8.x`
- `Microsoft.EntityFrameworkCore.Design` `8.x`
- `Swashbuckle.AspNetCore` `6.6.2`

### Base de datos y portabilidad

Se utiliza **SQLite** porque es una base de datos embebida, no requiere instalar
ni configurar un servidor externo y permite ejecutar la prueba técnica de forma
rápida y sencilla. La aplicación conserva el acceso a datos aislado en
`Turnos.Infrastructure` mediante Entity Framework Core.

Para trabajar con **SQL Server** en un entorno productivo, el cambio principal
consiste en instalar el proveedor `Microsoft.EntityFrameworkCore.SqlServer`,
cambiar `UseSqlite(...)` por `UseSqlServer(...)` en la configuración del
contexto y actualizar la cadena de conexión. Las entidades, servicios,
repositorios y reglas de negocio no necesitan cambios.

### Pruebas del backend

- **xUnit** `2.9.0`: framework de pruebas unitarias.
- **Moq** `4.20.70`: creación de mocks para repositorios e interfaces.
- **FluentAssertions** `6.12.0`: aserciones legibles.
- **Microsoft.NET.Test.Sdk** `17.11.1`: ejecución de pruebas .NET.
- **Entity Framework Core InMemory** `8.0.8`: soporte para pruebas con base de datos en memoria.

## Frontend

| Tecnología | Versión / detalle |
|---|---|
| Framework | Angular `18.2.x` |
| Lenguaje | TypeScript `5.5.x` |
| Plataforma | Angular CLI `18.2.x` |
| Runtime | Node.js / npm |
| Cliente HTTP | `HttpClient` de Angular |
| Reactividad | RxJS `7.8.x` |
| Detección de cambios | Zone.js `0.14.x` |
| Arquitectura | Standalone Components |
| Navegación | Angular Router |
| Formularios | Angular Forms |
| Estilos | CSS |
| Build | Angular CLI y `@angular-devkit/build-angular` |

### Pruebas del frontend

- **Jasmine** `5.2.x`: framework de pruebas.
- **Karma** `6.4.x`: test runner.
- **ChromeHeadless**: navegador utilizado en ejecución sin interfaz gráfica.
- **Angular TestBed**: pruebas de componentes y servicios.
- **HttpTestingController**: simulación y verificación de solicitudes HTTP.
- **karma-coverage** `2.2.x`: generación de cobertura.
- **karma-jasmine-html-reporter** `2.1.x`: reporte HTML de resultados.

Jest no está implementado actualmente en el frontend.

## Comunicación entre aplicaciones

```text
Angular 18 SPA -- HTTP/JSON + JWT --> ASP.NET Core 8 Web API -- EF Core --> SQLite
```

SQLite es la opción actual para facilitar la ejecución local. El mismo backend
puede utilizar SQL Server cambiando el proveedor de EF Core y la cadena de
conexión en unas pocas líneas, sin modificar la lógica de negocio.

## Comandos principales

### Backend

```bash
dotnet restore TurnosBackend.sln
dotnet build TurnosBackend.sln
dotnet test TurnosBackend.sln
dotnet run --project src/Turnos.Api/Turnos.Api.csproj
```

### Frontend

```bash
npm install
npm run build
npm start
npm test
npm run test:ci
```
