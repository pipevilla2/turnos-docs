# Concurrencia y Escalabilidad — Generación de Turnos

## 1. Requisito de la prueba

> **Eficiencia y escalabilidad:** considerar la eficiencia y escalabilidad de la
> solución, especialmente en términos de manejo de solicitudes concurrentes y
> escalabilidad horizontal.

El defecto rompía este requisito: la solución no era segura ante concurrencia ni
apta para escalar horizontalmente (varias instancias del API agravarían el
problema porque cada una mantenía su propio ciclo lectura-escritura).

## 2. Solución adoptada — Incremento atómico en base de datos

Se reemplazó el read-modify-write por una **única operación atómica con bloqueo**
en la base de datos, ejecutada dentro de una transacción:

```sql
MERGE dbo.TurnosConsecutivos WITH (HOLDLOCK) AS target
USING (SELECT @sucursalId AS SucursalId) AS src
    ON target.SucursalId = src.SucursalId
WHEN MATCHED THEN
    UPDATE SET UltimoConsecutivo = target.UltimoConsecutivo + 1
WHEN NOT MATCHED THEN
    INSERT (SucursalId, UltimoConsecutivo) VALUES (src.SucursalId, 1);
```

Claves de la solución:

- **`MERGE` (upsert):** en una sola sentencia crea la fila si no existe o la
  incrementa si ya existe, cubriendo el primer turno de cada sucursal.
- **`WITH (HOLDLOCK)`:** toma un bloqueo de rango sobre la fila de la sucursal y
  lo mantiene hasta el `COMMIT`, serializando el incremento. Dos peticiones
  concurrentes ya no pueden obtener el mismo valor.
- **El índice único `UX_Turnos_CodigoTurno` se conserva** como garantía final: es
  la última línea de defensa ante cualquier fallo de la aplicación.

### Por qué esta opción

| Criterio | Beneficio |
|----------|-----------|
| Eficiencia | Una sola ida a la base de datos; sin transacciones largas ni contención global. |
| Escalabilidad horizontal | El consecutivo vive en la base de datos, no en memoria del proceso: N instancias del API tras un balanceador comparten el mismo contador de forma consistente. |
| Simplicidad | No requiere cambiar el modelo de datos ni introducir componentes externos. |

## 3. Alternativas evaluadas

| Opción | Descripción | Decisión |
|--------|-------------|----------|
| **Incremento atómico (MERGE + HOLDLOCK)** | Una sentencia con bloqueo de fila | **Elegida** |
| Reintento ante clave duplicada | Capturar error 2601/2627 y reintentar | Complemento válido (defensa en profundidad) |
| Concurrencia optimista (`rowversion`) | Token de versión + retry | Válida, más código |
| Transacción `Serializable` completa | Aísla lectura + inserción del turno | Válida, mayor contención |
| SQL Sequence por sucursal | `NEXT VALUE FOR` | Válida, cambia el modelo de datos |

## 4. Consideraciones de escalabilidad horizontal

- **API stateless:** el estado (consecutivo) reside en la base de datos, por lo
  que se pueden ejecutar múltiples instancias del API sin coordinación adicional.
- **I/O asíncrono:** todos los accesos a datos usan `async/await`, liberando
  hilos durante la espera de la base de datos y mejorando el throughput.
- **Servicio de expiración en segundo plano:** opera de forma idempotente para no
  interferir cuando corran varias instancias.
- **Pool de conexiones:** se mantiene el pooling de EF Core / SQL Server para
  soportar concurrencia sin agotar conexiones.
