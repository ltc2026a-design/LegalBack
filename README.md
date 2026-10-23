# LegalBack — API del Sistema de Consulta Legal (Render)

Repositorio publicado, generado desde el monorepo de desarrollo con:

```bash
npm run preparar:repos -- back
```

La fuente de verdad es `C:\Users\harol\Desktop\App de asistencia de trabajo`. No edites código aquí.

Express 5 + Prisma 7 + PostgreSQL. Sirve la API en `/api` y, si le apuntas las carpetas
`dist` de los frentes, también los sirve en el mismo origen (variable `FRENTE_CONSULTA_DIR`
/ `FRENTE_ADMIN_DIR`). En el despliegue normal los frentes viven en Vercel y aquí solo corre la API.

## Publicar en Render

1. Crea el repositorio `LegalBack` y sube este código (ver abajo "Después de cada cambio").
2. Render → **New → Blueprint**, conecta el repo y escoge `render.yaml`.
3. Render pregunta por las variables marcadas `sync: false`: pega la **URI de Supabase** en
   `DATABASE_URL` y los orígenes de Vercel en `ORIGINES_PERMITIDAS`
   (`https://*.vercel.app` basta para pocas personas); deja vacías las de IA/push si aún no las usas.
4. El blueprint ya no crea base de datos: vive en Supabase (ver la sección de abajo).
5. Al terminar el build revisa `https://<tu-servicio>.onrender.com/api/health` →
   `{"ok":true,...}`.
6. En Vercel, pon `VITE_API_URL = https://<tu-servicio>.onrender.com/api` y redepliega.

### Si creaste el servicio a mano (sin Blueprint)

Render dejó el *Build Command* en `npm install`, que solo baja dependencias: no genera el
cliente de Prisma ni compila, y por eso revienta al arrancar con
`Cannot find module '/opt/render/project/src/dist/server.js'`. En **Settings → Build & Test**
(deja guardado con Save y luego *Manual Deploy*) pon:

| Campo | Valor |
| --- | --- |
| Build command | `NODE_ENV=development npm ci && npx prisma generate && npx prisma migrate deploy && npm run build` |
| Start command | `node dist/server.js` |
| Health check path | `/api/health` |

El `NODE_ENV=development` del inicio **no es decorativo**: Render arranca con `NODE_ENV=production`
y npm omite entonces las devDependencies, donde viven los `@types/*` de express, cors, jsonwebtoken,
web-push y node-cron. Sin esas definiciones `tsc` no compila y nunca nace `dist/` (comprobado
instalando así en limpio: `tsc` y `prisma` sobreviven solo por transitividad de `@prisma/client`,
pero los tipos no).

Y en **Environment** hacen falta mínimo estas, porque `server.ts` se niega a arrancar sin ellas:
`DATABASE_URL` (la URI de Supabase), `JWT_SECRET` (una cadena larga al azar),
`ORIGINES_PERMITIDAS=https://*.vercel.app`, `TRUST_PROXY=1`, `PG_POOL_MAX=10`.

## La base de datos: Supabase

Supabase → **Connect** (arriba a la derecha) → pestaña *Database* → **Connection string** →
*URI*, y elige el **pooler de sesión (puerto 5432)**. El usuario es `postgres.<ref>`:

```
postgresql://postgres.djdvfpqikvtttliidpuh:<CONTRASENA>@aws-0-us-east-1.pooler.supabase.com:5432/postgres?sslmode=no-verify
```

`sslmode=no-verify` y no `require`: el driver `pg` de Node trata `require` como `verify-full` y
la cadena de certificados del pooler no es verificable en público (`SELF_SIGNED_CERT_IN_CHAIN`).
La **conexión directa** (`db.<ref>.supabase.co`) es solo IPv6 y desde una casa con IPv4 no
arranca sin el add-on de pago, así que no la uses. El **pooler de transacciones (6543)** hace
pgbouncer y rompe las sentencias preparadas de las migraciones. La `sb_publishable_...` y la
`anon` que enseña Supabase son para su API REST: **Prisma no las usa**.

Esa cadena va en dos sitios y en **ninguno se sube a git**: el `DATABASE_URL` del servicio en
Render, y `backend/.env` local solo cuando toque migrar o volcar.

**Ya aplicado (2026-10-23):** las 4 migraciones están corriendo en Supabase y el contenido
volcado (14.838 filas: 9 libros, 6.824 artículos). Cuando cambie el esquema otra vez, el orden
es este — `replicar` **no crea tablas**, lee las que existen, las vacía con
`TRUNCATE ... CASCADE` e inserta:

```bash
cd backend
DATABASE_URL="<uri>" npx prisma migrate deploy
npm run replicar -- --destino="<uri>" --confirmar=sobrescribir
```

Notas del plan free de Supabase: la base **no se borra** a los 90 días (a diferencia de la de
Render), pero el proyecto **se pausa** si nadie lo toca en una semana y despierta con la primera
conexión — la primera llamada después de la pausa puede fallar una vez (`P1001`) y ya. El pooler
admite 60 conexiones; con `PG_POOL_MAX=10` por proceso sobra.

## Cómo llegan los libros nuevos o actualizados

La base de datos es **una sola** y todos los frentes leen de ella, así que un contenido
nuevo se ve en la web y en el APK sin publicar nada otra vez. Tres caminos:

**a) Desde el panel de administración** (lo normal): *Cambios pendientes → Aprobar*.
El conector vuelve a leer la fuente oficial y materializa el artículo.

**b) Desde tu computadora, volcando la base local a Supabase:**

```bash
# en el monorepo, con el PostgreSQL local corriendo (npm run db)
npm run replicar -- --destino="postgresql://...pooler.supabase.com:5432/postgres?sslmode=require" --confirmar=sobrescribir
```

`replicar` copia las 13 tablas en su orden de claves foráneas, respeta el árbol
autorreferente de `seccion`, omite columnas generadas (`texto_ts`) y verifica los conteos.
Es **destructivo en el destino** por eso exige `--confirmar=sobrescribir`.
Nota: si Render bloquea conexiones externas, corre el mismo comando desde el entorno de Render (Shell).

**c) El worker de sincronización** (`npm run worker`) revisa las fuentes cada hora con cron.
En el plan free el servicio se duermen, así que el cron no corre de forma confiable: o lo
activas en un servicio que no duerma, o te quedas con (a) y (b).

## Comandos

| Comando | Para qué |
| --- | --- |
| `npm run build` | `tsc` → `dist/` |
| `npm start` | corre `dist/server.js` |
| `npm run db:deploy` | aplica migraciones Prisma |
| `npm run replicar -- --destino=... --confirmar=sobrescribir` | vuelca la base local |
| `npm run sync` | corre los conectores una vez |
| `npm run importar:md` | libros federales desde el espejo Lex MX |
| `npm run sync:qroo` | descarga los códigos de Quintana Roo |
| `npm run admin:create` / `usuario:create` | **única** forma de crear cuentas |
| `npm test` | 159 pruebas del backend |

## Cuentas

No existe el autorregistro. Ni en la app ni en el panel: las credenciales se crean con
`npm run usuario:create` / `npm run admin:create` (o desde el Shell de Render). Solo quien
tiene acceso al servidor puede dar de alta a una persona.
