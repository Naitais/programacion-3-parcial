# Guía para la presentación — 2° Parcial Programación III

> Léela en este orden. Las secciones 2, 3, 5, 6 y 15 son las que más
> probabilidad tienen de aparecer en las preguntas.

---

## Índice

0. [Qué había antes y qué agregamos en el parcial](#0-qué-había-antes-y-qué-agregamos-en-el-parcial) ← **leer primero**
1. [Cómo arrancar el proyecto](#1-cómo-arrancar-el-proyecto)
2. [Sincrónico y asincrónico](#2-sincrónico-y-asincrónico)
3. [Qué es un callback](#3-qué-es-un-callback)
4. [Promesas y async / await](#4-promesas-y-async--await)
5. [Qué es Node, npm, Express y MySQL](#5-qué-es-node-npm-express-y-mysql)
6. [La arquitectura: Modelo - Vista - Controlador](#6-la-arquitectura-modelo---vista---controlador)
7. [El recorrido completo de un pedido](#7-el-recorrido-completo-de-un-pedido)
8. [Middlewares](#8-middlewares)
9. [API REST y códigos HTTP](#9-api-rest-y-códigos-http)
10. [Autenticación y autorización](#10-autenticación-y-autorización)
11. [Las sesiones en este proyecto](#11-las-sesiones-en-este-proyecto)
12. [CORS](#12-cors)
13. [Las validaciones (punto A)](#13-las-validaciones-punto-a)
14. [Detalles de SQL](#14-detalles-de-sql)
15. [Preguntas probables con respuesta corta](#15-preguntas-probables-con-respuesta-corta)
16. [Guión de la demostración](#16-guión-de-la-demostración)

---

## 0. Qué había antes y qué agregamos en el parcial

La consigna dice **"no empezar de cero, ampliar lo que ya existe"**, y en la
defensa te van a preguntar *"¿qué modificaste?"*. Esta es la respuesta.
Lo de la columna "Antes" sale del documento **Trabajo Práctico** de la
cursada.

### Lo que ya existía (cursada)

| Parte | Cómo estaba |
|---|---|
| Base de datos | `gestion_turnos` con todas las tablas, incluidas `usuario` y `rol`. Un solo usuario: el admin |
| Servidor | `server.js` con Express, `express.json()` y las rutas `/api/turnos` y `/api/auth` |
| Conexión | `config/database.js` con el pool de `mysql2/promise` y `.env` con `dotenv` |
| CRUD de turnos | Route → Controller → Service. `GET` todos (`SELECT *`), `GET` por id, `POST`, `PUT` y `DELETE` |
| Validaciones | **Casi ninguna**: solo verificaba que el DNI del paciente existiera. En el TP quedaron anotadas como pendientes (estado con cualquier texto, fecha pasada, médico/equipo duplicado, horario del centro...) |
| Login | JWT + bcrypt: `authController`, `authService`, `authRoutes` y el middleware `verificarToken` |
| Roles | `autorizarRol(1)` aplicado a **todas** las rutas de turnos: **solo el ADMIN podía usar el sistema**. Médico y paciente no tenían usuario ni acceso |
| Frontend | `index.html` (login), `dashboard.html` con Inicio y una tabla de Turnos, el token en `sessionStorage`, `api.js` con `fetch` |

### Lo que agregamos en el 2° parcial

**A. Validaciones (punto A de la consigna)**
- `validators/reglasTurno.js` — **archivo nuevo** con 10 reglas de negocio:
  fecha pasada, horario del centro, que el médico haga la práctica, equipo
  compatible, que el médico atienda ese día y franja, y que no se
  superpongan turnos del médico, del paciente ni del equipo.
- `validators/turnoValidator.js` — **ampliado**: campos obligatorios,
  formato de fecha y hora, estado y medio solo con valores válidos.
- `validarTurnoModificable()` — un turno cancelado no se puede tocar.
- `database/cambios_parcial2.sql` — `CHECK` en la base para el estado.

**B. Perfiles y sesiones (obligatorio)**
- Los **3 perfiles reales**: `scripts/crearUsuarios.js` crea un médico y un
  paciente **vinculados** a su matrícula y su DNI (antes solo había admin).
- `autorizarRol` ahora se usa **ruta por ruta** y con nombres de rol
  (`ADMIN`, `MEDICO`, `PACIENTE`), no con el número 1 para todo.
- **Control por dueño del dato** en `turnoController.js`: un paciente no
  ve el turno de otro aunque cambie el número en la URL → 403.
- `filtrosSegunRol()`: cada uno ve solo lo suyo en la lista.
- `GET /api/auth/perfil` y el encabezado del panel con nombre y perfil
  (*identificar al usuario que inició sesión*).
- Frontend: menú con `data-roles`, botones según rol, y si el token vence
  vuelve solo al login.

**C. Mejora funcional elegida: ciclo de vida del turno + filtros**
- Rutas nuevas: `PUT /:id/reprogramar`, `PATCH /:id/confirmar`,
  `PATCH /:id/cancelar`, y filtros `?fecha_desde&fecha_hasta&estado`.
- Formulario de alta con **desplegables encadenados** (rutas nuevas de
  `/api/catalogo`).
- Extra: **CRUD de pacientes** (listado con búsqueda y edición de contacto).

**D. Mejoras de organización** (lo que en el TP figuraba como
"manejo centralizado de errores" y "organización final")
- `middlewares/manejadorErrores.js` y `middlewares/cors.js` (nuevos).
- Crear / reprogramar / borrar con **transacciones** (por la tabla `registra`).
- Comentarios en español en todos los archivos.

### La frase para decir en la defensa

> "Partimos del sistema de la cursada, que tenía el CRUD de turnos y el
> login con JWT pero solo lo podía usar el administrador y casi no
> validaba nada. Agregamos diez reglas de negocio en el backend, los
> perfiles de médico y paciente con control de acceso por rol y por dueño
> del dato, y como mejora funcional el ciclo completo del turno:
> reprogramar, confirmar, cancelar y filtrar."

### Thunder Client (lo vas a usar en la demo)

Es una **extensión de VS Code** para mandarle pedidos HTTP directo a la API,
sin pasar por la página web (como Postman, pero adentro de VS Code). Se
instala desde Extensiones (`Ctrl+Shift+X`) buscando "Thunder Client", y
aparece un rayo en la barra izquierda → **New Request**.

Sirve para demostrar que **la seguridad está en el backend**: aunque alguien
se saltee la interfaz, el servidor igual responde **401** sin token y
**403** si el rol o el dueño del dato no corresponde.

Para mandar el token: pestaña **Auth → Bearer** y pegar el token que
devuelve el login. Los pedidos para la demo están en la sección 16 (B).

---

## 1. Cómo arrancar el proyecto

```
1) Importar la base en MySQL:        database/gestion_turnos.sql
2) Ejecutar los cambios del parcial: database/cambios_parcial2.sql
3) cd backend
4) npm install
5) Revisar backend/.env  (DB_PORT y DB_PASSWORD según tu MySQL)
6) node scripts/test-db.js        -> tiene que decir "Conexion a MySQL: OK"
7) node scripts/crearUsuarios.js  -> crea los 3 usuarios de prueba
8) npm start                      -> servidor en http://localhost:3000
9) Abrir http://localhost:3000 en el navegador
```

**Usuarios de prueba** (en la pantalla de login hay botones que los completan):

| Perfil | Email | Contraseña |
|---|---|---|
| Administrador | `admin@turnos.com` | `AdminTurnos2026!` |
| Médico (mat. 1001) | `medico@turnos.com` | `MedicoTurnos2026!` |
| Paciente (DNI 30100101) | `paciente@turnos.com` | `PacienteTurnos2026!` |

**Con Laragon** (como en esta PC): usuario `root`, contraseña vacía,
puerto `3306`. El `.env` ya quedó así. Antes de `npm start`, apretar
**Start All** en Laragon para que MySQL esté prendido.

Si algo no arranca, el orden para buscar el problema es:
`test-db.js` → consola del servidor → consola del navegador (F12).

---

## 2. Sincrónico y asincrónico

### La idea en una frase

- **Sincrónico**: una cosa después de la otra. Cada línea **espera** que
  termine la anterior.
- **Asincrónico**: se lanza una tarea que tarda (red, disco, base de datos)
  y el programa **sigue de largo**. Cuando la tarea termina, recién entonces
  se ejecuta el código que dejamos preparado para ese momento.

### Por qué JavaScript necesita ser asincrónico

JavaScript es **de un solo hilo** (*single thread*): hace **una sola cosa
a la vez**.

Si una consulta a MySQL tarda 200 milisegundos y el código se quedara
esperando, durante esos 200 ms el servidor **no podría atender a nadie
más**. Con 50 usuarios al mismo tiempo, el sistema se arrastraría.

La solución: las tareas lentas se mandan a hacer "afuera" (al sistema
operativo) y el hilo de JavaScript queda libre para seguir trabajando.
Cuando la tarea termina, avisa, y JavaScript ejecuta el código que quedó
pendiente.

> **Frase para la defensa:** *"JavaScript tiene un solo hilo, así que no
> puede quedarse esperando. Todo lo que tarda —la red, la base de datos—
> se hace de forma asincrónica para que el servidor pueda seguir
> atendiendo otros pedidos mientras espera."*

### Cómo se ve en el código

**Sincrónico** (`backend/validators/reglasTurno.js`):

```js
const hoy = fechaDeHoy();          // instantáneo
if (fecha < hoy) { ... }           // se ejecuta recién cuando terminó la línea de arriba
```

Esto es cálculo puro, en memoria: no tarda nada, así que es sincrónico.

**Asincrónico** (mismo archivo):

```js
const [practicas] = await db.execute('SELECT ... FROM practica WHERE ...');
```

Esto sale de Node, viaja a MySQL, MySQL busca, y la respuesta vuelve. Tarda.
Por eso lleva `await`.

### El Event Loop (si preguntan "cómo hace Node")

Node tiene tres piezas:

1. **Call stack (pila)**: donde se ejecuta el código. Solo cabe una cosa a la vez.
2. **Las APIs de afuera**: red, archivos, base de datos. Trabajan en paralelo,
   fuera de la pila.
3. **La cola de tareas (callback queue)**: donde se forman los resultados que
   ya llegaron, esperando turno.

El **Event Loop** es el ciclo que mira: *"¿la pila está vacía? Entonces
agarro lo primero de la cola y lo ejecuto"*. Así vuelve a entrar el código
que quedó pendiente.

### Ejemplo clásico que lo demuestra

```js
console.log('1');
setTimeout(function () { console.log('2'); }, 0);
console.log('3');
```

Imprime **1, 3, 2**. Aunque el tiempo de espera sea 0, el `setTimeout` no
se ejecuta ahí mismo: su función se va a la cola y el Event Loop la trae
recién cuando terminó todo el código sincrónico.

---

## 3. Qué es un callback

### Definición

Un **callback** es una **función que se pasa como argumento a otra
función**, para que la ejecute más tarde.

La clave: **yo no la llamo nunca**. La entrego y alguien más decide cuándo
corresponde ejecutarla. De ahí el nombre: "llamame de vuelta".

En JavaScript se puede hacer porque las funciones son **valores**: se
pueden guardar en variables, pasar como parámetro y devolver desde otra
función (se dice que son *funciones de primera clase*).

### Los cuatro callbacks que tiene este proyecto

**1) Un evento del usuario** — `frontend/js/app.js`

```js
formulario.addEventListener('submit', async function (evento) {
    evento.preventDefault();
    ...
});
```

Esa función es un callback. Se la entregamos al navegador, que la guarda y
la ejecuta **el día que el usuario apriete el botón**. Puede no ejecutarse
nunca (si nadie entra) o ejecutarse 50 veces.

**2) El arranque del servidor** — `backend/server.js`

```js
app.listen(PUERTO, function () {
    console.log('Servidor funcionando...');
});
```

El mensaje se imprime cuando el servidor **terminó** de levantarse, no antes.

**3) Un middleware de Express** — `backend/middlewares/authMiddleware.js`

```js
function verificarToken(req, res, next) { ... next(); }
```

`verificarToken` **es** un callback: nosotros se lo damos a Express
(`router.use(verificarToken)`) y Express lo ejecuta en cada pedido.
Y `next` **también** es un callback: lo recibimos como parámetro y lo
llamamos para decir "listo, seguí con el que viene".

**4) Recorrer una lista** — `frontend/js/dashboard.js`

```js
turnos.map(function (turno) { return `<tr>...</tr>`; });
```

Acá el callback **no** es asincrónico. Es un callback sincrónico: `map` lo
ejecuta una vez por cada elemento, de inmediato.

> **Ojo con esto en la defensa:** callback y asincrónico **no son lo mismo**.
> Un callback es "una función que paso para que otro la llame"; el que está
> en `map` se ejecuta ya mismo. Lo que pasa es que *casi todo* lo
> asincrónico se resolvía históricamente con callbacks.

### El problema: callback hell

Cuando una tarea depende de la anterior, los callbacks se van anidando:

```js
// Así se escribía antes (esto NO está en el proyecto)
db.query('SELECT ...', function (error, practica) {
    db.query('SELECT ...', function (error, medico) {
        db.query('SELECT ...', function (error, turnos) {
            // tres niveles y recién arrancamos
        });
    });
});
```

Se le dice **callback hell** o "pirámide de la perdición": difícil de leer y
el manejo de errores se repite en cada nivel. Para resolverlo se inventaron
las **promesas**, y arriba de las promesas, `async/await`.

---

## 4. Promesas y async / await

### Promesa

Una **promesa** (`Promise`) es un objeto que representa **un resultado que
todavía no llegó**. Tiene tres estados:

- *pendiente* — la tarea está en curso
- *cumplida* (`resolved`) — llegó el resultado
- *rechazada* (`rejected`) — hubo un error

### async / await

`async/await` es **azúcar sintáctico** sobre las promesas: por debajo son
promesas, pero se escribe como si fuera código sincrónico, de arriba hacia
abajo.

- `async` antes de una función significa dos cosas: que **devuelve una
  promesa** y que **adentro se puede usar `await`**.
- `await` significa: *"pausá esta función hasta que la promesa se resuelva,
  y dame el valor de una vez"*.

```js
// Con promesas "a mano"
function cargar() {
    return api.obtenerTurnos().then(function (datos) {
        dibujarTabla(datos.turnos);
    });
}

// Con async/await: lo mismo, más legible
async function cargar() {
    const datos = await api.obtenerTurnos();
    dibujarTabla(datos.turnos);
}
```

### Lo más importante que te pueden preguntar

**¿`await` bloquea el servidor?** → **No.** `await` pausa **esa función**,
no el programa. Mientras esa función espera a MySQL, Node sigue atendiendo
otros pedidos. Por eso un solo hilo alcanza para muchos usuarios.

**¿Cómo se manejan los errores?** Con `try / catch`, igual que el código
sincrónico. Eso es otra ventaja contra los callbacks, que necesitaban
chequear un parámetro `error` en cada nivel.

```js
try {
    const datos = await api.iniciarSesion(email, password);   // puede fallar
    guardarSesion(datos.token, datos.usuario);
} catch (error) {
    mensaje.textContent = error.message;   // acá caen todos los errores
} finally {
    boton.disabled = false;                // esto se ejecuta SIEMPRE
}
```

**¿Qué pasa si me olvido el `await`?** No te da el valor: te da la promesa
sin resolver. Es el bug más común. Si `turnos` imprime
`Promise { <pending> }`, falta un `await`.

---

## 5. Qué es Node, npm, Express y MySQL

### Node.js

JavaScript nació para correr **dentro del navegador**. **Node.js** es un
programa que toma el motor de JavaScript de Chrome (el motor **V8**) y lo
saca afuera, para poder ejecutar JavaScript **en la computadora como
cualquier otro programa**.

Eso es lo que permite usar JavaScript en el backend: leer archivos, abrir
un puerto de red, conectarse a MySQL. Cosas que el navegador no deja hacer.

- En el navegador existe `window` y `document`. En Node **no existen**.
- En Node existe `process`, `require`, el sistema de archivos. En el
  navegador **no existen**.
- `console.log` existe en los dos.

**Node no es un lenguaje ni un framework**: es el *entorno de ejecución*
(*runtime*) de JavaScript fuera del navegador.

### npm

**npm** = *Node Package Manager*. Es el instalador de librerías que viene
con Node.

- `package.json` es la **lista de ingredientes** del proyecto: nombre,
  scripts y dependencias con su versión.
- `npm install` lee ese archivo y baja todo a la carpeta `node_modules`.
- `node_modules` **no se sube a GitHub** (está en `.gitignore`), porque se
  reconstruye con `npm install`.

Las 5 dependencias de este proyecto:

| Paquete | Para qué |
|---|---|
| `express` | el framework del servidor web |
| `mysql2` | el driver para hablar con MySQL (versión con promesas) |
| `dotenv` | lee el archivo `.env` y lo carga en `process.env` |
| `bcryptjs` | hashea y verifica las contraseñas |
| `jsonwebtoken` | crea y valida los tokens JWT |

### Express

Con Node "pelado" se puede hacer un servidor web, pero hay que programar a
mano la lectura de la URL, el método, las cabeceras... **Express** es un
**framework** que pone todo eso resuelto y agrega dos cosas clave:

- el **router**: "si llega un POST a `/api/turnos`, ejecutá esta función";
- los **middlewares**: una fila de funciones por la que pasa cada pedido.

### MySQL

El **motor de base de datos** donde viven los datos de verdad
(31 turnos, 20 médicos, 30 pacientes...). Es un programa aparte, que escucha
en su propio puerto (3307 en esta configuración). Node se conecta a él como
un cliente más.

### Cómo se encadena todo

```
Navegador  --HTTP-->  Express (Node)  --SQL-->  MySQL
   (vista)             (backend)              (datos)
```

---

## 6. La arquitectura: Modelo - Vista - Controlador

### Para qué sirve separar en capas

Para que **cada archivo tenga un solo trabajo**. Si todo estuviera en
`server.js`, un archivo de 2000 líneas, cualquier cambio sería un riesgo.
Con capas: si hay que cambiar una consulta SQL, se toca el *service*; si
hay que cambiar un color, el CSS. **Cada cambio queda contenido.**

### Las tres capas clásicas

| Capa | Qué hace | Dónde está en este proyecto |
|---|---|---|
| **Modelo** | Los datos y las reglas del negocio. No sabe nada de pantallas. | `services/` + `validators/` + la base MySQL |
| **Vista** | Lo que ve y toca el usuario. No tiene reglas de negocio. | `frontend/` (HTML, CSS, JS) |
| **Controlador** | El intermediario. Recibe el pedido, coordina y responde. | `controllers/` |

> Si en la cursada lo llamaron **"controlador, visor y vista"**, es la
> misma idea: el *visor* es la parte que muestra los datos (nuestras
> pantallas y la tabla que dibuja `dashboard.js`) y la *vista* es la
> estructura HTML. Lo importante es la separación: **datos y reglas /
> presentación / coordinación**.

### Cómo quedó, capa por capa

```
frontend/                          VISTA
  index.html, dashboard.html       estructura de la pantalla
  css/estilos.css                  presentación
  js/api.js                         habla con el backend (fetch)
  js/dashboard.js                   dibuja los datos y escucha los clics
  js/sesion.js                      guarda el token en el navegador

backend/
  server.js                        arranque y configuración general
  routes/                          QUÉ dirección existe y QUIÉN puede entrar
  middlewares/                     controles que se ejecutan ANTES (token, rol, CORS, errores)
  validators/                      REGLAS (formato y reglas de negocio)
  controllers/                     CONTROLADOR: coordina y arma la respuesta HTTP
  services/                        MODELO: todo el SQL
  config/                          conexión a la base y constantes

database/                          la base de datos y los cambios del parcial
```

### La regla que hay que poder recitar

- **El router** sabe *qué direcciones existen* y *qué rol puede entrar*.
  No sabe SQL.
- **El controlador** lee `req`, decide y arma `res`. **No escribe SQL.**
- **El service** escribe SQL. **No sabe que existe HTTP** (no tiene `req`
  ni `res` ni códigos 404).
- **El validator** dice si los datos sirven. No guarda nada.

> **Pregunta trampa típica:** *"¿por qué el service no devuelve un 404?"*
> Porque 404 es un concepto de HTTP y el service no sabe nada de HTTP. El
> service devuelve `undefined` si no encontró el turno, y **el controlador**
> traduce ese `undefined` a un 404. Así el mismo service se podría usar
> desde otro lugar que no sea una API web.

---

## 7. El recorrido completo de un pedido

Esta es **la respuesta más valiosa** de toda la presentación. Si la contás
bien, demostrás que entendés el proyecto entero.

**Caso: el paciente aprieta "Cancelar" en el turno 45.**

```
 1. dashboard.js       el clic entra al escuchador del <tbody>
                       lee data-accion="cancelar" y data-id="45"
         |
 2. api.js             pedir('/turnos/45/cancelar', 'PATCH')
                       agrega la cabecera Authorization: Bearer <token>
         |
 3. ===== la red =====  PATCH http://localhost:3000/api/turnos/45/cancelar
         |
 4. server.js          middleware de CORS  -> pone las cabeceras
                       express.json()      -> lee el cuerpo
         |
 5. server.js          app.use('/api/turnos', turnoRoutes)
         |
 6. turnoRoutes.js     router.use(verificarToken)
                       -> ¿hay token? ¿la firma es válida? ¿no venció?
                       -> si está mal: 401 y termina acá
                       -> si está bien: req.usuario = datos del token, next()
         |
 7. turnoRoutes.js     autorizarRol(ADMIN, MEDICO, PACIENTE)
                       -> ¿el rol está en la lista? si no: 403
         |
 8. turnoController    cancelarTurno(req, res, next)
                       a) turnoService.obtenerTurnoPorId(45)
                       b) ¿existe? si no: 404
                       c) ¿el turno es de ESTE paciente? si no: 403
                       d) ¿no está ya cancelado? si lo está: 409
         |
 9. turnoService       UPDATE turno SET estado_turno='Cancelado' WHERE id_turno=45
         |
10. ===== MySQL =====  modifica la fila y devuelve affectedRows: 1
         |
11. turnoController    res.json({ mensaje: 'Turno cancelado correctamente', turno })
         |
12. api.js             respuesta.ok es true -> devuelve los datos
         |
13. dashboard.js       mostrarAviso(mensaje, 'ok') y cargarTurnos() de nuevo
```

Y si algo explota en el camino (por ejemplo MySQL está apagado), el
controlador hace `next(error)` y el pedido salta directo al paso final:
`manejadorErrores.js`, que lo escribe en la consola del servidor y le
contesta al navegador un JSON con `{ error: ... }`.

**Fijate que hay CUATRO controles distintos** antes de tocar la base:
token (401), rol (403), existencia (404) y propiedad del dato (403). Eso es
lo que pide la consigna cuando dice "verificar que un usuario no pueda
acceder a funcionalidades de otro perfil".

---

## 8. Middlewares

### Qué es

Un **middleware** es una función que se ejecuta **en el medio**, entre que
llega el pedido y que responde el controlador. Recibe tres parámetros:

```js
function miMiddleware(req, res, next) { ... }
```

Y tiene exactamente **dos salidas posibles**:

1. **llamar a `next()`** → "todo bien, seguí con el siguiente";
2. **responder** (`res.status(...).json(...)`) → corta la cadena ahí mismo;
   el controlador **nunca se ejecuta**.

Si no hace ninguna de las dos, el pedido **se queda colgado** para siempre
(el navegador queda cargando). Es un error típico: olvidarse el `next()`.

### Los middlewares de este proyecto

| Middleware | Qué controla | Si falla |
|---|---|---|
| `cors.js` | que el frontend de otro puerto pueda hablar | el navegador bloquea |
| `express.json()` | convierte el cuerpo JSON en `req.body` | 400 |
| `authMiddleware.js` | que el token exista y sea válido | **401** |
| `autorizarRol.js` | que el rol tenga permiso | **403** |
| `turnoValidator.js` | que los datos tengan la forma correcta | **400** |
| `manejadorErrores.js` | atrapa cualquier error no previsto | **500** |

### El orden importa (y lo pueden preguntar)

Express ejecuta los middlewares **en el orden en que se escriben**. Por eso
en `server.js` va primero CORS, después `express.json()`, después las rutas
y **al final** el manejador de errores.

Si pusiera `express.json()` después de las rutas, `req.body` llegaría
`undefined` a todos los controladores.

### Por qué `autorizarRol` se escribe distinto

```js
function autorizarRol(...rolesPermitidos) {
    return function (req, res, next) { ... };
}
```

Es una **función que devuelve un middleware** (una "fábrica"). Hace falta
porque Express solo le pasa `(req, res, next)` a los middlewares: no hay
forma de pasarle además la lista de roles. Entonces la función de afuera
recibe los roles y la de adentro —la que Express ejecuta— se los
**recuerda** gracias a las **clausuras** (*closures*) de JavaScript.

Por eso en las rutas se escribe `autorizarRol(ROLES.ADMIN)` **con
paréntesis** (lo llamamos nosotros, y el resultado es el middleware),
mientras que `verificarToken` va **sin paréntesis** (se lo pasamos entero
a Express para que lo llame él).

---

## 9. API REST y códigos HTTP

### Qué es una API REST

Una forma de organizar un backend: cada **recurso** (los turnos, los
pacientes) tiene su **dirección**, y lo que se hace con él lo dice el
**método HTTP**. Se comunica con **JSON** y **no guarda estado** entre
pedidos (*stateless*): cada pedido viaja con todo lo necesario para ser
entendido, incluido el token.

### Los métodos

| Método | Significa | En el proyecto |
|---|---|---|
| `GET` | traer datos (no modifica nada) | `GET /api/turnos` |
| `POST` | crear algo nuevo | `POST /api/turnos` |
| `PUT` | reemplazar datos de algo que existe | `PUT /api/turnos/45/reprogramar` |
| `PATCH` | modificar **una parte** | `PATCH /api/turnos/45/cancelar` |
| `DELETE` | borrar | `DELETE /api/turnos/45` |

> *"¿Por qué cancelar es PATCH y no DELETE?"* Porque cancelar **no borra**:
> cambia el estado a `Cancelado` y la fila queda para el historial. Y es
> PATCH y no PUT porque solo modificamos un campo, no el turno completo.

### Los códigos que usamos y qué significa cada uno

| Código | Nombre | Cuándo lo devolvemos |
|---|---|---|
| 200 | OK | salió bien |
| 201 | Created | se creó el turno |
| **400** | Bad Request | datos mal o regla de negocio no cumplida |
| **401** | Unauthorized | **no sé quién sos** (falta token o venció) |
| **403** | Forbidden | **sé quién sos, pero no te corresponde** |
| 404 | Not Found | no existe |
| **409** | Conflict | choca con algo que ya existe (horario ocupado, turno cancelado) |
| 500 | Internal Server Error | se rompió el servidor |

> **La diferencia 401 / 403 es pregunta de examen.** 401 = *no estás
> identificado*. 403 = *estás identificado pero no tenés permiso*.

### Dónde viajan los datos de un pedido

| Lugar | Cómo se lee | Ejemplo |
|---|---|---|
| la URL | `req.params` | `/turnos/45` → `req.params.id`  |
| la query | `req.query` | `/turnos?estado=Reservado` → `req.query.estado` |
| el cuerpo | `req.body` | el JSON del POST |
| las cabeceras | `req.headers` | `Authorization: Bearer ...` |

---

## 10. Autenticación y autorización

Son **dos cosas distintas** y conviene decirlo así:

- **Autenticación** = *¿quién sos?* → el login y el token.
  Lo hace `authMiddleware.js`. Si falla: **401**.
- **Autorización** = *¿qué podés hacer?* → los roles.
  Lo hace `autorizarRol.js`. Si falla: **403**.

### Cómo se guardan las contraseñas (bcrypt)

En la base **no se guarda la contraseña**: se guarda un **hash**.

- Un hash es el resultado de pasar la contraseña por una función **de un
  solo sentido**: de la contraseña se llega al hash, pero **del hash no se
  puede volver** a la contraseña.
- Para verificar el login, bcrypt vuelve a hashear lo que escribió el
  usuario y compara los dos hashes (`bcrypt.compare`).
- bcrypt es **lento a propósito** (el `10` del `bcrypt.hash(password, 10)`
  son las vueltas que da). Así, probar millones de contraseñas por fuerza
  bruta se vuelve carísimo.
- bcrypt además agrega un **salt** (un texto al azar) a cada contraseña, así
  dos usuarios con la misma contraseña tienen hashes distintos.

> Si alguien roba la base de datos, no tiene las contraseñas: tiene hashes.

### Cómo funciona el token JWT

Un **JWT** (JSON Web Token) es un texto con tres partes separadas por puntos:

```
eyJhbGciOiJIUzI1NiJ9 . eyJpZF91c3VhcmlvIjoxLCJub21icmVfcm9sIjoiQURNSU4ifQ . 4f2b...
     encabezado                        datos (payload)                        firma
```

- Las dos primeras partes son JSON codificado en **base64**: **se pueden
  leer**. **No están encriptadas.** Por eso adentro del token nunca se
  ponen datos secretos (ahí va el id, el email y el rol, nada más).
- La tercera parte es la **firma**, que el servidor calcula con la clave
  secreta `JWT_SECRET` (está en el `.env`).

**La parte importante:** si un usuario edita el token para ponerse
`"nombre_rol": "ADMIN"`, los datos cambian pero **la firma ya no coincide**.
`jwt.verify()` lo detecta y devuelve 401. Y no puede recalcular la firma
porque **no tiene la clave secreta**, que nunca sale del servidor.

> **Pregunta probable:** *"¿y si te robo el token?"* Es un riesgo real: el
> que tiene el token es el usuario. Por eso (a) el token **vence** en 2 horas
> (`expiresIn: '2h'`), (b) va en `sessionStorage` y no en `localStorage`, y
> (c) en producción iría todo por **HTTPS** para que no se pueda leer en el
> camino.

---

## 11. Las sesiones en este proyecto

### Las dos formas de manejar sesiones

| | **Sesión clásica (con estado)** | **JWT (sin estado)** — la que usamos |
|---|---|---|
| Dónde está la info | en el **servidor** (memoria o base) | **en el token**, en el cliente |
| Qué guarda el cliente | un id de sesión en una cookie | el token completo |
| El servidor recuerda | sí, tiene la lista de sesiones | **no recuerda nada** |
| Cerrar sesión | se borra del servidor | se borra el token del navegador |

Nosotros usamos **JWT, sin estado**: el backend **no guarda ninguna lista
de usuarios conectados**. Cada pedido llega con su token y el backend lo
valida de cero. Eso es lo que lo hace *stateless*, como pide REST.

### El ciclo completo de la sesión acá

```
1. LOGIN      POST /api/auth/login con email y contraseña
              -> authService compara el hash con bcrypt
              -> authController firma el token con jwt.sign()
              -> responde { token, usuario }

2. GUARDAR    sesion.js  ->  sessionStorage.setItem('token', token)

3. USAR       api.js agrega en CADA pedido:
              Authorization: Bearer <token>
              -> authMiddleware hace jwt.verify() y llena req.usuario

4. IDENTIFICAR  GET /api/auth/perfil devuelve quién inició sesión
                (el panel lo muestra arriba a la derecha)

5. CERRAR     sessionStorage.removeItem('token') y vuelta al login

6. VENCER     a las 2 horas jwt.verify() falla -> 401
              -> api.js detecta el 401, avisa y manda al login
```

### `sessionStorage` vs `localStorage` vs cookies

- `sessionStorage`: se borra **al cerrar la pestaña**. Es el que usamos.
- `localStorage`: queda guardado **para siempre** hasta que se borre.
- cookies: se mandan solas en cada pedido; con la opción `httpOnly` ni
  JavaScript las puede leer (es la opción más segura, pero no es la que
  trabajamos en la cursada).

> **Pregunta probable:** *"si yo edito a mano el rol que está guardado en
> sessionStorage, ¿me convierto en admin?"*
> **No.** Lo guardado en el navegador **solo decide qué botones se ven**. En
> cuanto uses uno, el backend lee el **rol que está firmado dentro del
> token** —no el del sessionStorage— y te devuelve **403**. Por eso decimos
> que el frontend se adapta por comodidad y la seguridad está en el backend.

---

## 12. CORS

### El problema

El navegador tiene una regla de seguridad llamada **same-origin policy**:
una página solo puede pedir datos por JavaScript **al mismo origen** del
que se bajó.

Un **origen** son tres cosas juntas: **protocolo + dominio + puerto**.

```
Frontend (Live Server):  http://127.0.0.1:5500
Backend  (Express):      http://localhost:3000
                                        ^^^^  cambia el puerto -> OTRO origen
```

Sin permiso, el navegador **bloquea la respuesta** y en la consola aparece
el error clásico: *"has been blocked by CORS policy"*.

Esa regla existe para que una página maliciosa no pueda pedirle datos en tu
nombre a tu banco usando tus cookies.

### La solución

**CORS** = *Cross-Origin Resource Sharing*. Es el mecanismo por el cual el
**servidor** avisa, con unas cabeceras en la respuesta, qué orígenes tienen
permiso:

```js
res.setHeader('Access-Control-Allow-Origin', origen);
res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
```

Está en `backend/middlewares/cors.js`, con una **lista blanca** de orígenes
permitidos.

> **Tres cosas para no equivocarse:**
> 1. **CORS lo aplica el navegador, no el servidor.** Thunder Client o Postman
>    no tienen CORS: por eso ahí los pedidos funcionan igual. Si en el
>    navegador falla y en Thunder Client funciona, el problema es CORS.
> 2. **`Access-Control-Allow-Headers` con `Authorization` es obligatorio acá.**
>    Sin esa línea, el navegador no deja mandar el token y todo el login no
>    serviría de nada.
> 3. **El preflight:** antes de un POST o un PUT con JSON, el navegador manda
>    solo un pedido `OPTIONS` para preguntar "¿puedo?". Nuestro middleware lo
>    contesta con un `204` y no sigue hasta el controlador. Si ves un OPTIONS
>    en la consola del servidor, es eso.

> **Dato útil:** en este proyecto el backend también sirve el frontend
> (`express.static`). Si entrás por `http://localhost:3000`, frontend y
> backend **son el mismo origen** y CORS ni se activa. Si entrás por Live
> Server en el 5500, CORS es imprescindible. Sirve para mostrar las dos
> cosas en la demostración.

---

## 13. Las validaciones (punto A)

Están todas en **dos archivos**, y la división es la clave para explicarlas:

### `validators/turnoValidator.js` — la **forma** de los datos

No toca la base de datos. Es un control rápido y barato. Devuelve **400**.

- campos obligatorios completos;
- formato de fecha (`AAAA-MM-DD`) y de hora (`HH:MM`);
- el estado es uno de los tres válidos (antes se podía guardar cualquier texto);
- el medio de solicitud es uno de los permitidos;
- los campos numéricos son números.

### `validators/reglasTurno.js` — las **reglas del negocio**

Necesitan preguntarle a la base. Son 10 reglas numeradas en el archivo:

| # | Regla | Código |
|---|---|---|
| 1 | la fecha y hora no pueden ser anteriores a ahora | 400 |
| 2 | la práctica tiene que existir (y de ahí sale la duración) | 404 |
| 3 | el turno entra en el horario del centro (L-V 8-20, Sáb 8-14, Dom cerrado) | 400 |
| 4 | el paciente está registrado | 404 |
| 5 | el médico existe **y puede hacer esa práctica** (`practica_medico`) | 400 |
| 6 | el equipo es **compatible** con la práctica (`practica_equipo`) | 400 |
| 7 | el médico **atiende** ese día y en esa franja (`medico_disponibilidad`) | 400 |
| 8 | el **médico** no tiene otro turno superpuesto | 409 |
| 9 | el **paciente** no tiene otro turno superpuesto | 409 |
| 10 | el **equipo** no está ocupado | 409 |

Y aparte, `validarTurnoModificable()`: **un turno cancelado no se puede
reprogramar, confirmar ni volver a cancelar** (409).

### La superposición, que es la más interesante

Dos turnos **se pisan** si:

```
inicioNuevo < finViejo   Y   inicioViejo < finNuevo
```

Hay que razonarlo con un dibujo: si el nuevo **termina antes** de que
empiece el viejo, no se pisan; si **empieza después** de que el viejo
terminó, tampoco. En cualquier otro caso, sí.

En SQL:

```sql
AND ? < ADDTIME(t.hora_turno, SEC_TO_TIME(pr.duracion * 60))   -- inicioNuevo < finViejo
AND t.hora_turno < ?                                           -- inicioViejo < finNuevo
```

El `fin` **no está guardado** en la base: se calcula sumándole a
`hora_turno` la `duracion` de la práctica (de ahí el JOIN con `practica`).

Dos detalles importantes:

- **`estado_turno <> 'Cancelado'`**: un turno cancelado libera el horario,
  así que no cuenta para la superposición.
- **`t.id_turno <> ?`**: al reprogramar, hay que **excluir el propio turno**;
  si no, choca consigo mismo y nunca dejaría guardar.

> **Pregunta probable:** *"¿por qué no lo resolviste con un UNIQUE en la
> base de datos?"* Por dos razones: (1) un UNIQUE compara valores exactos,
> y un turno de 09:00 que dura 45 minutos se pisa con otro de 09:30, que
> para el UNIQUE es un valor distinto; (2) un UNIQUE no entiende de estados,
> así que impediría reutilizar un horario que quedó libre por una
> cancelación. Está explicado en `database/cambios_parcial2.sql`.

### Por qué validar en el backend y no solo en el frontend

Porque **el frontend se puede saltear**. Con Thunder Client se le pega
directo a la API sin pasar por ninguna pantalla. El frontend valida para
dar una buena experiencia (avisar rápido, filtrar los desplegables); el
backend valida **porque es el único lugar seguro**.

**Las dos capas trabajan juntas:**
- el frontend pone `min` de hoy en el campo fecha y filtra los médicos por
  práctica, así el usuario casi no puede equivocarse;
- el backend vuelve a controlar **todo** y devuelve el mensaje de error, que
  el frontend muestra en el cartel rojo.

---

## 14. Detalles de SQL

### Los `?` (consultas preparadas)

```js
db.execute('SELECT * FROM turno WHERE id_turno = ?', [id]);
```

Los valores **nunca** se pegan con `+` dentro del texto del SQL. Se mandan
aparte y el driver los escapa. Eso evita la **inyección de SQL**: si alguien
escribe `1; DROP TABLE turno`, se trata como un **valor**, no como código.

### `JOIN` vs `LEFT JOIN`

- `JOIN` (interno): trae solo las filas que **tienen pareja** en las dos
  tablas. Lo usamos con `paciente` y `practica`, que son obligatorios en un
  turno.
- `LEFT JOIN`: trae **todas** las filas de la izquierda, y si no hay pareja
  pone `NULL`. Lo usamos con `equipo`, `registra` y `medico`, que pueden
  faltar.

> Si usáramos `JOIN` con `equipo`, los turnos sin equipo **desaparecerían**
> de la lista. Es un error muy común y se lo puede preguntar.

### Por qué `DATE_FORMAT` en la fecha

```sql
DATE_FORMAT(t.fecha_turno, '%Y-%m-%d') AS fecha_turno
```

Si traemos la fecha tal cual, `mysql2` la convierte en un objeto `Date` de
JavaScript y al pasarla a JSON queda `"2026-11-20T03:00:00.000Z"`: **se
corre por la zona horaria** y puede mostrar el día anterior. Formateándola
en SQL llega como texto `"2026-11-20"`, se muestra bien y además entra
directo en un `<input type="date">`.

### Las transacciones

Crear un turno son **dos INSERT**: uno en `turno` y otro en `registra` (la
tabla que relaciona el turno con su médico y su franja).

Si el primero funciona y el segundo falla, quedaría **un turno sin médico**:
datos corruptos. La transacción lo evita:

```js
await conexion.beginTransaction();   // arranca
...                                  // los dos inserts
await conexion.commit();             // confirma los dos
// y si algo falla:
await conexion.rollback();           // deshace TODO
```

**O se guardan los dos, o no se guarda ninguno.** Es la propiedad de
**atomicidad** (la A de ACID).

Dos detalles técnicos: la transacción tiene que viajar **por la misma
conexión** (por eso `db.getConnection()` en vez de `db.query` directo), y
hay que hacer `conexion.release()` en el `finally` para devolverla al pool.

### El pool de conexiones

Abrir una conexión a MySQL tarda. El **pool** abre varias una sola vez y las
va prestando; cuando una consulta termina, la conexión vuelve al pool.
Está configurado en `config/database.js` con `connectionLimit: 10`.

### Dónde está el médico de un turno

Esto conviene tenerlo claro porque **no es obvio** en el modelo: la tabla
`turno` **no tiene** una columna `matricula_medico`. El médico está en la
tabla **`registra`** (`matricula_medico`, `id_turno`, `id_disponibilidad`).
Por eso todas las consultas de turnos hacen `JOIN registra`, y por eso
crear un turno son dos INSERT con transacción.

---

## 15. Preguntas probables con respuesta corta

**1. ¿Qué modificaste para este parcial?**
Dos grupos de cosas. Validaciones nuevas en el backend (10 reglas de negocio
en `validators/reglasTurno.js` más los controles de formato en
`turnoValidator.js`), y el manejo completo de perfiles y sesiones con los
tres roles, más la mejora funcional del ciclo de vida del turno:
reprogramar, confirmar, cancelar y búsqueda con filtros.

**2. ¿Qué es asincrónico y por qué lo usás?**
Porque JavaScript tiene un solo hilo. Si se quedara esperando a MySQL no
podría atender a nadie más. Las tareas lentas se lanzan y el hilo sigue
libre; cuando el resultado llega, el Event Loop ejecuta el código pendiente.
En el proyecto se ve en todos los `await db.execute(...)` y en los `fetch`.

**3. ¿Qué es un callback?**
Una función que paso como argumento a otra función para que la ejecute más
tarde. Yo no la llamo: la entrego. Ejemplo del proyecto:
`formulario.addEventListener('submit', función)` — esa función la guarda el
navegador y la ejecuta cuando el usuario envía el formulario. Y `next` en
los middlewares también es un callback.

**4. ¿Por qué el controlador no tiene SQL?**
Por separación de responsabilidades. El controlador maneja HTTP (lee `req`,
arma `res`, elige el código de estado) y el service maneja los datos. Así, si
cambia una consulta, se toca un solo archivo, y el service se podría
reutilizar desde otro lugar que no sea una API web.

**5. ¿Cómo evitás que un paciente vea los turnos de otro?**
En dos niveles. Primero, en el listado: `filtrosSegunRol()` **fuerza** el
filtro `dni_paciente` con el DNI que viene **dentro del token**, así que
aunque mande otro DNI por la URL se lo sobreescribo. Segundo, en el acceso
a un turno puntual: `verificarQueElTurnoLeCorresponde()` compara el dueño
del turno con el usuario del token y devuelve **403** si no coincide. Lo
puedo demostrar con Thunder Client pidiendo un turno ajeno.

**6. ¿Qué pasa si edito el token para ponerme ADMIN?**
`jwt.verify()` falla, porque la firma no coincide con los datos nuevos, y
devuelve 401. No puedo recalcular la firma porque la clave `JWT_SECRET`
está solo en el servidor, en el `.env`.

**7. ¿Dónde guardás las contraseñas?**
En la columna `password_hash` de la tabla `usuario`, hasheadas con bcrypt.
La contraseña original no se guarda en ninguna parte y no se puede recuperar
del hash. Para el login, bcrypt vuelve a hashear lo que escribió el usuario
y compara.

**8. ¿Diferencia entre 401 y 403?**
401 es "no sé quién sos": falta el token o venció. 403 es "sé quién sos pero
no te corresponde": el token está bien pero el rol o el dueño del dato no te
habilitan.

**9. ¿Qué es CORS y por qué lo necesitás?**
El navegador bloquea los pedidos entre orígenes distintos. El frontend está
en el puerto 5500 y el backend en el 3000, así que son orígenes distintos.
CORS son las cabeceras con las que el servidor autoriza esos orígenes. Lo
aplica el navegador: en Thunder Client el pedido funciona igual sin CORS.

**10. ¿Qué es un middleware y en qué orden corren?**
Una función que se ejecuta entre el pedido y el controlador, con
`(req, res, next)`. O llama a `next()` y sigue, o responde y corta. Corren en
el orden en que están escritos en `server.js` y en el router: CORS,
`express.json()`, `verificarToken`, `autorizarRol`, el validator, el
controlador, y el manejador de errores al final.

**11. ¿Por qué usás una transacción al crear un turno?**
Porque son dos INSERT (en `turno` y en `registra`). Si el segundo falla,
quedaría un turno sin médico asignado. Con la transacción, o se guardan los
dos o se deshace todo con `rollback`.

**12. ¿Cómo detectás que dos turnos se superponen?**
Comparando los intervalos: se pisan si el nuevo empieza antes de que termine
el viejo y el viejo empieza antes de que termine el nuevo. El fin no está
guardado, lo calculo sumando la `duracion` de la práctica con `ADDTIME`.
Ignoro los cancelados y excluyo el propio turno cuando reprogramo.

**13. ¿Qué pasa si reprogramo un turno que estaba confirmado?**
Vuelve a estado `Reservado` y se limpian `fecha_confirmacion` y
`medico_confirmacion`, porque hay que confirmarlo de nuevo. Es la regla del
enunciado del trabajo práctico.

**14. ¿Por qué cancelar no borra el turno?**
Porque el centro necesita el historial. Cancelar cambia el estado; como las
validaciones de superposición ignoran los turnos cancelados, el horario
queda libre igual. El `DELETE` existe pero es solo para el administrador.

**15. ¿Qué es Node?**
El entorno que permite ejecutar JavaScript fuera del navegador, usando el
motor V8 de Chrome. No es un lenguaje ni un framework: es el *runtime*. Es lo
que permite que el backend esté escrito en JavaScript.

---

## 16. Guión de la demostración

Probado en este orden, muestra todo lo que pide la consigna en unos
10 minutos. **Tené el navegador en una ventana y la consola del servidor
visible en otra.**

### A. Perfiles y sesiones (punto B obligatorio)

1. Entrar como **paciente**. Mostrar arriba el nombre y el perfil
   (*identificar al usuario que inició sesión*).
2. Señalar el menú: no aparecen "Pacientes" ni acciones de admin
   (*frontend adaptado al perfil*).
3. Ir a **Turnos**: solo aparecen los turnos de ese DNI.
4. Cerrar sesión y entrar como **médico**: aparecen solo los turnos de su
   agenda, no hay "Nuevo turno".
5. Entrar como **admin**: ve todos los turnos y todos los botones.

### B. Que los permisos no se puedan saltear

6. Con sesión de paciente, **pegar en la URL** `pages/dashboard.html` sin
   token (o cerrar sesión y volver atrás): redirige al login.
7. **Con Thunder Client**, lo más contundente:
   - `GET http://localhost:3000/api/turnos` **sin** la cabecera
     Authorization → **401**;
   - login como paciente, copiar el token, y pedir un turno **de otro
     paciente**: `GET /api/turnos/1` → **403 "Ese turno no te pertenece"**;
   - con el token del paciente, `DELETE /api/turnos/1` → **403 "No tenés
     permisos"**.

### C. Las validaciones (punto A)

Como **paciente**, en "Nuevo turno" (cada caso tiene que mostrar el cartel
rojo con el mensaje):

8. Fecha de **ayer** → *"No se pueden asignar turnos en fechas anteriores a hoy"*.
9. Un **domingo** → *"El centro no atiende los domingos"*.
10. Un día que el médico no atiende → *"El médico atiende los Lunes y elegiste un Martes"*.
11. Una hora fuera de la franja del médico → *"El médico atiende de 08:00 a 12:00 ese día"*.
12. Un turno válido → **se crea** (anotá el número).
13. **Otro turno a la misma hora** → *"El paciente ya tiene otro turno a las ..."* (409).
14. Entrar como **admin** y cargar un turno para **otro paciente** con el
    **mismo médico y la misma hora** → *"El médico ya tiene un turno a las ..."* (409).

### D. La mejora funcional

15. **Reprogramar** el turno que creaste: cambiar fecha y hora → se guarda y
    el estado vuelve a `Reservado`.
16. Intentar reprogramarlo a una hora ocupada → error (se ven las
    validaciones corriendo de nuevo).
17. Como **admin** o **médico**, **Confirmar** el turno → pasa a `Confirmado`.
18. Reprogramarlo otra vez → vuelve a `Reservado` (la regla del enunciado).
19. **Cancelar** el turno → pasa a `Cancelado`.
20. Intentar **confirmarlo** ahora → *"El turno está cancelado y no se puede
    confirmar"* (409).
21. Mostrar los **filtros**: por estado `Cancelado`, por rango de fechas.

### E. Que se refleja en la base de datos

22. En MySQL Workbench:
    ```sql
    SELECT id_turno, fecha_turno, hora_turno, estado_turno
    FROM turno ORDER BY id_turno DESC LIMIT 5;
    ```
    Se ve el turno nuevo con su estado actualizado.
23. Mostrar también la fila en `registra` con el médico asignado:
    ```sql
    SELECT * FROM registra ORDER BY id_turno DESC LIMIT 5;
    ```

### F. Cierre

24. Mostrar la consola del servidor con los pedidos que fueron llegando
    (`-> PATCH /api/turnos/45/cancelar`) y contar **el recorrido completo
    de un pedido** (sección 7 de esta guía). Ese es el mejor cierre posible.
