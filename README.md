# Sistema de Agendamiento de Turnos — Amaris

Prueba técnica: sistema de agendamiento de turnos bancarios compuesto por un
backend en **ASP.NET Core 8 Web API** (`turnos-backend`) y un frontend en
**Angular 18** (`turnos-frontend`), documentados en detalle en
[`INFORME_ARQUITECTURA.md`](./INFORME_ARQUITECTURA.md).

## Repositorios / carpetas del proyecto

| Carpeta | Contenido |
|---|---|
| `turnos-backend` | API REST en .NET 8 (Clean Architecture) |
| `turnos-frontend` | SPA en Angular 18 |
| `turnos-docs` | Documentación del proyecto (este archivo, informe de arquitectura, etc.) |

## Demo desplegada en Azure

- **Frontend (Azure Static Web App)**: https://gray-field-03d5bbe0f.3.azurestaticapps.net/login
- **Backend (Azure App Service - Swagger)**: https://turnos-backend-hpgygthvgpcdcbbf.westus3-01.azurewebsites.net/swagger/index.html

## Requisitos previos

- [.NET SDK 8.0](https://dotnet.microsoft.com/download) o superior
- [Node.js 18+](https://nodejs.org/) y npm
- [Angular CLI 18](https://angular.dev/tools/cli) (`npm install -g @angular/cli`)
- Acceso a una base de datos **SQL Server** (local, Docker o Azure SQL)

## 1. Backend (`turnos-backend`)

### 1.1 Configuración previa

1. Ubicar el archivo `src/Turnos.Api/appsettings.json` (o
   `appsettings.Development.json` para desarrollo local) y actualizar:
   - `ConnectionStrings:Default`: cadena de conexión a tu instancia de SQL
     Server (local, Docker o Azure SQL).
   - `Jwt:Key`: llave secreta para firmar los tokens (mínimo 32 caracteres).
   - `Cors:OrigenesPermitidos`: agregar la URL desde la que se ejecutará el
     frontend (por ejemplo `http://localhost:4200`).

   > **Nota de seguridad**: no dejes credenciales reales de base de datos ni
   > llaves JWT en `appsettings.json` dentro del repositorio. Para uso local
   > usa `dotnet user-secrets`, y en producción usa variables de entorno o un
   > vault (Azure Key Vault, etc.).

2. La base de datos y las tablas se crean automáticamente al iniciar la API
   (`DbSeeder.Seed` ejecuta `Database.EnsureCreated()` y siembra las
   sucursales de ejemplo), por lo que **no es necesario ejecutar migraciones
   manualmente**; solo se requiere que la cadena de conexión apunte a una
   base de datos SQL Server accesible.

### 1.2 Restaurar, compilar y probar

```bash
cd turnos-backend
dotnet restore TurnosBackend.sln
dotnet build TurnosBackend.sln
dotnet test TurnosBackend.sln
```

### 1.3 Ejecutar la API

```bash
dotnet run --project src/Turnos.Api/Turnos.Api.csproj
```

La API queda disponible en la URL indicada en consola (por defecto algo como
`https://localhost:7150`) y Swagger en `/swagger/index.html`.

### 1.4 Usuarios de prueba

- **Cliente**: `POST /api/auth/token-cliente` con `{ "cedula": "..." }`.
- **Empleado**: `POST /api/auth/login` con las credenciales de demostración
  configuradas en `EmpleadoDemo` (`appsettings.json`).

## 2. Frontend (`turnos-frontend`)

### 2.1 Configuración previa

1. Revisar `src/environments/environment.ts` (desarrollo) y
   `environment.prod.ts` (producción) y ajustar `apiUrl` para que apunte a la
   URL donde esté corriendo el backend (por defecto
   `https://localhost:7150/api` en desarrollo).

### 2.2 Instalar dependencias

```bash
cd turnos-frontend
npm install
```

### 2.3 Ejecutar en modo desarrollo

```bash
npm start
```

La aplicación queda disponible en `http://localhost:4200`.

### 2.4 Ejecutar pruebas unitarias

```bash
npm test          # modo interactivo (watch)
npm run test:ci   # una sola corrida en ChromeHeadless
```

### 2.5 Compilar para producción

```bash
npm run build
```

El resultado queda en `dist/amaris-turnos-frontend/browser`, listo para
desplegar en Azure Static Web Apps u otro hosting estático.

## 3. Orden recomendado de ejecución local

1. Levantar SQL Server (local, contenedor Docker o Azure SQL) y actualizar la
   cadena de conexión del backend.
2. Ejecutar el backend (`dotnet run`) y confirmar que responde en Swagger.
3. Actualizar `environment.ts` del frontend con la URL del backend si difiere
   de la URL por defecto.
4. Ejecutar el frontend (`npm start`) y acceder a `http://localhost:4200`.

## 4. Documentación adicional

- [`INFORME_ARQUITECTURA.md`](./INFORME_ARQUITECTURA.md): arquitectura,
  patrones de diseño, modelo de datos, seguridad, despliegue en Azure y
  pruebas unitarias.
- [`STACK_TECNOLOGICO.md`](./STACK_TECNOLOGICO.md): detalle de tecnologías y
  versiones usadas en backend y frontend.
- [`CONCURRENCIA_Y_ESCALABILIDAD.md`](./CONCURRENCIA_Y_ESCALABILIDAD.md):
  análisis del manejo de concurrencia en la generación de turnos.
- [`PRUEBA_TECNICA_AMARIS.md`](./PRUEBA_TECNICA_AMARIS.md): enunciado
  original de la prueba técnica.
