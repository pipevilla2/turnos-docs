# Manual de usuario

## Sistema de Agendamiento de Turnos Bancarios

Este manual explica cómo solicitar y consultar un turno para ser atendido en una sucursal bancaria.

## 1. Acceder al sistema

1. Abra la aplicación desde el navegador.
2. En la pantalla **Bienvenido**, escriba su número de cédula.
3. La cédula debe contener entre **6 y 15 dígitos** y solo debe incluir números.
4. Seleccione **Validar cédula**.

Si la cédula es válida, el sistema mostrará el menú principal y la cédula identificada.

### Controles disponibles al ingresar la cédula

- **Borrar**: elimina el último dígito escrito.
- **Limpiar**: elimina toda la cédula ingresada.
- **Validar cédula**: inicia la sesión.

## 2. Menú principal

Después de validar la cédula, estarán disponibles estas opciones:

- **Agendar turno**: solicita un nuevo turno en una sucursal.
- **Ver mis turnos**: consulta los turnos asociados a la cédula.
- **Salir**: cierra la sesión y regresa a la pantalla de inicio.

La cédula también se muestra en las pantallas de agendamiento y consulta de turnos.

## 3. Agendar un turno

1. Seleccione **Agendar turno**.
2. En el campo **Sucursal**, elija la sucursal donde desea ser atendido.
3. Seleccione **Agendar turno**.
4. Espere la confirmación del sistema.

Al crear el turno, se mostrarán:

- El código del turno.
- La sucursal seleccionada.
- El estado actual del turno.
- El tiempo disponible para activarlo.

Si no selecciona una sucursal, el sistema solicitará que la elija antes de continuar.

## 4. Activar el turno en la sucursal

Un turno nuevo queda inicialmente en estado **Pendiente**.

1. Diríjase a la sucursal seleccionada.
2. Llegue antes de que termine el contador.
3. Seleccione **Activar turno en sucursal** desde la pantalla de agendamiento, o seleccione **Ver** desde la lista de turnos y luego **Activar turno**.
4. Espere la confirmación.

Tiene **15 minutos** desde la creación del turno para activarlo. Cuando se activa correctamente, el estado cambia a **Activado**.

Si el tiempo termina, el turno pasa a estado **Expirado** y deberá solicitar uno nuevo.

## 5. Consultar mis turnos

1. Seleccione **Ver mis turnos** en el menú principal, o **Mis turnos** desde la barra superior.
2. Revise la tabla con sus turnos.
3. La tabla muestra el código, la sucursal, la fecha de creación y el estado.
4. Seleccione **Ver** para consultar el detalle de un turno y, si corresponde, activarlo.

Si no tiene turnos registrados, aparecerá el mensaje:

> Aún no tienes turnos agendados.

## 6. Estados de un turno

| Estado | Significado |
|---|---|
| **Pendiente** | El turno fue creado, pero todavía debe activarse en la sucursal. |
| **Activado** | El turno fue activado correctamente y está listo para la atención. |
| **Atendido** | La atención del turno finalizó. |
| **Expirado** | No se activó dentro del tiempo permitido. |
| **Cancelado** | El turno fue cancelado. |

## 7. Reglas importantes

- La cédula debe tener entre 6 y 15 dígitos.
- Debe llegar a la sucursal y activar el turno dentro de los 15 minutos posteriores a su creación.
- Se pueden solicitar como máximo 5 turnos por cédula en un mismo día.
- Solo se pueden agendar turnos en sucursales activas.
- Los turnos se consultan utilizando la cédula con la que inició sesión.

## 8. Cerrar sesión

Para terminar la sesión, seleccione **Salir** en la barra superior o en el menú principal. El sistema eliminará la sesión actual y regresará a la pantalla de bienvenida.

## 9. Solución de problemas

### La cédula no es válida

Verifique que tenga entre 6 y 15 dígitos y que no contenga espacios, letras ni otros caracteres.

### No se puede iniciar sesión

Revise la cédula e intente nuevamente. Si el problema continúa, comuníquese con el administrador del sistema.

### No aparecen mis turnos

Confirme que inició sesión con la misma cédula utilizada para agendarlos. También puede actualizar la pantalla y volver a seleccionar **Mis turnos**.

### El turno expiró

El turno no fue activado dentro de los 15 minutos disponibles. Regrese a **Agendar turno** y solicite un nuevo turno.

### Se alcanzó el límite diario

El sistema permite hasta 5 turnos por cédula al día. Podrá solicitar nuevos turnos al comenzar el siguiente día.

## 10. Acceso a la demo

La aplicación está disponible en:

https://gray-field-03d5bbe0f.3.azurestaticapps.net/login
