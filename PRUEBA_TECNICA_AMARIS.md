# Prueba técnica Amaris

## Agendamiento de turnos

Los clientes de la entidad bancaria deben poder agendar turnos con anticipación para ser atendidos en cualquiera de las sucursales disponibles.

### Agendamiento móvil o web

El agendamiento de turnos debe ser posible a través de una aplicación móvil o de la página web. Esto permite a los usuarios reservar turnos sin estar físicamente en la sucursal, utilizando su número de cédula.

### Tiempo límite

Cuando un usuario agenda un turno, se le otorga un límite de tiempo de 15 minutos para llegar a la sucursal correspondiente. Durante este período, el usuario puede activar el turno directamente en la sucursal.

Si el usuario no llega a la sucursal dentro de los 15 minutos, el turno debe expirar. Si el usuario lo requiere, debe poder generar otro turno.

Los turnos deben almacenarse en una base de datos. También se debe validar si la cédula tiene más de 5 turnos solicitados en el día; en ese caso, no se le permitirá generar más turnos hasta el día siguiente.

## 1. Requisitos

- Desarrollar una solución RESTful en C# .NET 6. También son válidas .NET 7 y .NET 8, que permita la creación y gestión de turnos.
- Desarrollar un front-end en Angular que permita a los usuarios agendar turnos y visualizar la información de los turnos.
- Implementar buenas prácticas de desarrollo y patrones de diseño adecuados.

## 2. Estructura de la API REST

- Crear una solución RESTful en C# .NET 6. También son válidas .NET 7 y .NET 8.
- Definir los modelos de datos para los turnos y las sucursales.
- Crear un servicio de creación y gestión de turnos que permita:
  - Crear un nuevo turno.
  - Obtener la lista de turnos disponibles.
  - Obtener la información de un turno específico.
  - Actualizar la información de un turno.
- Implementar la autenticación y autorización para asegurar que solo los usuarios autorizados puedan acceder a la API.

## 3. Estructura del front-end en Angular

- Crear un proyecto de Angular.
- Definir los componentes para el agendamiento de turnos y la visualización de la información de los turnos.
- Crear un servicio que permita interactuar con la API REST y obtener la información de los turnos.
- Implementar la lógica para agendar turnos y visualizar la información de los turnos.

## 4. Puntos a evaluar

- **Diseño de la arquitectura:** evaluar la estructura de la aplicación y el uso de patrones de diseño adecuados.
- **Buenas prácticas de código:** revisar la calidad del código y asegurar que siga convenciones de nomenclatura y estándares de codificación.
- **Seguridad:** evaluar cómo se abordan las preocupaciones de seguridad, como la autenticación y autorización.
- **Eficiencia y escalabilidad:** considerar la eficiencia y escalabilidad de la solución, especialmente en términos de manejo de solicitudes concurrentes y escalabilidad horizontal.
- **Pruebas unitarias:** verificar la implementación de pruebas unitarias sólidas que cubran las principales funcionalidades de la aplicación.

## 5. Plus

- Implementar pruebas unitarias para la API REST y el front-end en Angular.
- Utilizar herramientas de testing como xUnit, NUnit o Jest para las pruebas unitarias.

## 6. Entregables

- Entregar la prueba en un repositorio en el que se vea el historial de versiones e indicar los pasos previos a la ejecución.
- Un proyecto de ASP.NET Core Web API que incluya un servicio de creación y gestión de turnos.
- Un proyecto de Angular que permita a los usuarios agendar turnos y visualizar la información de los turnos.
- Un informe que describa la arquitectura de la solución y las decisiones de diseño tomadas.
