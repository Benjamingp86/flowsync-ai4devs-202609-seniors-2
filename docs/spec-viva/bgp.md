# Cuentas y acceso

## Purpose

Permitir que una persona cree una cuenta en FlowSync, inicie y cierre sesión, mantenga su sesión entre recargas y consulte su perfil, tanto a través de la API como desde la interfaz web.

## Requirements

### Requirement: Registro de cuenta por API

El sistema SHALL crear una cuenta nueva cuando recibe en `POST /api/v1/auth/signup` un nombre completo, un email, una contraseña y su confirmación válidos, y SHALL dejar a la persona autenticada en la misma respuesta.

#### Scenario: Registro correcto

- **WHEN** se envía `fullName`, `email`, `password` y `passwordConfirmation` válidos a `POST /api/v1/auth/signup`
- **THEN** la respuesta es `200` con `{ "data": { "user": {...}, "token": "..." } }`, donde `user` contiene `id`, `fullName`, `email`, `initials`, `createdAt` y `updatedAt`, y `token` es un token de acceso utilizable de inmediato

#### Scenario: La contraseña nunca se devuelve

- **WHEN** el registro tiene éxito
- **THEN** la respuesta no incluye la contraseña ni ningún derivado de ella

### Requirement: Nombre completo opcional pero obligatorio en la petición

El sistema SHALL aceptar `fullName` con valor nulo, pero SHALL exigir que la clave `fullName` venga en la petición de registro.

#### Scenario: Nombre nulo

- **WHEN** se registra una cuenta con `"fullName": null`
- **THEN** la cuenta se crea y el usuario devuelto tiene `fullName: null`

#### Scenario: Clave ausente

- **WHEN** se registra una cuenta sin incluir la clave `fullName`
- **THEN** la respuesta es `422` con un error de regla `required` sobre el campo `fullName`

### Requirement: Validación del email en el registro

El sistema SHALL rechazar con `422` un registro cuyo email no tenga formato válido, supere los 254 caracteres o ya pertenezca a otra cuenta.

#### Scenario: Email ya registrado

- **WHEN** se intenta registrar un email idéntico al de una cuenta existente
- **THEN** la respuesta es `422` con `{ "errors": [{ "rule": "database.unique", "field": "email", ... }] }` y no se crea la cuenta

#### Scenario: Mismo email con distintas mayúsculas

- **WHEN** se registra un email que solo se diferencia de uno existente en mayúsculas y minúsculas
- **THEN** la cuenta se crea como una cuenta distinta, con el email tal como se escribió

#### Scenario: Email con formato inválido

- **WHEN** se registra con un email sin formato de dirección de correo
- **THEN** la respuesta es `422` con un error de regla `email` sobre el campo `email`

### Requirement: Validación de la contraseña en el registro

El sistema SHALL exigir que la contraseña y su confirmación tengan entre 8 y 32 caracteres y que ambas coincidan.

#### Scenario: Contraseña demasiado corta

- **WHEN** se registra con una contraseña y una confirmación de menos de 8 caracteres
- **THEN** la respuesta es `422` con un error `minLength` (con `meta.min = 8`) para `password` y otro para `passwordConfirmation`

#### Scenario: Contraseña demasiado larga

- **WHEN** se registra con una contraseña de más de 32 caracteres
- **THEN** la respuesta es `422` con un error `maxLength` sobre `password`

#### Scenario: Confirmación distinta

- **WHEN** la contraseña y la confirmación tienen longitud válida pero no coinciden
- **THEN** la respuesta es `422` con un error de regla `sameAs` sobre `passwordConfirmation`

### Requirement: Inicio de sesión por API

El sistema SHALL emitir un token de acceso nuevo cuando recibe en `POST /api/v1/auth/login` un email y una contraseña que corresponden a una cuenta.

#### Scenario: Credenciales correctas

- **WHEN** se envía un email y una contraseña correctos a `POST /api/v1/auth/login`
- **THEN** la respuesta es `200` con `{ "data": { "user": {...}, "token": "..." } }`

#### Scenario: Varios inicios de sesión

- **WHEN** una misma cuenta inicia sesión más de una vez
- **THEN** cada inicio de sesión devuelve un token distinto y los tokens anteriores siguen siendo válidos

#### Scenario: Email con distintas mayúsculas

- **WHEN** se inicia sesión con un email que coincide exactamente, mayúsculas incluidas, con una cuenta
- **THEN** se entra en esa cuenta concreta, y no en otra cuyo email solo difiera en mayúsculas

### Requirement: Credenciales incorrectas indistinguibles

El sistema SHALL responder igual cuando el email no existe y cuando la contraseña es incorrecta.

#### Scenario: Contraseña incorrecta

- **WHEN** se inicia sesión con un email existente y una contraseña incorrecta
- **THEN** la respuesta es `400` con `{ "errors": [{ "message": "Invalid user credentials" }] }`, sin campo asociado

#### Scenario: Email inexistente

- **WHEN** se inicia sesión con un email que no pertenece a ninguna cuenta
- **THEN** la respuesta es idéntica a la de contraseña incorrecta: `400` con el mismo mensaje

### Requirement: Validación de formato en el inicio de sesión

El sistema SHALL rechazar con `422` un inicio de sesión cuyo email no tenga formato válido, sin comprobar las credenciales.

#### Scenario: Email mal formado

- **WHEN** se envía a `POST /api/v1/auth/login` un email sin formato de dirección de correo
- **THEN** la respuesta es `422` con un error de regla `email` sobre el campo `email`

### Requirement: Consulta del perfil propio

El sistema SHALL devolver los datos de la cuenta autenticada en `GET /api/v1/account/profile` cuando la petición lleva un token válido en la cabecera `Authorization: Bearer`.

#### Scenario: Token válido

- **WHEN** se pide el perfil con un token válido
- **THEN** la respuesta es `200` con `{ "data": { "id", "fullName", "email", "initials", "createdAt", "updatedAt" } }` de la cuenta dueña del token

### Requirement: Acceso protegido a la cuenta

El sistema SHALL responder `401` a cualquier petición a los endpoints de cuenta (`/api/v1/account/...`) que no lleve un token válido.

#### Scenario: Sin token

- **WHEN** se pide el perfil o se cierra sesión sin cabecera `Authorization`
- **THEN** la respuesta es `401` con `{ "errors": [{ "message": "Unauthorized access" }] }`

#### Scenario: Token revocado

- **WHEN** se pide el perfil con un token que ya se usó para cerrar sesión
- **THEN** la respuesta es `401`

### Requirement: Cierre de sesión por API

El sistema SHALL invalidar el token con el que se hace la petición a `POST /api/v1/account/logout`, y solo ese.

#### Scenario: Cierre correcto

- **WHEN** se cierra sesión con un token válido
- **THEN** la respuesta es `200` con `{ "message": "Logged out successfully" }` y ese token deja de ser aceptado

#### Scenario: Otras sesiones abiertas

- **WHEN** la cuenta tiene varios tokens y cierra sesión con uno de ellos
- **THEN** los demás tokens siguen siendo válidos

### Requirement: Vigencia de los tokens

El sistema SHALL mantener válido un token de acceso hasta que se use para cerrar sesión, sin caducidad por tiempo.

#### Scenario: Token antiguo

- **WHEN** se usa un token que nunca se usó para cerrar sesión, sea cual sea su antigüedad
- **THEN** el sistema lo acepta

### Requirement: Iniciales del usuario

El sistema SHALL calcular y devolver unas iniciales en mayúsculas para cada cuenta.

#### Scenario: Nombre con dos o más palabras

- **WHEN** el nombre completo tiene al menos dos palabras separadas por espacio
- **THEN** las iniciales son la primera letra de la primera palabra y la primera letra de la segunda

#### Scenario: Nombre de una sola palabra

- **WHEN** el nombre completo es una sola palabra, por ejemplo "Ada"
- **THEN** las iniciales son sus dos primeras letras ("AD")

#### Scenario: Sin nombre

- **WHEN** la cuenta no tiene nombre completo, por ejemplo con email `prueba@flowsync.test`
- **THEN** las iniciales son la primera letra de lo que va antes de la arroba y la primera letra de lo que va después ("PF")

### Requirement: Forma de las respuestas de error de la API

El sistema SHALL devolver los errores como `{ "errors": [ { "message", "rule"?, "field"?, "meta"? } ] }`, con `rule` y `field` presentes en los errores de validación.

#### Scenario: Error de validación

- **WHEN** una petición de registro o inicio de sesión no supera la validación
- **THEN** la respuesta es `422` con un elemento en `errors` por cada campo que falla, cada uno con su `rule` y su `field`

### Requirement: Pantalla de registro

La interfaz SHALL ofrecer una pantalla "Crea tu cuenta" con los campos Nombre completo (opcional), Email, Contraseña y Repite la contraseña, y SHALL llevar al perfil al completarse el registro.

#### Scenario: Registro correcto desde la pantalla

- **WHEN** una persona sin sesión rellena el formulario con datos válidos y pulsa "Crear cuenta"
- **THEN** el botón muestra "Creando cuenta…" mientras se envía y, al terminar, la persona queda con la sesión iniciada y ve su perfil

#### Scenario: Contraseñas distintas

- **WHEN** la contraseña y su repetición no coinciden y se pulsa "Crear cuenta"
- **THEN** aparece "Las contraseñas no coinciden." bajo "Repite la contraseña" sin que se envíe nada al servidor

#### Scenario: Email ya registrado

- **WHEN** se intenta registrar un email que ya existe
- **THEN** aparece "Ese email ya está registrado. Inicia sesión en su lugar." bajo el campo Email

#### Scenario: Ayuda de la contraseña

- **WHEN** el campo Contraseña no tiene ningún error
- **THEN** debajo se muestra la pista "Entre 8 y 32 caracteres."

#### Scenario: Enlace al inicio de sesión

- **WHEN** la persona está en la pantalla de registro
- **THEN** ve "¿Ya tienes cuenta? Inicia sesión", que la lleva a la pantalla de inicio de sesión

### Requirement: Pantalla de inicio de sesión

La interfaz SHALL ofrecer una pantalla "Inicia sesión" con los campos Email y Contraseña, y SHALL llevar al perfil al iniciar sesión correctamente.

#### Scenario: Inicio de sesión correcto

- **WHEN** una persona sin sesión introduce credenciales válidas y pulsa "Entrar"
- **THEN** el botón muestra "Entrando…" mientras se envía y, al terminar, ve su perfil

#### Scenario: Credenciales incorrectas

- **WHEN** introduce un email o una contraseña que no corresponden a ninguna cuenta
- **THEN** aparece arriba del formulario el aviso "El email o la contraseña no son correctos."

#### Scenario: Enlace al registro

- **WHEN** la persona está en la pantalla de inicio de sesión
- **THEN** ve "¿Aún no tienes cuenta? Crea una", que la lleva a la pantalla de registro

### Requirement: Mensajes de error en castellano

La interfaz SHALL mostrar los errores de validación traducidos al castellano, bajo el campo al que se refieren, y un aviso general cuando el error no corresponde a ningún campo visible.

#### Scenario: Errores de campo

- **WHEN** el servidor rechaza un campo por formato de email, por ausencia, por longitud mínima o por longitud máxima
- **THEN** bajo ese campo aparece, respectivamente, "Introduce una dirección de email válida.", "Falta rellenar …", "… debe tener al menos N caracteres." o "… no puede superar los N caracteres."

#### Scenario: Servidor inaccesible

- **WHEN** se envía un formulario y el servidor no responde
- **THEN** aparece el aviso "No se pudo conectar con el servidor. Comprueba que el backend está arrancado."

#### Scenario: Error inesperado del servidor

- **WHEN** el servidor responde con un error que no es de validación, credenciales ni autenticación
- **THEN** aparece el aviso "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento."

### Requirement: Sesión persistente entre recargas

La interfaz SHALL conservar la sesión al recargar la página o volver a abrir el navegador, comprobándola contra el servidor antes de mostrar contenido protegido.

#### Scenario: Recarga con sesión válida

- **WHEN** una persona con sesión iniciada recarga la página
- **THEN** ve un indicador de carga y después sigue en su perfil sin volver a introducir credenciales

#### Scenario: Sesión rechazada por el servidor

- **WHEN** se carga la aplicación con una sesión guardada que el servidor ya no acepta
- **THEN** se descarta la sesión guardada y la persona ve la pantalla de inicio de sesión con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión."

#### Scenario: Servidor caído al recargar

- **WHEN** se carga la aplicación con una sesión guardada y el servidor no responde
- **THEN** la persona ve la pantalla de inicio de sesión con el aviso de que no se pudo conectar, y la sesión guardada se conserva para recuperarse al recargar cuando el servidor vuelva

### Requirement: Protección de pantallas

La interfaz SHALL mostrar el perfil solo a personas con sesión y SHALL mostrar las pantallas de inicio de sesión y registro solo a personas sin sesión.

#### Scenario: Perfil sin sesión

- **WHEN** una persona sin sesión abre `/profile`
- **THEN** es redirigida a `/login`

#### Scenario: Login o registro con sesión

- **WHEN** una persona con sesión abre `/login` o `/register`
- **THEN** es redirigida a `/profile`

#### Scenario: Dirección desconocida

- **WHEN** alguien abre cualquier otra dirección de la aplicación
- **THEN** es redirigido a `/profile`, y desde ahí a `/login` si no tiene sesión

### Requirement: Pantalla de perfil

La interfaz SHALL mostrar a la persona con sesión sus iniciales, su nombre, su email y la fecha en que se creó su cuenta.

#### Scenario: Perfil con nombre

- **WHEN** una persona con nombre completo abre su perfil
- **THEN** ve un círculo con sus iniciales, su nombre, su email y "Miembro desde" con la fecha en formato largo en castellano (por ejemplo "30 de septiembre de 2026")

#### Scenario: Perfil sin nombre

- **WHEN** una persona registrada sin nombre completo abre su perfil
- **THEN** en lugar del nombre ve "Sin nombre"

### Requirement: Cierre de sesión desde la interfaz

La interfaz SHALL permitir cerrar sesión desde el perfil y SHALL dejar a la persona sin sesión aunque el servidor no confirme el cierre.

#### Scenario: Cerrar sesión

- **WHEN** la persona pulsa "Cerrar sesión" en su perfil
- **THEN** el botón muestra "Cerrando sesión…", la persona termina en la pantalla de inicio de sesión y recargar no la devuelve al perfil

#### Scenario: Cierre con el servidor caído

- **WHEN** la persona cierra sesión y el servidor no responde
- **THEN** igualmente termina en la pantalla de inicio de sesión sin sesión en el navegador

---

## Parte B

### 1. Requisitos escritos y comprobados

- Requisitos escritos por el agente: 20
- Requisitos comprobados por mí abriendo el código: 7

### 2. Incoherencias que aparecieron al escribirla

- **"Un email, una cuenta" se rompe con las mayúsculas.** El registro rechaza un email repetido, pero acepta la misma dirección escrita con otras mayúsculas (`probe@flowsync.test` y `PROBE@FLOWSYNC.TEST` son dos cuentas distintas), y el login entra en una u otra según cómo se escriba. Se ve en el registro y en el inicio de sesión por API.
- **Todas las respuestas correctas van dentro de `data`, menos la del cierre de sesión.** Registro, inicio de sesión y perfil responden `{ "data": ... }`; el cierre de sesión responde `{ "message": "Logged out successfully" }` sin envoltorio. Se ve en `POST /api/v1/account/logout`.
- **Los mensajes de error empiezan con mayúscula, menos los de longitud.** "Introduce una dirección de email válida.", "Falta rellenar el email." o "Las contraseñas no coinciden." empiezan en mayúscula, pero el de longitud sale "la contraseña debe tener al menos 8 caracteres." (y el de máximo igual). Se ve en la pantalla de registro, bajo el campo Contraseña.

### 3. Lo que no supe decidir si era bug o contrato

- **Login que oculta si el email existe vs. registro que lo revela.** El inicio de sesión responde exactamente igual si el email no existe que si la contraseña es incorrecta (incluso gasta tiempo en un hash ficticio para no delatarse), pero el registro con un email ya usado responde "Ese email ya está registrado". Lectura A: es un bug, porque la protección del login no sirve si el registro deja comprobar qué emails existen. Lectura B: es contrato, porque se priorizó avisar claramente en el registro y lo del login es solo el comportamiento por defecto de la librería, que nadie eligió. Leyendo el código no hay forma de saber cuál de las dos se decidió.
- **Tokens que nunca caducan vs. un mensaje de "sesión caducada".** No hay ninguna caducidad configurada (la columna de expiración existe pero queda vacía), y sin embargo la pantalla de login explica cualquier rechazo del token con "Tu sesión ha caducado. Vuelve a iniciar sesión." Lectura A: es un bug, se quiso que las sesiones caducaran (por eso existen la columna y el mensaje) y faltó configurarlo. Lectura B: es contrato, las sesiones son permanentes a propósito y "caducado" es solo una forma genérica de decir "tu token ya no vale". Además, que el token no caduque nunca no se puede confirmar leyendo: habría que esperar, así que no lo cuento como comprobado.
- **Iniciales sacadas del dominio del correo.** Sin nombre, la segunda inicial sale del dominio (`prueba@flowsync.test` → "PF"; todos los de Gmail llevarían una "G"), mientras que con un nombre de una sola palabra se usan sus dos primeras letras ("Ada" → "AD"). Lectura A: es contrato, alguien decidió que el email aportara dos letras distintas. Lectura B: es casualidad, se reutilizó la lógica del nombre partiendo el email por la arroba como si fuera "nombre apellido". Leyendo el código no se distingue una de otra.
