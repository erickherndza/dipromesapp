# CLAUDE.md — DIPROMES

> Archivo de contexto del proyecto para Claude Code (`cc`).
> Léelo antes de tocar cualquier archivo del proyecto.

---

## Descripción del proyecto

**DIPROMES** (antes MediTrack Pro) es un sistema de gestión de activos médicos y pacientes para **Dipromes Terapias VAC**, negocio de terapia de compresión/presoterapia en Santo Domingo, República Dominicana. El negocio coloca máquinas terapéuticas en pacientes en domicilio o centros médicos y registra cada colocación con su facturación.

El sistema reemplazó un flujo manual en Excel.

> ⚠️ **Este repositorio (`dipromesapp`) es un fork de trabajo** creado el 2026-09-15 a partir de `dipromes` para migrar el hosting de Render.com a **Banahosting** (plan Bana Professional Deluxe Unlimited SSD, cPanel + Python Selector 3.9.23). El repo original `dipromes` **no se toca** — sigue en producción en Render tal cual. Todos los cambios de esta migración (y desarrollo futuro) se hacen aquí.

**Estado actual:** Código idéntico a `dipromes` al momento del fork (commit `bb62a17`), en proceso de adaptación a cPanel/Passenger. Backend Flask + MySQL (ver nota abajo). Frontend HTML+CSS+Vanilla JS en un solo archivo `index.html`.

> ⚠️ **Cambio de motor de BD (2026-09-15):** el plan original de esta migración asumía PostgreSQL Databases en cPanel (igual que Render), pero el cPanel de Banahosting (plan Bana Professional Deluxe Unlimited SSD) **no ofrece PostgreSQL** — solo MySQL Databases/phpMyAdmin/MySQL Database Wizard. Por eso `dipromesapp` corre sobre **MySQL** (driver `PyMySQL`, pure-Python) mientras que `dipromes` (Render) sigue sobre **PostgreSQL** (driver `pg8000`) sin tocarse. El schema es 100% SQLAlchemy ORM (portable); solo se ajustaron 2 líneas de SQL crudo en `apply_migrations()` (`backend/app.py`) que eran específicas de Postgres.

**Repositorio de esta versión (Banahosting):** `https://github.com/erickherndza/dipromesapp`
**Repositorio original (Render, producción, intacto):** `https://github.com/erickherndza/dipromes` → `https://dipromes.onrender.com`

---

## Migración a Banahosting — plan y guía

### Por qué
Aprovechar el plan de hosting ya pagado (Bana Professional Deluxe Unlimited SSD) en vez de depender del free tier de Render (que duerme por inactividad), sin arriesgar la producción actual: se trabaja en este repo separado hasta validar que todo funciona igual.

### Diferencias clave vs. `dipromes` (Render)

| Aspecto | `dipromes` (Render) | `dipromesapp` (Banahosting) |
|---|---|---|
| Entry point | `wsgi.py` → Gunicorn (`gunicorn wsgi:app`) | `passenger_wsgi.py` → Passenger (variable `application`) |
| Python | 3.11.0 (`render.yaml`) | 3.9.23 (cPanel Python Selector) — sin sintaxis 3.10+ en el código, compatible |
| Config de deploy | `render.yaml` | No aplica — todo se configura manualmente en cPanel |
| Variables de entorno | Dashboard de Render | cPanel → Setup Python App → Environment variables |
| Base de datos | PostgreSQL gestionado por Render | **MySQL Databases en cPanel** (Banahosting no ofrece PostgreSQL — crear DB + usuario ahí) |
| Driver de BD | `pg8000` (pure Python) | `PyMySQL` (pure Python) |
| Servidor WSGI en requirements | Gunicorn (usado) | Gunicorn queda en `requirements.txt` pero **no se usa** (Passenger reemplaza su función) — no hace falta quitarlo, solo no correrlo |

`passenger_wsgi.py` ya existe en este repo (raíz), replica la misma lógica de init de BD (`db.create_all()`, `apply_migrations()`, `seed_if_empty()`, `emergency_reset()`) que `wsgi.py`, expuesta como `application`.

### Guía paso a paso — configurar en cPanel

1. **Crear la base de datos MySQL**
   - cPanel → *MySQL® Databases* (o *MySQL Database Wizard*) → crear base + usuario + asignar usuario a la base (todos los privilegios)
   - cPanel antepone el usuario de cPanel al nombre de la base y del usuario de BD (ej. `cpaneluser_dipromes`, `cpaneluser_dipro_admin`) — anotar los nombres exactos que asigna el panel
   - Anotar: nombre de base, usuario, password, host (normalmente `localhost`), puerto (`3306`)
   - Armar el `DATABASE_URL`: `mysql://usuario:password@localhost:3306/nombre_bd` (la app lo normaliza internamente a `mysql+pymysql://`)

2. **Crear la aplicación Python**
   - cPanel → *Setup Python App* → *Create Application*
   - Python version: `3.9.23`
   - Application root: carpeta donde subirás el código (ej. `dipromesapp`)
   - Application URL: dominio/subdominio a usar
   - Application startup file: `passenger_wsgi.py`
   - Application Entry point: `application`

3. **Configurar Environment variables** (mismo panel de Setup Python App)
   - `SECRET_KEY` — string largo y fijo, distinto al de Render
   - `DATABASE_URL` — el armado en el paso 1
   - `ALLOWED_ORIGIN` — dominio final en Banahosting
   - `ADMIN_PASS`, `DR1_PASS`, `FELI_PASS` — opcional, si quieres contraseñas iniciales distintas a los defaults del código (`dipromes2026`, `doctor123`, `Feli@2026`)
   - `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS` — opcional, para forgot-password

4. **Subir el código**
   - Vía Git (cPanel → *Git Version Control* → clonar `https://github.com/erickherndza/dipromesapp.git`) o File Manager
   - Asegurar que quedan en el root de la app: `index.html`, `passenger_wsgi.py`, `backend/`, `render.yaml` (no molesta, simplemente no se usa)

5. **Instalar dependencias**
   - Dentro de *Setup Python App*, abrir la terminal/virtualenv de la app (botón provisto por cPanel)
   - `pip install -r backend/requirements.txt`

6. **Reiniciar la app** desde *Setup Python App* (botón *Restart*) para que tome las env vars y dependencias nuevas

7. **Verificar**
   - Abrir la URL configurada → debe cargar el login
   - Probar login con `admin` / `ADMIN_PASS` (o el default `dipromes2026` si no configuraste esa env var)
   - Revisar que Dashboard, Registros, Máquinas y Consentimiento carguen datos (confirma que `DATABASE_URL` y las migraciones corrieron bien)
   - Probar una exportación (Excel/CSV) y un login con forgot-password si configuraste SMTP

### Checklist de migración
- [x] Base MySQL creada en cPanel
- [x] Setup Python App creado (3.9.23, `passenger_wsgi.py`, entry point `application`)
- [x] Environment variables cargadas (`SECRET_KEY`, `DATABASE_URL`, `ALLOWED_ORIGIN`, mínimo)
- [x] Código subido (Git — clonado en `/home/mybcfcli/dipromesapp`)
- [x] `pip install -r requirements.txt` corrido sin errores (venv del Python App)
- [x] App reiniciada y accesible por URL
- [x] Login funcional, datos cargando (Dashboard con 14 pacientes, 4 activos, 5/8 máquinas, RD$355,500 facturación — datos semilla)
- [x] Dominio propio apuntando a la app: **`https://dipromes.erickhernandezarias.net`**
- [x] `dipromes`/Render sigue intacto — no se ha tocado nada allá

### ✅ Deploy funcional (2026-09-15) — URL final: `https://dipromes.erickhernandezarias.net`

**La app quedó corriendo en un subdominio dedicado, NO en `globalistinternational.org/dipromesapp` como se planeó originalmente.** Motivo: `globalistinternational.org` tiene un WordPress en la raíz cuyas reglas de `RewriteRule` en `public_html/.htaccess` interceptaban cualquier ruta que no fuera un archivo/carpeta física (`RewriteCond %{REQUEST_FILENAME} !-f/-d` → `/index.php`), así que todas las llamadas a `/dipromesapp/api/...` devolvían el 404 de WordPress antes de llegar a Passenger/Flask. La carpeta `/dipromesapp/` en sí cargaba bien (por ser directorio real), lo que hizo el diagnóstico confuso al principio. Se decidió (con el usuario) no tocar el WordPress y en vez de eso mover la app a un subdominio limpio.

**Configuración final:**
- Subdominio: `dipromes.erickhernandezarias.net`, document root propio en `/home/mybcfcli/dipromes.erickhernandezarias.net` (NO comparte docroot con `erickhernandezarias.net` ni `globalistinternational.org`)
- Setup Python App: mismo Application root `/home/mybcfcli/dipromesapp` (mismo código), Application URL cambiada al subdominio (sin subpath) → `.htaccess` autogenerado con `PassengerBaseURI "/"` (raíz limpia)
- `erickhernandezarias.net` usa DNS de **Cloudflare** (`jacob.ns.cloudflare.com`/`stephane.ns.cloudflare.com`), no los nameservers de Banahosting — el registro `A` para `dipromes` se creó manualmente en el dashboard de Cloudflare (`dipromes` → `50.31.176.135`, proxy **DNS only**, no proxied) porque crear el subdominio en cPanel no basta cuando el DNS real vive en otro proveedor.
- SSL: se emitió vía AutoSSL (Let's Encrypt) desde cPanel → SSL/TLS Status → Run AutoSSL, una vez el DNS de Cloudflare ya resolvía. (Un primer intento del usuario instaló un certificado autofirmado por error — no sirve para navegadores, hubo que forzar AutoSSL para reemplazarlo.)
- `ALLOWED_ORIGIN` actualizado a `https://dipromes.erickhernandezarias.net`

**Bugs de código encontrados y arreglados en esta sesión** (aplicados tanto en local como directamente en el servidor vía terminal/Execute Python Script, luego confirmados idénticos):
1. **`passenger_wsgi.py` se corrompía al crear/editar la app en Setup Python App** — cPanel sobrescribía el archivo real con un stub genérico que se cargaba a sí mismo (`RecursionError`). Solución final: escribirlo directamente por Terminal con `cat > passenger_wsgi.py << 'EOF'` tecleado (no pegado) — pegar bloques grandes en esa Terminal web los corrompía o colgaba la sesión.
2. **`hashlib.scrypt` no existe en el Python 3.9 de Banahosting** (OpenSSL sin soporte scrypt) → `generate_password_hash()` de Werkzeug (que usa scrypt por defecto) crasheaba con `AttributeError`. Fix en `backend/app.py`: se envuelve `generate_password_hash` para forzar `method="pbkdf2:sha256"` (líneas ~15-25). Esto es específico de este hosting — no aplica a `dipromes`/Render.
3. **Bug de orden en el seed inicial**: `apply_migrations()` crea el usuario `feli` (vía `_ensure_user`) *antes* de que `seed_if_empty()` revise `Usuario.query.count() == 0` — en una base de datos nueva, `feli` ya existe para cuando se hace esa revisión, así que `admin`/`dr1` nunca se creaban. Fix: `seed_if_empty()` ahora usa `_ensure_user("admin", ...)` y `_ensure_user("dr1", ...)` en vez del bloque `if count == 0`. Este bug también afectaría a `dipromes`/Render si alguna vez arrancara con una base vacía (nunca ha pasado porque su base tiene meses de datos).
4. **Frontend (`index.html`) usaba rutas absolutas `/api/...`** asumiendo que la app vive en la raíz del dominio. Se agregó `const API_BASE = location.pathname.replace(/\/(index\.html)?$/, '')` y se prefijaron las ~10 llamadas (`api.get/post/put/del`, exportaciones, logout, PDF de consentimiento) con `API_BASE`. Con la app ahora en la raíz del subdominio esto es un no-op (`API_BASE` = `""`), pero deja la app portable a un subpath en el futuro sin romperse.

**⚠️ Estado del repo:** los fixes de esta sesión (`backend/app.py`, `backend/requirements.txt`, `index.html`, este `CLAUDE.md`) están aplicados en el servidor y en el working tree local, **pero todavía NO se han comiteado ni pusheado a GitHub** — el usuario no ha confirmado. El único commit real pusheado sigue siendo `41138f6` (root `requirements.txt`). Antes de continuar, preguntar si se quiere comitear/pushear el resto.

**Pendiente opcional:**
- Password de `admin`/`dr1`/`feli` siguen en su default (`dipromes2026` / `doctor123` / `Feli@2026`) — cambiar si se van a usar en producción real, vía `ADMIN_PASS`/`DR1_PASS`/`FELI_PASS` env vars en Setup Python App (requiere borrar el usuario existente o cambiarle el hash manualmente, ya que `_ensure_user` no sobreescribe si ya existe).
- Probar exportaciones (Excel/PDF) y forgot-password (SMTP) end-to-end.
- Considerar apuntar un dominio "bonito" propio de Dipromes al subdominio si se quiere algo más profesional que `dipromes.erickhernandezarias.net`.

### ✅ Recuperación de datos reales de producción desde Render — completada (2026-09-16)

La base de datos real de producción (`dipromes-db`, PostgreSQL en Render) había expirado (política de 30 días del plan free) y quedó `Suspended`. El socio del usuario pagó el upgrade a $6/mes, se desbloqueó, se descargó el backup JSON completo desde `dipromes.onrender.com` (87 registros, 16 máquinas, 6 usuarios reales incluyendo `felipeadmin`, 20 pacientes_master — archivo guardado en `/Users/erickhernandez/Desktop/dipromes_backup.json`, ~12MB) y se importó en `https://dipromes.erickhernandezarias.net`.

**Dos bugs reales se encontraron y arreglaron durante el import** (aplicados en local y en el servidor):

1. **`/api/importar/backup` comparaba usuarios solo por `id`, no por `user`** (`backend/app.py`, función `importar_backup`) — como los IDs de usuario en Render (ej. `feli`=`U008`) no coinciden con los IDs ya sembrados en la base nueva (`feli`=`U001`), el import intentaba insertar un `feli` duplicado y violaba el `UNIQUE` de la columna `user`, tumbando **toda la transacción** (nada se importaba). Fix: `Usuario.query.get(u_data["id"]) or Usuario.query.filter_by(user=u_data["user"]).first()`.
2. **La columna `registros.fotos` era `TEXT` (límite de 64KB en MySQL)** — insuficiente para arrays de fotos en base64 de varios megabytes. MySQL truncaba el valor silenciosamente al insertar (sin lanzar error), dejando JSON inválido que rompía `GET /api/registros` con `JSONDecodeError` al leerlo de vuelta. Fix: `backend/models.py` cambia `fotos` a `db.Text(length=4294967295)` (LONGTEXT), más una migración `ALTER TABLE registros MODIFY COLUMN fotos LONGTEXT` en `apply_migrations()` (`backend/app.py`). Tras ampliar la columna, se re-corrió el mismo import (es idempotente — actualiza por ID) y quedó correcto.
3. También se agregó `SQLALCHEMY_ENGINE_OPTIONS = {"pool_pre_ping": True, "pool_recycle": 280}` y **commits periódicos cada 10 registros** dentro de `importar_backup()` (en vez de un solo commit al final), porque el primer intento de import falló con `MySQL server has gone away` a mitad de proceso — la petición de 12MB tardaba lo suficiente como para que la conexión se cayera antes de terminar.

**⚠️ Corrección importante — datos semilla contaminando el total:** el primer import "exitoso" (41 pacientes, RD$840,512) en realidad tenía los **datos de semilla/demo** (`SEED_REGISTROS`/`SEED_MAQUINAS`, incluye al paciente ficticio "Dr. Felix Batista" en "Maquina No. 1") sumados encima de los datos reales importados — nunca se limpió la base antes de importar, así que ambos coexistían como filas separadas (IDs distintos, sin colisión) e inflaban los totales del dashboard. El usuario lo detectó comparando visualmente el dashboard de Render vs el nuevo.

**Fix:** `POST /api/db/reset` con `{"reseed": false}` (borra `registros`, `maquinas`, `pacientes_master` — **conserva** `usuarios`/`Config`) y luego se re-corrió el mismo import sobre la base ya limpia.

**Resultado final confirmado (coincide exactamente con Render):** Dashboard en `https://dipromes.erickhernandezarias.net` muestra **27 pacientes únicos, 17 activos, 16/8 máquinas operativas, RD$485,012** en facturación — mismos números y mismo primer paciente activo ("Angel Sarante") que el dashboard real de `dipromes.onrender.com`.

**El import se hizo vía `curl` directo desde la Mac del usuario** (no por el navegador) porque el archivo de backup (~12MB) supera el límite de 10MB de la herramienta de upload de Claude in Chrome:
```bash
curl -c cookies.txt -X POST https://dipromes.erickhernandezarias.net/api/auth/login -H "Content-Type: application/json" -d '{"user":"admin","pass":"dipromes2026"}'
curl -b cookies.txt -X POST https://dipromes.erickhernandezarias.net/api/importar/backup -H "Content-Type: application/json" --data-binary @/ruta/al/backup.json
```

**Pendiente opcional:** una vez confirmado que todo está bien, avisar al usuario que puede eliminar `dipromes-db` y el servicio web `dipromes` en Render para no seguir pagando el mes siguiente.

---

## Stack tecnológico

| Capa | Tecnología | Notas |
|------|-----------|-------|
| Frontend | HTML + CSS + Vanilla JS | Un solo archivo `index.html` |
| Backend | Python 3.11 + Flask | `backend/app.py` |
| ORM | SQLAlchemy (Flask-SQLAlchemy 3.1) | Modelos en `backend/models.py` |
| Base de datos | SQLite (dev local) → **MySQL** (Banahosting, este repo) | Driver: `PyMySQL` (pure Python, sin C). `dipromes`/Render usa PostgreSQL + `pg8000` |
| Hosting | Banahosting (cPanel + Passenger) — este repo | `passenger_wsgi.py`. `dipromes`/Render usa `render.yaml` + `wsgi.py` |
| Servidor WSGI | Gunicorn 22 | 2 workers, timeout 60s |
| Íconos | Tabler Icons (webfont CDN) | `ti ti-*` |
| Colores de marca | Crimson `#7B1A1A`, Negro `#1A1A1A`, Blanco `#FFFFFF` | Paleta EHA |
| Export | openpyxl (Excel/CSV), HTML printable (PDF via browser) | Sin reportlab |

---

## Estructura del proyecto

```
dipromes/
├── index.html              # Frontend completo (SPA vanilla JS)
├── wsgi.py                 # Entry point Gunicorn — init DB en first request
├── render.yaml             # Configuración Render.com
├── CLAUDE.md               # Este archivo
├── backend/
│   ├── app.py              # Flask app: rutas, auth, seguridad
│   ├── models.py           # SQLAlchemy models
│   ├── exportar.py         # Helpers: Excel, CSV, PDF/HTML, consent_html
│   └── requirements.txt    # 6 paquetes (sin C extensions)
└── meditrack-plantilla.xlsx  # Plantilla de importación
```

---

## Módulos del sistema

### 1. Dashboard
Métricas en tiempo real: pacientes únicos, activos ahora, máquinas operativas, facturación del mes.

### 2. Pacientes
Lista deduplicada por nombre (→ por cédula en el futuro). Ficha con historial de colocaciones y total facturado.

### 3. Colocaciones (Activos ahora)
Filtrado en tiempo real: `estatus = Activo`. Acción rápida de retiro.

### 4. Registro Completo
Todas las colocaciones. Filtros por nombre/cédula, estado, mes. Exportar a Excel/CSV/PDF.

### 5. Máquinas (Inventario)
8 máquinas terapéuticas. Asignación 1:1 con paciente activo. CRUD completo.

### 6. Conduces de Descargo (Facturación)
Resumen de facturación y saldos pendientes por paciente.

### 7. Consentimiento Informado
- Módulo en sidebar: **Documentos → Consentimiento**
- Tabla `consentimientos` en la base de datos (MySQL en este repo, PostgreSQL en `dipromes`/Render)
- Pestaña **Registros guardados**: listado de consentimientos con estado Firmado/Pendiente, botones Imprimir y Eliminar
- Pestaña **Nuevo consentimiento**: formulario con autocompletar desde pacientes existentes, campos: nombre, cédula, edad, dirección, teléfono, médico, centro de salud, fecha firma, firmado (checkbox), notas
- `GET /api/consentimientos/<id>/pdf` → genera HTML A4 server-side (sin reportlab), fiel al documento `consentimiento-VAC.docx` original
- Botón **"Formulario en blanco"** en topbar y en la vista

### 8. Mapa GPS
Vista de pacientes activos con coordenadas.

### 9. Usuarios (admin)
Gestión de usuarios del sistema. Solo visible para rol `admin`.

---

## Seguridad implementada

| Aspecto | Implementación |
|---------|---------------|
| Autenticación | Flask `session` con cookie httpOnly firmada (SECRET_KEY fija en Render env) |
| Contraseñas | `werkzeug` scrypt hash — auto-upgrade de plaintext en primer login |
| Autorización | `@login_required` / `@admin_required` en todas las rutas |
| Rate limiting | In-memory: 10 req/min por IP en `/api/auth/login` |
| CORS | Restringido a `ALLOWED_ORIGIN` en producción |
| Headers | X-Frame-Options, X-Content-Type-Options, HSTS, Referrer-Policy |
| CSP | `Content-Security-Policy` con `'unsafe-inline'` — nonce removido porque bloqueaba todos los `onclick` inline del frontend (CSP spec: nonces solo aplican a bloques `<script nonce="...">`, no a event handlers) |
| SRI | Todos los recursos CDN (Tabler Icons @3.46.0, Leaflet CSS/JS, XLSX.js) tienen `integrity="sha384-..."` y `crossorigin="anonymous"` |
| XSS | Función `esc()` en todo innerHTML del frontend — mitigación principal dado que `'unsafe-inline'` está activo |
| Reset emergencia | `ADMIN_RESET_PASS` env var → eliminar después de usar |

**Mozilla Observatory:** A+ (100/100) — auditado agosto 2026.

**Variables de entorno requeridas en Render:**
- `DATABASE_URL` — provista automáticamente por Render PostgreSQL
- `SECRET_KEY` — string fijo y largo (crítico para sesiones multi-worker)
- `ALLOWED_ORIGIN` — `https://dipromes.onrender.com`
- `ADMIN_PASS` / `DR1_PASS` — contraseñas iniciales (solo aplican en seed)
- `FELI_PASS` — contraseña inicial del usuario `feli` (default: `Feli@2026` si no se define)
- `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` — para envío de contraseñas temporales por email (opcional)

---

## Modelo de datos

### Maquina (`maquinas`)
```python
id: String(10) PK          # MAQ01, MAQ02…
nombre: String(100)
serial: String(50)
estado: String(50)         # Operativa | Requiere Revisión | Fuera de Servicio
ubicacion: String(100)
notas: Text
```

### Registro (`registros`) — una fila por colocación
```python
id: String(10) PK          # R001, R002…
fecha: String(10)          # YYYY-MM-DD
nombre: String(200)
cedula: String(30)
sexo: String(1)            # M | F
edad: Integer
lesion: String(200)
direccion: String(300)
tel1, tel2: String(100)
dr_refiere: String(200)
ars: String(100)           # Privado | nombre ARS
maquina: String(10)        # FK → maquinas.id
estatus: String(20)        # Activo | Desactivado
motivo: String(300)
facturacion: Float
saldo_pendiente: Float
modo_uso, condicion_salida, condicion_retorno: String(100)
parametros, notas, notas_seguimiento: Text
proxima_colocacion, fecha_retiro: String(10)
observaciones_retiro: Text
lat, lng: Float
productos, fotos: Text     # JSON arrays
```

### Usuario (`usuarios`)
```python
id: String(10) PK          # U001…
user: String(50) UNIQUE
pass: Text                 # scrypt hash
nombre: String(200)
email: String(200)         # para forgot-password (agregado ago 2026)
rol: String(20)            # admin | usuario
activo: Boolean
```

### Consentimiento (`consentimientos`) ← NUEVO
```python
id: String(10) PK          # C001…
fecha_firma: String(10)    # YYYY-MM-DD
nombre: String(200)
cedula: String(30)
edad: Integer
direccion: String(300)
telefono: String(100)
medico: String(200)
centro_salud: String(200)
firmado: Boolean
notas: Text
```

### PacienteMaster (`pacientes_master`)
```python
nombre: String(200) PK
datos: Text                # JSON con datos extendidos — to_dict() hace json.loads()
```

### Config (`config`)
```python
key: String(50) PK         # ej: 'ars'
value: Text                # JSON
```

---

## Reglas de negocio

1. **Una máquina = un paciente a la vez.** `paciente_actual` se calcula en runtime.
2. **Múltiples colocaciones por paciente.** No se editan — se agregan filas nuevas.
3. **Deduplicación por nombre** (→ por cédula en el futuro).
4. **Montos en DOP** sin decimales. `Intl.NumberFormat('es-DO', {currency:'DOP'})`.
5. **ITBIS 18%** — futuro, al emitir e-CF.
6. **`apply_migrations()`** en `wsgi.py` corre antes de seed — maneja ALTER TABLE que `create_all` no puede. Cada bloque usa try/except + rollback individual para ser idempotente.
7. **No instalar paquetes con C extensions** — Render free tier se cuelga. Usar solo pure-Python.
8. **Importar backup NO sobreescribe usuarios existentes** — protege contraseñas en producción. Solo inserta usuarios con IDs nuevos.
9. **`_ensure_user(username, nombre, password, rol)`** — patrón para crear usuarios idempotentemente en `apply_migrations()`. Actualmente crea `feli` si no existe.

---

## Importación de datos

El frontend acepta **Excel (.xlsx/.xls) y CSV (.csv)**:
- Función `parsearArchivo(file)` detecta extensión y usa XLSX.js
- CSV → `reader.readAsText` + `XLSX.read(text, {type:'string'})`
- Excel → `reader.readAsArrayBuffer` + `XLSX.read(data, {type:'array'})`
- Envía JSON a `POST /api/registros/bulk`

**Columnas esperadas** (primera fila del archivo):
`Nombre, Cédula, Sexo, Edad, Área de lesión, Dirección, Teléfono 1, Teléfono 2, Dr. que refiere, ARS, ID Máquina, N° Colocación, Monto (DOP), Fecha`

---

## Exportación

| Endpoint | Formato | Función |
|----------|---------|---------|
| `GET /api/exportar/registros/excel` | .xlsx | `generate_excel()` |
| `GET /api/exportar/registros/csv` | .csv UTF-8 BOM | `generate_csv()` |
| `GET /api/exportar/registros/pdf` | HTML printable | `generate_pdf()` (requiere reportlab — NO instalado) |
| `GET /api/exportar/maquinas/excel` | .xlsx | inline en app.py |
| `GET /api/exportar/backup` | .json | backup completo (admin) |
| `POST /api/importar/backup` | .json | restaurar backup (admin) |
| `GET /api/consentimientos/<id>/pdf` | HTML printable | `consent_html()` — sin deps extras |

---

## Convenciones de código (frontend)

- Funciones de render: `renderXxx()` → escriben en `#content`
- Funciones de modal: `abrirXxx()` / `verXxxDetalle()` → escriben en `#modal-root`
- Funciones de guardar: `guardarXxx()` → llaman al API y actualizan `DB`
- IDs de inputs: prefijo 2-3 letras + guion + campo (`col-pac`, `cn-nombre`)
- **Nunca** `position:fixed` — rompe el layout
- **Siempre** `esc()` para cualquier valor dinámico en `innerHTML`
- Colores: solo variables CSS (`--brand`, `--ok`, `--warn`, `--danger`)

### Agregar un nuevo módulo de vista
```js
// 1. Sidebar
<div class="nav-item" onclick="go('nuevo')" id="nav-nuevo">
  <i class="ti ti-xxx"></i><span>Nombre</span>
</div>

// 2. Router objects
TITLES    = { ..., nuevo: 'Título de la vista' }
NUEVO_LABELS = { ..., nuevo: 'Acción nueva' }

// 3. Render function
async function renderNuevo(){ ... }

// 4. Router go()
({..., nuevo: renderNuevo})[v]?.()

// 5. handleNuevo()
({..., nuevo: ()=>accionNuevo()})[VIEW]?.()
```

---

## Lista de cambios solicitados (verificada 2026-09-15)

El usuario pasó una lista de 8 cambios deseados. Se verificó contra el código real (no contra CLAUDE.md/documentación) cuáles ya están implementados:

| # | Cambio | Estado | Evidencia |
|---|--------|--------|-----------|
| 1 | "Colocaciones" debe mostrar solo pacientes con máquina activa | ✅ Implementado | `renderColocaciones`/`filterColocaciones` (`index.html:876-938`) filtran estrictamente `estatus==='Activo'` |
| 2 | "Parámetros de configuración del equipo" en incrementos de 25 (100→300) | ✅ Implementado | `parametrosOpts()` (`index.html:471-475`): `[100,125,150,175,200,225,250,275,300]`, ya es un `<select>`, no texto libre |
| 3 | "Próxima visita" integrada con Google Calendar | ❌ No implementado | Es solo `<input type="date">` con alerta en dashboard si faltan ≤7 días (`index.html:663`). Sin API, sin `.ics`, sin link "agregar a calendario" |
| 4 | Quitar campos "máquina" y "área de lesión" al registrar una próxima colocación (paciente ya existente) | ❌ No implementado | Un solo formulario `abrirColocacion()` (`index.html:1475-1568`) se usa tanto para el primer registro como para colocaciones siguientes — siempre incluye `col-maq` y `col-lesion` |
| 5 | Fotos de evidencia a carpeta individual de Google Drive | ❌ No implementado | Sin ninguna referencia a Drive/Cloudinary/OAuth en el código. Fotos siguen como base64 en la columna `fotos` (Text) de MySQL — coincide con el roadmap ya documentado abajo |
| 6 | Poder indicar a qué colocación pertenecen las fotos subidas | ✅ Implementado | Flujo dedicado "Subir fotos" (`abrirSubirFotos()`, `index.html:2708-2762`) obliga a elegir paciente y luego la colocación específica antes de adjuntar fotos |
| 7 | Poder editar una colocación después de registrada | ✅ Implementado | `abrirEditarColocacion()`/`guardarEditarColocacion()` (`index.html:1659-1771`) — modal completo, botón lápiz en varias vistas de lista |
| 8 | Asignar máquina a un paciente recién registrado sin que cuente como nueva colocación | ✅ Implementado | `abrirAsignarEquipo()`/`guardarAsignarEquipo()` (`index.html:1985-2148`) actualiza el mismo registro existente en vez de crear uno nuevo (comentario explícito en el código, línea ~2122) cuando el paciente ya tiene un registro activo sin máquina |

**Pendiente para mañana (2026-09-16 o cuando aplique):** implementar los puntos **#3, #4 y #5**, condicionado a que el usuario conecte una cuenta de Google (Calendar API para #3, Drive API para #5 — ambas requieren OAuth2 + credenciales de Google Cloud Console). El punto #4 es un cambio de formulario puro (no depende de Google) y se puede hacer independientemente si se prefiere adelantarlo.

---

## Roadmap

### ✅ Completado
- [x] Backend Flask + PostgreSQL desplegado en Render
- [x] Autenticación segura (scrypt, sessions, rate limiting, headers)
- [x] CRUD completo: Registros, Máquinas, Usuarios, Config
- [x] Importar Excel y CSV (client-side parsing → `/api/registros/bulk`)
- [x] Exportar Excel, CSV desde backend
- [x] Backup/restore JSON completo (import protege usuarios existentes)
- [x] Módulo Consentimiento Informado (DB + PDF server-side)
- [x] Mapa GPS de pacientes activos
- [x] Gestión de usuarios (admin) con campo email
- [x] Pantalla login: forgot-password con envío de contraseña temporal por email (smtplib/STARTTLS)
- [x] Usuario `feli` creado automáticamente vía `_ensure_user()` en migrations

### 🔲 Pendiente
- [ ] Integración ECF SSD como PSFE (e-CF DGII tipo 01/02)
- [ ] Reportes 606/607 para la DGII
- [ ] Historial de pagos parciales (tabla `pagos`)
- [ ] Agenda de citas / mantenimiento de máquinas
- [ ] Alertas WhatsApp al vencer colocación
- [ ] App móvil (Flutter o React Native)
- [ ] Migrar frontend a React + Vite + TypeScript
- [ ] SSL explícito en conexión PostgreSQL (pg8000 ssl_context)
- [ ] Fotos migradas a Cloudinary (actualmente base64 en DB)

---

## Contexto del negocio

- **Empresa:** Dipromes Terapias VAC · Calle 6 Santo Tomás de Aquino No. 55, Zona Universitaria, Santo Domingo · RNC 131950965
- **Operación:** Máquinas de terapia VAC se colocan en pacientes en domicilio o centros médicos. Técnico instala → máquina queda X días → técnico retira → se cobra.
- **Volumen:** ~112 registros agosto 2026, ~15 pacientes únicos, hasta 22 máquinas (MAQ01–MAQ22)
- **Facturación típica:** RD$13,000 – RD$22,000 por colocación
- **ARS:** Actualmente todos `Privado`. ARS previstas en el futuro.
- **Dev:** Erick Hernández Arias · Inicio: junio 2026

---

## Archivos de referencia

| Archivo | Descripción |
|---------|-------------|
| `Registro_Pacientes_Mayo_2026-Activos.xlsx` | Excel original — fuente de verdad inicial |
| `consentimiento-VAC.docx` | Documento original de consentimiento informado |
| `index.html` | Frontend SPA completo |
| `backend/app.py` | Flask app: todas las rutas y lógica de negocio |
| `backend/models.py` | Modelos SQLAlchemy |
| `backend/exportar.py` | Helpers de exportación |
| `wsgi.py` | Entry point: init DB, migrations, seed, emergency reset |
| `render.yaml` | Config de deploy (pythonVersion: 3.11.0) |

---

*Proyecto: DIPROMES · Cliente: Dipromes Terapias VAC, Santo Domingo RD · Dev: Erick Hernández Arias · Inicio: junio 2026*
