# Prompt para migrar el módulo de suscripción y bloqueo programado

Copiá desde la línea siguiente y pegalo en Gemini dentro del proyecto de destino.

---

Quiero que analices este proyecto e implementes de punta a punta un módulo de suscripción mensual con bloqueo programado por vencimiento. Adaptá la solución al stack, arquitectura, convenciones, autenticación, base de datos y diseño visual que ya existan en este repositorio. No reemplaces la arquitectura ni agregues un framework paralelo. Antes de editar, inspeccioná el proyecto y localizá el modelo de usuarios, autenticación, roles, rutas/API, acceso a datos, punto de entrada de la interfaz y sistema de estilos.

## Objetivo funcional

El dueño o desarrollador del software debe poder configurar un día de corte mensual. Cinco días antes del vencimiento, los usuarios comunes deben ver una advertencia. Cuando la fecha vence, la aplicación debe quedar bloqueada para esos usuarios hasta que un operador interno reactive el servicio. El bloqueo no debe borrar, modificar ni corromper datos.

Llamá a esta función en código y textos técnicos `subscription`, `subscription gate` o “bloqueo por vencimiento”. No uses nombres ocultos u ofuscados.

## Estados del módulo

Implementá una máquina de estados derivada de los datos persistidos:

1. `UNCONFIGURED`: no existe una configuración completa. La aplicación funciona sin restricciones.
2. `ACTIVE`: existe configuración y faltan más de 5 días para vencer.
3. `WARNING`: faltan entre 0 y 5 días calendario, inclusive.
4. `EXPIRED`: la hora actual ya superó la fecha y hora de vencimiento.
5. `UNKNOWN`: el cliente todavía está cargando el estado o no pudo consultarlo. No lo presentes como una suscripción activa. Mostrá un estado de carga o error de conexión y permití reintentar. La autorización final siempre la decide el backend.

La respuesta normalizada del estado debe tener como mínimo esta forma, adaptando nombres solo si el proyecto tiene una convención firme:

```json
{
  "isConfigured": true,
  "subscriptionDay": 10,
  "nextExpiry": "2026-10-10T02:59:59.999Z",
  "isExpired": false,
  "isWarning": true,
  "daysRemaining": 3
}
```

## Persistencia

Si el proyecto maneja una sola instalación o cliente, creá una configuración singleton:

- `id`: clave primaria fija con valor `1`.
- `subscription_day`: entero nullable entre 1 y 31.
- `next_expiry`: fecha/hora nullable, persistida en UTC.
- `updated_at`: fecha/hora de la última modificación.

Si el sistema es multiempresa o multicliente, no uses un singleton global: asociá la configuración a `tenant_id`, `organization_id` o la entidad equivalente, con una restricción única por cliente. Toda consulta y autorización debe quedar aislada por tenant.

Creá una migración idempotente y compatible con el mecanismo de migraciones existente. No borres ni reinicialices datos. Agregá índices o restricciones útiles y validá el rango 1–31 también en el backend, aunque exista validación en la interfaz.

## Zona horaria y cálculo del vencimiento

Usá una zona horaria de negocio explícita y configurable, por ejemplo `BUSINESS_TIMEZONE=America/Argentina/Buenos_Aires`. No dependas de la zona horaria local del servidor. Guardá el instante final en UTC y convertí a la zona de negocio para calcular y mostrar fechas.

Creá una única función de dominio testeable, equivalente a `calculateNextExpiry(subscriptionDay, now, timezone)`, con estas reglas:

- Obtené año, mes y día de `now` en la zona horaria del negocio.
- Ajustá el día de corte al último día real del mes. Por ejemplo, un corte configurado para el día 31 vence el último día de febrero.
- Si el día calendario de hoy es menor que el día de corte ajustado, el vencimiento ocurre este mismo mes a las `23:59:59.999` de la zona de negocio.
- Si hoy es el día de corte o ya pasó, el vencimiento ocurre el mes siguiente, también al final del día y ajustado a la cantidad de días de ese mes.
- Convertí el resultado a UTC para persistirlo.
- `isExpired = now > nextExpiry`.
- `daysRemaining = max(0, ceil((nextExpiry - now) / 24 horas))`.
- `isWarning = !isExpired && daysRemaining <= 5`.

La configuración inicial y la reactivación después de vencer usan esa misma función. La operación llamada “reactivar” debe estar habilitada únicamente cuando el estado sea `EXPIRED`, tanto en frontend como en backend. Si el producto necesita permitir pagos anticipados, implementá otra operación separada que extienda desde el vencimiento vigente; no hagas que una reactivación temprana recalcule la misma fecha sin extender el período.

## Usuario operador interno

Agregá un rol interno `creator` o `subscription_manager`, eligiendo el nombre que mejor encaje con los roles existentes. Este usuario:

- inicia sesión mediante la autenticación normal del proyecto;
- accede a un panel propio de suscripción;
- nunca queda bloqueado por el vencimiento;
- es el único autorizado a configurar el día de corte y reactivar el servicio;
- no aparece en listados de usuarios, formularios de creación, selectores de roles ni paneles administrativos comunes;
- no puede ser creado, convertido, editado, desactivado o eliminado desde endpoints administrativos generales.

Creá o actualizá este usuario mediante migración o seed idempotente usando variables de entorno obligatorias, por ejemplo `CREATOR_EMAIL` y `CREATOR_PASSWORD`. Hasheá la contraseña con el mecanismo existente. No incluyas credenciales predeterminadas ni secretos en el código, logs, respuestas API, commits o documentación. Si falta una variable obligatoria, el seed debe fallar con un mensaje claro.

No confíes en ocultar el usuario como medida de seguridad: todas las operaciones sensibles deben verificar el rol en el backend.

## API y autorización

Implementá los equivalentes a estos endpoints, respetando el estilo de rutas del proyecto:

- `GET /api/subscription/status`: cualquier usuario autenticado puede consultar el estado correspondiente a su instalación o tenant.
- `POST /api/subscription/configure` con `{ "day": 1..31 }`: solo el operador interno. Hace upsert de la configuración y devuelve el estado actualizado.
- `POST /api/subscription/renew`: solo el operador interno y solo si la suscripción está vencida. Recalcula `nextExpiry`, persiste y devuelve el estado actualizado.

Aplicá autenticación y control de rol en el servidor. Respondé errores con códigos HTTP coherentes y mensajes estructurados.

El bloqueo debe ser efectivo también en backend. Creá un middleware, guard o policy reutilizable que, para usuarios comunes, consulte el estado y rechace las operaciones de negocio cuando esté `EXPIRED`. Usá un error estable, por ejemplo:

```json
{
  "code": "SUBSCRIPTION_EXPIRED",
  "message": "La suscripción está vencida"
}
```

Usá de forma consistente `402 Payment Required` o `423 Locked`, según las convenciones del proyecto. Excluí del bloqueo:

- inicio y cierre de sesión y los endpoints mínimos de sesión;
- consulta del estado de suscripción;
- endpoints de configuración y reactivación, que ya están protegidos por rol;
- health checks;
- el operador interno.

Protegé todas las rutas de negocio aunque el usuario intente llamar la API directamente sin pasar por la interfaz. Evitá una consulta costosa por cada request si la arquitectura permite una caché breve, pero invalidala inmediatamente al configurar o reactivar. En un sistema multi-tenant, la clave de caché debe incluir el tenant.

## Integración en el frontend

Creá un provider, store, composable o servicio global acorde al framework existente. Debe:

- consultar el estado al entrar un usuario autenticado;
- refrescarlo cada 60 segundos;
- volver a consultarlo al recuperar foco o conexión y después de configurar o reactivar;
- cancelar timers y requests cuando corresponda;
- exponer el estado derivado y las acciones autorizadas;
- interpretar también el error `SUBSCRIPTION_EXPIRED` de cualquier request y refrescar el estado, para que el bloqueo aparezca inmediatamente;
- no asumir que un error de red significa “suscripción activa”.

Para usuarios comunes:

- En `ACTIVE` o `UNCONFIGURED`, no muestres nada adicional.
- En `WARNING`, mostrales un aviso superior visible desde toda la aplicación. Debe indicar la fecha límite y “vence hoy”, “vence mañana” o “vence en N días”. Desde 2 días o menos, usá una apariencia urgente.
- Permití minimizar el aviso grande, pero mantené una barra compacta y persistente que permita volver a abrirlo. Volvé a mostrar el aviso completo si cambia la fase o `daysRemaining`.
- En `EXPIRED`, mostrales una pantalla modal a viewport completo, por encima de toda la app, sin forma de cerrarla. Explicá que deben contactar al proveedor para reactivar el servicio. La única acción disponible será cerrar sesión.
- El frontend es una representación del estado; no debe ser la única barrera de seguridad.

Para el operador interno:

- Después del login, redirigilo a un panel separado del producto principal.
- Mostrá tarjetas con día de corte, próximo vencimiento y estado (`Sin configurar`, `Activo`, `Aviso` o `Bloqueado`).
- Incluí un campo numérico 1–31 para configurar el día mensual, con validación y mensajes de éxito/error.
- Incluí una sección para reactivar. El botón solo se habilita en `EXPIRED`, pide confirmación si el sistema visual existente la usa y muestra el nuevo vencimiento al completar.
- Incluí cerrar sesión.
- No le muestres el overlay de advertencia o bloqueo.

Reutilizá componentes, colores, tipografía, iconos, toasts, accesibilidad y comportamiento responsive del proyecto. No copies estilos de otro producto si chocan con el diseño existente.

## Integridad, seguridad y concurrencia

- La fecha y el estado autoritativos se calculan en el servidor. Nunca aceptes `isExpired`, `nextExpiry`, roles o fechas calculadas por el cliente.
- Validá fechas inválidas y configuraciones incompletas de forma segura.
- Evitá condiciones de carrera en configure/renew mediante transacción, upsert o control equivalente.
- Registrá en auditoría quién configuró o reactivó, cuándo, el valor anterior y el nuevo, si el proyecto ya dispone de auditoría. No registres secretos.
- No mezcles la expiración del JWT con la expiración de la suscripción: son estados independientes.
- No borres ni cierres automáticamente operaciones o datos del negocio al vencer. Solo impedí nuevas operaciones hasta la reactivación.

## Pruebas y casos de aceptación

Agregá pruebas siguiendo las herramientas existentes; no introduzcas otro runner si ya hay uno. Como mínimo verificá:

1. Día configurado 10 y fecha actual día 7: vence el día 10 del mes actual al final del día de negocio.
2. Día configurado 10 y fecha actual día 10: vence el día 10 del mes siguiente.
3. Día configurado 31 en febrero: vence el último día de febrero.
4. Cambio de diciembre a enero y un año bisiesto.
5. Exactamente 5 días restantes activa `WARNING`; más de 5 queda `ACTIVE`.
6. Un instante posterior a `nextExpiry` activa `EXPIRED`.
7. Sin configuración queda `UNCONFIGURED` y no bloquea.
8. Un usuario común no puede configurar ni reactivar, aunque llame la API manualmente.
9. Un usuario común vencido no puede ejecutar endpoints de negocio.
10. El operador interno vencido conserva acceso a su panel y puede reactivar.
11. El operador interno no aparece ni puede alterarse desde la administración común de usuarios.
12. Configurar o reactivar actualiza la interfaz sin esperar el siguiente polling.
13. Un fallo de red no se interpreta visualmente como suscripción activa.
14. En multi-tenant, el vencimiento de un cliente no afecta a otro.

## Forma de trabajo y entrega

1. Primero resumí brevemente la arquitectura relevante que encontraste y enumerá los archivos que vas a modificar o crear.
2. Implementá el módulo completo; no te quedes en pseudocódigo ni en una guía.
3. Conservá los cambios ajenos y evitá refactors que no sean necesarios para esta función.
4. Ejecutá migraciones solo de manera segura para el entorno disponible. No borres datos.
5. Ejecutá las pruebas, lint y build que correspondan.
6. Al terminar, informá archivos cambiados, decisiones de adaptación, variables de entorno necesarias, comandos de migración/seed, pruebas ejecutadas y cualquier limitación real pendiente.

Si encontrás una decisión menor no especificada, resolvela siguiendo las convenciones del proyecto. Preguntame únicamente si falta una decisión de producto que cambie de forma material el comportamiento o los datos.

---

