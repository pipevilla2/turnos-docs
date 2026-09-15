# Sistema de Agendamiento de Turnos Bancarios

Prueba técnica: sistema de agendamiento de turnos bancarios compuesto por un
backend en **ASP.NET Core 8 Web API** (`turnos-backend`) y un frontend en
**Angular 18** (`turnos-frontend`), documentados en detalle en
[`INFORME_ARQUITECTURA.md`](./INFORME_ARQUITECTURA.md). Para ver las
gráficas de la arquitectura, revisa
[`INFORME_ARQUITECTURA.pdf`](./INFORME_ARQUITECTURA.pdf), que sí las incluye
y debe leerse junto con el `.md`.
[`MANUAL_USUARIO.md`](./MANUAL_USUARIO.md): instrucciones básicas para
   ingresar, agendar, activar y consultar turnos.
## Repositorios / carpetas del proyecto

| Carpeta | Contenido | Repositorio en GitHub |
|---|---|---|
| `turnos-backend` | API REST en .NET 8 (Clean Architecture) | https://github.com/pipevilla2/turnos-backend |
| `turnos-frontend` | SPA en Angular 18 | https://github.com/pipevilla2/turnos-frontend |
| `turnos-docs` | Documentación del proyecto (este archivo, informe de arquitectura, etc.) | https://github.com/pipevilla2/turnos-docs |

## Demo desplegada en Azure

✅ **La demo desplegada en Azure está completamente funcionando** y se puede
usar directamente, sin necesidad de instalar nada localmente:

- **Frontend (Azure Static Web App)**: https://gray-field-03d5bbe0f.3.azurestaticapps.net/login
- **Backend (Azure App Service - Swagger)**: https://turnos-backend-hpgygthvgpcdcbbf.westus3-01.azurewebsites.net/swagger/index.html

## Requisitos previos

- [.NET SDK 8.0](https://dotnet.microsoft.com/download) o superior
- [Node.js 18+](https://nodejs.org/) y npm
- [Angular CLI 18](https://angular.dev/tools/cli) (`npm install -g @angular/cli`)
- Acceso a la base de datos de pruebas: servidor **Azure SQL** `myservidorpruebas`,
  base de datos `dbturnos` (la contraseña se proporciona aparte)

## 1. Backend (`turnos-backend`)

### 1.1 Configuración previa

1. El archivo `src/Turnos.Api/appsettings.json` ya trae configurada la
   cadena de conexión (`ConnectionStrings:Default`) al servidor Azure SQL de
   pruebas `myservidorpruebas`, base de datos `dbturnos`. No es necesario
   cambiar nada: el servidor es solo de pruebas y ya está listo para usarse.

2. La base de datos de pruebas (`dbturnos`) ya está creada en Azure SQL con
   sus tablas y datos por defecto. El script de creación se encuentra en
   [`scriptBASEDATOS.sql`](./scriptBASEDATOS.sql), dentro de `turnos-docs`,
   por si necesitas recrearla en otro servidor.

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

## 2. Frontend (`turnos-frontend`)

### 2.1 Configuración previa

1. Revisar `src/environments/environment.ts` (desarrollo) y
   `environment.prod.ts` (producción) y ajustar `apiUrl` para que apunte a la
   URL donde esté corriendo el backend (por defecto
   `https://localhost:7150/api` en desarrollo).

   > **Importante**: para que el frontend corra localmente y apunte al
   > backend local (en vez del backend desplegado en Azure), debes cambiar
   > `apiUrl` en `environment.ts` a la URL local del backend
   > (`https://localhost:7150/api`).

### 2.2 Instalar dependencias

```bash
cd turnos-frontend
npm install
```

### 2.3 Ejecutar en modo desarrollo

```bash
npm start
ng serve -o
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

1. Ejecutar el backend (`dotnet run`) y confirmar que responde en Swagger.
2. Actualizar `environment.ts` del frontend con la URL del backend si difiere
   de la URL por defecto.
3. Ejecutar el frontend (`npm start`) (`ng serve -o`) y acceder a `http://localhost:4200`.

## 4. Documentación adicional

- [`MANUAL_USUARIO.md`](./MANUAL_USUARIO.md): instrucciones básicas para
   ingresar, agendar, activar y consultar turnos.
- [`INFORME_ARQUITECTURA.md`](./INFORME_ARQUITECTURA.md): arquitectura,
  patrones de diseño, modelo de datos, seguridad, despliegue en Azure y
  pruebas unitarias.
- [`STACK_TECNOLOGICO.md`](./STACK_TECNOLOGICO.md): detalle de tecnologías y
  versiones usadas en backend y frontend.
- [`PRUEBA_TECNICA_AMARIS.md`](./PRUEBA_TECNICA_AMARIS.md): enunciado
  original de la prueba técnica.
