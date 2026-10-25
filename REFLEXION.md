# Reflexión · CI/CD con Firebase Hosting y hardening de headers


### Antes (B1): producción sin cambios

Calificación en securityheaders.com: **D**

```
▶ HTML: https://fir-hosting-tp-eb401.web.app/
  status 200
  ✘ content-security-policy (falta)
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✘ x-content-type-options (falta)
  ✘ x-frame-options (falta)
  ✘ referrer-policy (falta)
  ✘ permissions-policy (falta)
  · cache-control: max-age=3600

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✘ cache-control: max-age=3600

❌ 6 control(es) pendiente(s).
```

### Después (B5): producción con los cambios mergeados

Calificación en securityheaders.com: **A**

```
▶ HTML: https://fir-hosting-tp-eb401.web.app/
  status 200
  ✔ content-security-policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self'; img-src 'self' https://picsum.photos …
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✔ x-content-type-options: nosniff
  ✔ x-frame-options: DENY
  ✔ referrer-policy: strict-origin-when-cross-origin
  ✔ permissions-policy: camera=(), microphone=(), geolocation=()
  · cache-control: public, max-age=0, must-revalidate

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✔ cache-control: public, max-age=31536000, immutable

✅ Todos los controles pasaron.
## Preguntas de reflexión

### 1. ¿Por qué el preview tiene vencimiento? ¿Qué riesgos tendría uno que no vence nunca?

Cada preview es una copia completa del sitio, accesible públicamente por quien
tenga la URL, con la versión de un PR que todavía no fue revisada. El
vencimiento (`expires: 3d`) hace que esas copias se limpien solas. Si no
vencieran, se acumularían versiones viejas y sin mantenimiento (posiblemente
con bugs, vulnerabilidades o contenido de prueba) expuestas indefinidamente, y
se consumiría cuota del plan Spark. También podría quedar accesible un cambio
que se decidió no publicar.

### 2. ¿Qué diferencia hay entre desplegar y liberar? ¿Dónde aparece cada concepto?

**Desplegar** es subir una versión a la infraestructura; **liberar** es decidir
qué versión ven los usuarios. En este flujo:

- El workflow de PR *despliega* a un preview channel, pero no *libera*: los
  usuarios siguen viendo producción.
- El merge a `main` despliega y libera en el mismo paso (canal `live`).
- El rollback desde la consola y `hosting:clone` solo cambian qué versión está
  liberada: no recompilan nada, reutilizan versiones ya desplegadas. Por eso el
  rollback fue instantáneo.

### 3. ¿Por qué la clave de la service account va en un secreto y no en el repo? ¿Qué podría hacer quien la obtenga? ¿Cómo limitarías el daño?

Es una credencial con permiso para desplegar. Mi repo es público, así que si la
clave estuviera en el código cualquiera podría usarla. Quien la obtenga podría
publicar contenido arbitrario en el sitio (reemplazarlo, hacer phishing bajo mi
dominio) y usar cualquier otro permiso que tenga la cuenta. Un secreto de GitHub
no se muestra en los logs ni se expone en PRs de forks.

Para limitar el daño:
- Dar a la service account solo el rol mínimo necesario (el rol de Hosting, no
  Propietario). 
- Rotar la clave periódicamente y revocarla si hay sospecha de filtración.
- Exigir aprobación manual en el Environment `production` antes de desplegar.

### 4. La CSP rompió la app: ¿por qué? ¿Qué opciones evaluaste y qué costo tiene cada una?

Con `script-src 'self'`, el navegador solo ejecuta scripts cargados desde
archivos del propio sitio. Next.js, incluso con export estático, incrusta
scripts inline en el HTML (en `/cursos` aparecieron tres bloqueados) para
hidratar la página, y sin ellos el filtro no respondía. La consola indicaba
cada bloqueo con la directiva violada.

Opciones evaluadas:
`'unsafe-inline'` en `script-src`: Permite cualquier script inline, incluido uno inyectado por un atacante, así que anula buena parte de la protección contra XSS. Es lo que usé. 
Hashes `'sha256-...'`  Permiten solo los scripts exactos. Casi sin costo de seguridad, pero cambian con cada build y por página, así que hay que calcularlos automáticamente en el pipeline después de `npm run build`. 

Usé `'unsafe-inline'` en `script-src`, aceptando ese costo, y dejé el
resto de la política restrictiva (`object-src 'none'`, `base-uri 'self'`,
`frame-ancestors 'none'`). La imagen de portada viene de `picsum.photos`, que
responde con un 302 hacia otro dominio (lo comprobé en Network).
Como la CSP se aplica a cada salto de la redirección, permití los dos orígenes
exactos en `img-src` (`https://picsum.photos` y `https://fastly.picsum.photos`),
sin comodines.

### 5. Para cada header de B2, ¿qué ataque mitiga?

- **Strict-Transport-Security:** ataques de *downgrade* a HTTP y
  man-in-the-middle; el navegador usa HTTPS en visitas futuras durante el
  `max-age`.
- **X-Content-Type-Options: nosniff:** *MIME sniffing*; evita que el navegador
  interprete como script o estilo un archivo servido con otro tipo.
- **X-Frame-Options: DENY:** *clickjacking*; impide que otro sitio embeba el
  mío en un iframe para engañar al usuario con clics ocultos.
- **Referrer-Policy: strict-origin-when-cross-origin:** fuga de información;
  hacia otros sitios solo se envía el origen, no rutas ni parámetros de la URL.
- **Permissions-Policy:** abuso de APIs del navegador (cámara, micrófono) por código inyectado o de terceros; la app no las necesita,así que las deshabilité.

### 6. ¿Por qué los assets de `/_next/static/` pueden cachearse un año y el HTML no?

Los archivos de `/_next/static/` tienen el hash del contenido en el nombre: si
cambia el contenido, cambia la URL. Por eso una URL dada nunca va a servir
contenido distinto y puede cachearse un año como `immutable`. El HTML siempre
tiene la misma URL (`/`, `/cursos`), así que si se cacheara sin revalidar, los
usuarios seguirían viendo la versión anterior después de un deploy, y también
después de un rollback. 

### 7. Si la app necesitara SSR, ¿qué cambiaría en la arquitectura, la CSP y el costo?

- **Arquitectura:** Hosting en plan Spark solo sirve estáticos. Con SSR, Server
  Actions o rutas dinámicas habría que usar App Hosting, que corre sobre Cloud
  Run, y el pipeline pasaría de publicar una carpeta `out/` a desplegar un
  servicio.
- **CSP:** al haber servidor, se pueden generar *nonces* por request, así que
  la política podría prescindir de `'unsafe-inline'` sin tener que calcular
  hashes en el build.
- **Costo:** App Hosting exige el plan Blaze con cuenta de facturación, así que
  el costo deja de ser cero (se paga por uso de Cloud Run). Además aparecen
  nuevas superficies de ataque (inyecciones del lado servidor, secretos de
  runtime).

---

## Trazabilidad y rollback (A6)

- El footer muestra el SHA, la rama y la fecha de build. En el preview de mi
  primer PR mostró el SHA `be7bc07` y la rama `feat/pipeline`; en producción
  muestra el SHA de `main`.
- Hice el rollback desde la consola (Hosting → Release history → Rollback) y
  verifiqué en el footer que volvió el SHA anterior.
- Promoví un preview a producción sin recompilar con
  `npx firebase hosting:clone fir-hosting-tp-eb401:pr2-feat-titulo fir-hosting-tp-eb401:live`.
  Como `clone` copia el mismo build, el footer mostró la rama `feat/titulo`, no
  `main`: el footer indica qué commit se compiló, no qué hay en `main`.
- **Estado de `main` tras el rollback:** `main` sigue teniendo el código nuevo,
  así que el próximo merge volvería a desplegar lo que había revertido.
