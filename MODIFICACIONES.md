# Documento de modificaciones — 2° Parcial de Programación III

**Sistema de Gestión de Turnos Médicos**

| | |
|---|---|
| **Integrante** | Enzo Fuentes |
| **Repositorio** | https://github.com/Naitais/programacion-3-parcial |
| **Fecha** | 13 de noviembre de 2026 |

> Para la entrega en Classroom: exportar este documento a PDF (máximo
> 2 páginas) y agregarle las capturas de pantalla indicadas al final.

---

## 1. Validaciones incorporadas

Las validaciones se implementaron **en el backend** y se reflejan en el
frontend mostrando el mensaje de error devuelto por la API. Están separadas
en dos archivos según si necesitan consultar la base de datos.

### a) Validaciones de formato — `backend/validators/turnoValidator.js`

Middleware que se ejecuta antes del controlador y corta el pedido con
**400** si los datos están mal:

- campos obligatorios completos (fecha, hora, paciente, médico, práctica y
  franja horaria);
- formato de fecha `AAAA-MM-DD` y de hora `HH:MM`;
- el **estado** debe ser uno de los tres válidos (`Reservado`, `Confirmado`,
  `Cancelado`): antes la columna aceptaba cualquier texto y `NULL`;
- el **medio de solicitud** debe ser uno de los permitidos;
- los campos numéricos tienen que ser números.

### b) Reglas de negocio — `backend/validators/reglasTurno.js`

Diez reglas que consultan la base de datos antes de guardar o reprogramar:

| # | Regla | Código HTTP |
|---|---|---|
| 1 | No se permiten turnos en fechas u horas anteriores a la actual | 400 |
| 2 | La práctica solicitada tiene que existir | 404 |
| 3 | El turno completo tiene que entrar en el horario de atención del centro (L-V 8 a 20, sábados 8 a 14, domingos cerrado) | 400 |
| 4 | El paciente tiene que estar registrado | 404 |
| 5 | El médico tiene que existir y **poder realizar la práctica** solicitada (`practica_medico`) | 400 |
| 6 | El equipo tiene que ser **compatible con la práctica** (`practica_equipo`) | 400 |
| 7 | El médico tiene que **atender ese día y en esa franja horaria** (`medico_disponibilidad` + `disponibilidad`) | 400 |
| 8 | El **médico** no puede tener otro turno superpuesto | 409 |
| 9 | El **paciente** no puede tener otro turno superpuesto | 409 |
| 10 | El **equipo** no puede estar ocupado en ese horario | 409 |

Y además, en `validarTurnoModificable()`: **un turno cancelado no se puede
reprogramar, confirmar ni volver a cancelar** (409).

**Detección de superposiciones.** Dos turnos se pisan si
`inicioNuevo < finViejo` **y** `inicioViejo < finNuevo`. La hora de fin no
está guardada en la base: se calcula sumando la `duracion` de la práctica
con `ADDTIME` y `SEC_TO_TIME`. La consulta ignora los turnos `Cancelado`
(porque liberan el horario) y excluye el propio turno cuando se reprograma.

**A nivel base de datos** (`database/cambios_parcial2.sql`) se agregó
además un `CHECK` sobre `estado_turno` y se lo pasó a `NOT NULL` con valor
por defecto `Reservado`, para que la regla se cumpla incluso si alguien
inserta directo con SQL.

---

## 2. Gestión de perfiles y sesiones

### Perfiles

Tres roles en la tabla `rol`: **ADMIN**, **MEDICO** y **PACIENTE**. Cada
usuario se vincula con su persona: `usuario.dni_paciente` o
`usuario.matricula_medico`. Esa vinculación es la que permite que cada uno
vea solamente lo suyo.

### Sesiones

- `POST /api/auth/login` verifica el email y la contraseña (hasheada con
  **bcrypt**) y devuelve un **token JWT** firmado con `JWT_SECRET`, que
  vence a las 2 horas.
- El frontend guarda el token en `sessionStorage` (`frontend/js/sesion.js`)
  y lo manda en **cada** pedido con la cabecera `Authorization: Bearer ...`
  (`frontend/js/api.js`).
- `GET /api/auth/perfil` identifica al usuario que inició sesión; el panel
  muestra su nombre, su perfil y su email en el encabezado.
- Si el token vence, el backend devuelve **401** y el frontend cierra la
  sesión y vuelve al login automáticamente.

### Protección de las rutas del backend

- `middlewares/authMiddleware.js` — **autenticación**: valida la firma y el
  vencimiento del token y deja los datos del usuario en `req.usuario`. Si
  falla: **401**.
- `middlewares/autorizarRol.js` — **autorización**: comprueba que el rol
  esté entre los permitidos para esa ruta. Si falla: **403**.
- **Control por dueño del dato** en los controladores
  (`verificarQueElTurnoLeCorresponde`): un paciente no puede acceder al
  turno de otro cambiando el número en la URL, ni un médico a un turno que
  no es de su agenda. Si falla: **403**.
- En el listado, `filtrosSegunRol()` **fuerza** el filtro con el DNI o la
  matrícula que vienen **dentro del token**, ignorando lo que mande el
  navegador.

### Frontend adaptado al perfil

Cada opción del menú declara en el HTML qué roles la pueden ver
(`data-roles="ADMIN,PACIENTE"`) y `dashboard.js` oculta las que no
corresponden. Los botones de cada turno también se arman según el rol y el
estado del turno.

---

## 3. Mejora funcional implementada

**Ciclo de vida completo del turno + búsqueda con filtros.**

| Operación | Ruta | Qué hace |
|---|---|---|
| **Reprogramar** | `PUT /api/turnos/:id/reprogramar` | cambia fecha, hora y franja horaria, volviendo a aplicar **todas** las validaciones. Si el turno estaba `Confirmado`, vuelve a `Reservado` y se limpian `fecha_confirmacion` y `medico_confirmacion`, tal como pide el enunciado |
| **Confirmar** | `PATCH /api/turnos/:id/confirmar` | pasa a `Confirmado` y registra `fecha_confirmacion` (con `NOW()`) y `medico_confirmacion` |
| **Cancelar** | `PATCH /api/turnos/:id/cancelar` | pasa a `Cancelado`; no borra la fila, y el horario queda libre porque las validaciones ignoran los cancelados |
| **Buscar y filtrar** | `GET /api/turnos?fecha_desde=&fecha_hasta=&estado=` | filtros combinables, con `WHERE` dinámico y parámetros preparados |

**Integración de las tres capas:**

- **Frontend**: panel de reprogramación, botones de acción por turno,
  formulario de filtros y formulario de alta con **desplegables
  encadenados** (al elegir la práctica se cargan solo los médicos
  habilitados y los equipos compatibles; al elegir el médico, solo sus
  franjas horarias).
- **Backend**: rutas nuevas con sus middlewares de rol, controladores,
  validaciones y servicios.
- **Base de datos**: consultas de `UPDATE` sobre `turno` y `registra`
  dentro de **transacciones** (`beginTransaction` / `commit` / `rollback`),
  porque el médico del turno está en `registra` y hay que modificar las dos
  tablas de forma atómica.

Se completó además el **CRUD de pacientes**: listado con búsqueda por
nombre, apellido o DNI, y actualización de los datos de contacto con
validación de formato de email y longitud de teléfono.

---

## 4. Archivos modificados y creados

### Backend

| Archivo | Qué contiene |
|---|---|
| `server.js` | middlewares generales, rutas y manejo centralizado de errores |
| `config/database.js` | pool de conexiones a MySQL |
| `config/roles.js` | nombres de los roles |
| `routes/authRoutes.js` | login y perfil |
| `routes/turnoRoutes.js` | rutas de turnos con sus permisos por rol |
| `routes/pacienteRoutes.js` | rutas de pacientes |
| `routes/catalogoRoutes.js` | listas para los desplegables |
| `controllers/authController.js` | login (firma el JWT) y perfil |
| `controllers/turnoController.js` | CRUD, reprogramar, confirmar, cancelar y control por dueño del dato |
| `controllers/pacienteController.js` | listado, ficha y actualización |
| `controllers/catalogoController.js` | prácticas, médicos, equipos y franjas |
| `services/authService.js` | verificación de credenciales con bcrypt |
| `services/turnoService.js` | todo el SQL de turnos, con transacciones |
| `services/pacienteService.js` | SQL de pacientes |
| `services/catalogoService.js` | SQL de los catálogos |
| `validators/turnoValidator.js` | validaciones de formato |
| `validators/reglasTurno.js` | las 10 reglas de negocio |
| `middlewares/authMiddleware.js` | verificación del token JWT |
| `middlewares/autorizarRol.js` | autorización por rol |
| `middlewares/cors.js` | CORS escrito a mano |
| `middlewares/manejadorErrores.js` | manejo centralizado de errores |
| `scripts/test-db.js` | prueba de conexión |
| `scripts/crearUsuarios.js` | crea los usuarios de los tres perfiles |

### Frontend

| Archivo | Qué contiene |
|---|---|
| `index.html` | login, con botones de usuarios de prueba |
| `pages/dashboard.html` | panel con las cinco secciones y `data-roles` en el menú |
| `css/estilos.css` | estilos de las dos pantallas |
| `js/sesion.js` | guarda, lee y borra el token; protege la página |
| `js/api.js` | todas las llamadas al backend; agrega el token y traduce los errores |
| `js/app.js` | lógica del login |
| `js/dashboard.js` | menú por rol, tabla de turnos, acciones, filtros y formularios |

### Base de datos

| Archivo | Qué contiene |
|---|---|
| `database/gestion_turnos.sql` | base original de la cursada |
| `database/cambios_parcial2.sql` | roles, `CHECK` del estado, índices y asignación de médico a los turnos viejos que no tenían fila en `registra` |

**No se incorporaron tecnologías nuevas.** Se usan las mismas cinco
dependencias de la cursada: `express`, `mysql2`, `dotenv`, `bcryptjs` y
`jsonwebtoken`.

---

## 5. Capturas de pantalla a incluir en el PDF

1. Panel del **paciente**: se ve su nombre y perfil en el encabezado, y
   solamente sus turnos.
2. Panel del **administrador**: el menú completo y todos los turnos.
3. Mensaje de error de una **validación** (por ejemplo, fecha anterior a hoy
   o médico con el horario ocupado).
4. Un turno **reprogramado** volviendo al estado `Reservado`.
5. **Thunder Client** devolviendo **401** sin token y **403** al pedir un
   turno de otro paciente.
6. Consulta en **MySQL Workbench** mostrando el turno con su estado
   actualizado.
