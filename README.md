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
3. Render pregunta por las variables marcadas `sync: false`: pega los orígenes de Vercel
   en `ORIGINES_PERMITIDAS` (`https://*.vercel.app` basta para pocas personas) y deja
   vacías las de IA/push si aún no las usas.
4. Crea el **PostgreSQL** del blueprint (Render lo levanta solo y asigna `DATABASE_URL`).
5. Al terminar el build revisa `https://<tu-servicio>.onrender.com/api/health` →
   `{"ok":true,...}`.
6. En Vercel, pon `VITE_API_URL = https://<tu-servicio>.onrender.com/api` y redepliega.

**El plan free de base de datos caduca a los 90 días** (Render la borra y avisa por correo).
Si el servicio se pausa, la primera llamada tarda ~50 s en arrancar. Para una decena de
usuarios basta; si se queda corto, el plan de pago del mismo Render o una VM gratuita de
siempre (Oracle Cloud Always Free / Google Cloud e2-micro) corren el mismo código sin cambios.

## Cómo llegan los libros nuevos o actualizados

La base de datos es **una sola** y todos los frentes leen de ella, así que un contenido
nuevo se ve en la web y en el APK sin publicar nada otra vez. Tres caminos:

**a) Desde el panel de administración** (lo normal): *Cambios pendientes → Aprobar*.
El conector vuelve a leer la fuente oficial y materializa el artículo.

**b) Desde tu computadora, volcando la base local a Render:**

```bash
# en el monorepo, con el PostgreSQL local corriendo (npm run db)
npm run replicar -- --destino="postgresql://USER:PASS@HOST:5432/legal?sslmode=require" --confirmar=sobrescribir
```

`replicar` recrea las 13 tablas con su orden de claves foráneas, respeta el árbol
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
