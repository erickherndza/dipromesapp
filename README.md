# DIPROMES

Sistema de gestión de activos médicos y pacientes para **Dipromes Terapias VAC**, negocio de terapia de compresión/presoterapia en Santo Domingo, República Dominicana. El negocio coloca máquinas terapéuticas en pacientes en domicilio o centros médicos y registra cada colocación con su facturación.

> ⚠️ **Este repositorio (`dipromesapp`) es un fork de trabajo** creado a partir de [`dipromes`](https://github.com/erickherndza/dipromes) para migrar el hosting de Render.com a **Banahosting** (cPanel + Passenger). El repo original sigue en producción en Render, sin tocarse. Todo el desarrollo activo ocurre aquí.
>
> **Demo en vivo:** `https://dipromes.erickhernandezarias.net`

## Stack

| Capa | Tecnología |
|------|-----------|
| Frontend | HTML + CSS + Vanilla JS (SPA en un solo archivo `index.html`) |
| Backend | Python 3.9 + Flask (`backend/app.py`) |
| ORM | SQLAlchemy (Flask-SQLAlchemy) — modelos en `backend/models.py` |
| Base de datos | **MySQL** (vía `PyMySQL`, pure-Python) en este repo · `dipromes`/Render usa PostgreSQL |
| Hosting | Banahosting — cPanel Python Selector + Passenger (`passenger_wsgi.py`) |
| Export | openpyxl (Excel/CSV), HTML imprimible (PDF vía navegador) |

## Módulos

- **Dashboard** — métricas en tiempo real: pacientes únicos, activos ahora, máquinas operativas, facturación del mes
- **Pacientes** — historial de colocaciones y total facturado por paciente
- **Colocaciones** — pacientes con máquina activa ahora mismo, retiro rápido
- **Registro Completo** — todas las colocaciones históricas, filtros y exportación
- **Máquinas** — inventario, CRUD, asignación 1:1 con paciente activo
- **Conduces de Descargo** — resumen de facturación y saldos pendientes
- **Consentimiento Informado** — formulario + PDF server-side, listado de firmados/pendientes
- **Mapa GPS** — pacientes activos con coordenadas
- **Usuarios** (admin) — gestión de accesos del sistema

## Correr localmente

No requiere Docker ni servicios externos — usa SQLite por defecto si no hay `DATABASE_URL`.

```bash
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r backend/requirements.txt
python backend/app.py             # sirve en http://127.0.0.1:5000
```

Login inicial (se crean automáticamente al arrancar con base vacía):

| Usuario | Contraseña | Rol |
|---------|-----------|-----|
| `admin` | `dipromes2026` | admin |
| `dr1` | `doctor123` | usuario |
| `feli` | `Feli@2026` | usuario |

Cambia los defaults en producción vía las variables de entorno `ADMIN_PASS`, `DR1_PASS`, `FELI_PASS`.

## Variables de entorno

| Variable | Requerida | Descripción |
|----------|-----------|-------------|
| `DATABASE_URL` | No (default: SQLite local) | `mysql://user:pass@host:3306/db` — se normaliza internamente a `mysql+pymysql://` |
| `SECRET_KEY` | Sí en producción | String largo y fijo para firmar sesiones |
| `ALLOWED_ORIGIN` | Sí en producción | Origen permitido para CORS, ej. `https://tudominio.com` |
| `ADMIN_PASS` / `DR1_PASS` / `FELI_PASS` | No | Sobreescriben los passwords iniciales de seed |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` | No | Habilitan forgot-password por email |

## Estructura del proyecto

```
dipromesapp/
├── index.html              # Frontend completo (SPA vanilla JS)
├── passenger_wsgi.py        # Entry point Passenger (Banahosting)
├── wsgi.py                  # Entry point Gunicorn (compatibilidad con dipromes/Render)
├── backend/
│   ├── app.py                # Rutas Flask, auth, seguridad
│   ├── models.py              # Modelos SQLAlchemy
│   ├── exportar.py            # Helpers de exportación (Excel, CSV, PDF/HTML)
│   └── requirements.txt
└── CLAUDE.md                 # Contexto detallado del proyecto (para desarrollo asistido por IA)
```

## Seguridad

Autenticación con `werkzeug` (hash `pbkdf2:sha256`), sesiones firmadas httpOnly, rate limiting en login, headers de seguridad (CSP, HSTS, X-Frame-Options), SRI en recursos CDN, y sanitización XSS (`esc()`) en todo el frontend.

---
*Cliente: Dipromes Terapias VAC, Santo Domingo, RD · Dev: Erick Hernández Arias*
