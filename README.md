# InvAI — Control inteligente de inventario

InvAI es una aplicación local para controlar existencias con trazabilidad por trabajador. Está orientada a ferreterías, constructoras y pequeñas empresas que ya poseen un sistema tributario y necesitan complementarlo con movimientos, conteos físicos, reportes y compras.

La solución utiliza FastAPI, SQLAlchemy, SQLite, JavaScript nativo y una PWA responsive. La API vive bajo `/api`; la interfaz y los archivos PWA se sirven desde `/`.

## Alcance actual

- Catálogo con nombre, categoría, SKU, código de barras, unidad, ubicación, costo, precio opcional, stock y mínimo.
- Importación de la planilla de bodega (`.xlsx`) desde **Inventario → Importar planilla**: se lee la hoja con las columnas PRODUCTO y CANTIDAD (además UNIDAD, CÓDIGO, CATEGORÍA, SUBCATEGORÍA y OBS), se muestra una vista previa con lo nuevo, lo repetido y lo que ya existe, y solo se crean productos nuevos. El archivo se lee en el navegador; al servidor viajan únicamente las filas confirmadas.
- Cantidades decimales con hasta 3 decimales: permite recibir 12,5 kg de plancha o entregar 3,75 m de perfil.
- Entradas por compra, devolución o transferencia.
- Salidas por venta, entrega a obra, consumo interno, merma, transferencia o devolución a proveedor.
- Trazabilidad con usuario, fecha, motivo, referencia, destino, observación y stock anterior/posterior.
- Protección transaccional contra stock negativo y operaciones parciales.
- Solicitudes de productos nuevos: el empleado propone y un administrador revisa antes de crear catálogo o stock.
- Conteos físicos pendientes de aprobación administrativa, con efecto exacto antes/después y observación de revisión.
- Kardex, valorización a costo, alertas, reportes por trabajador y exportación CSV.
- Cuentas individuales, roles, contraseñas temporales y sesiones revocables.
- Exportación, restauración y vaciado protegido de los datos operativos, exclusivo del propietario.
- Compras: boletas y facturas por proveedor con su detalle de materiales (hojas de boletas de la planilla); no modifican el stock.

InvAI no emite boletas o facturas, no reemplaza al sistema del SII y no lleva contabilidad.

## Uso diario recomendado

1. El encargado revisa productos agotados, bajo mínimo y conteos pendientes en el dashboard.
2. Cuando llega mercadería, busca el producto y registra una entrada con cantidad y documento de referencia.
3. Cuando sale material, selecciona los productos, el motivo y, si corresponde, la obra o destino.
4. Cada movimiento queda asociado automáticamente al usuario conectado.
5. Durante un conteo físico, el trabajador ingresa las cantidades reales y lo envía para revisión.
6. Si un artículo no existe, el trabajador lo solicita con nombre, código, unidad, ubicación y detalle; no se crea stock automáticamente.
7. Un administrador revisa las diferencias del conteo y el resultado final antes de aprobar. Los movimientos posteriores al conteo se conservan.
8. Al cierre, el administrador consulta reportes, descarga el CSV con las columnas de la planilla y revisa las compras.

## Requisitos

- Python 3.12 recomendado.
- Dependencias de `backend/requirements.txt`, incluida `cryptography` (cifrado de la clave del asistente).
- Windows 10/11, macOS o Linux.

## Instalación segura en Windows

```powershell
cd backend
py -3.12 -m venv .venv-win
.\.venv-win\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Edita `backend/.env` y define como mínimo `SECRET_KEY`. Puedes generar una con:

```powershell
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Para una base nueva, define temporalmente `INITIAL_ADMIN_PASSWORD` con una contraseña de 10 a 72 caracteres. Si se omite, la aplicación genera una y la muestra una sola vez en la consola.

Inicia la aplicación:

```powershell
python start.py
```

Abre <http://127.0.0.1:8000>. La documentación de la API está en <http://127.0.0.1:8000/api/docs>.

## Asistente de inventario

La pantalla **Asistente** responde preguntas sobre las existencias, los movimientos y (solo a administradores) las compras de la bodega. Es de solo lectura y está acotado al inventario: ante otros temas responde que solo puede ayudar con la bodega.

Un administrador o el propietario configura el asistente desde la propia pantalla **Asistente**: clave, URL del servicio (cualquier servicio compatible con OpenAI; Groq tiene plan gratuito) y modelo, con un botón para probar la conexión. Los cambios rigen de inmediato, sin reiniciar.

Los valores `AI_API_KEY`, `AI_BASE_URL` y `AI_MODEL` de `backend/.env` quedan solo como valor inicial: se usan cuando la configuración guardada desde la pantalla está vacía.

```text
AI_API_KEY=tu-clave
AI_BASE_URL=https://api.groq.com/openai/v1
AI_MODEL=llama-3.3-70b-versatile
```

- La clave se guarda cifrada con una llave derivada de `SECRET_KEY` y nunca vuelve en ninguna respuesta de la API. Si `SECRET_KEY` cambia (o no está definida y se genera una distinta en cada arranque), la clave guardada deja de poder leerse y hay que volver a ingresarla.
- Los empleados solo ven un aviso genérico cuando el asistente no está disponible; el motivo técnico lo ven únicamente administradores y propietario.
- Al cambiar el servidor (URL) hay que volver a ingresar la clave. Si el servidor no tiene una `SECRET_KEY` fija, la clave guardada se pierde al reiniciar.
- La clave no entra en los respaldos, y la configuración sobrevive al vaciado de datos.
- Estados de falla que la pantalla puede mostrar a administradores: sin clave, clave ilegible (cambió `SECRET_KEY`), clave inválida o sin permisos, cuota agotada, modelo no disponible, proveedor saturado y error desconocido.

Sin clave, la pantalla avisa que no está activo y el resto de la aplicación funciona igual. Al responder, el texto de los productos, los últimos 40 movimientos y (para administradores) las últimas 40 compras se envían al proveedor configurado; no se envían contraseñas ni nombres de usuarios. Cada cuenta tiene un tope de 12 preguntas por minuto.

## Esquema y actualización

El esquema base es SQLite v4. Al iniciar, InvAI agrega de forma compatible los campos operativos, solicitudes de producto, notas de revisión y auditoría de operaciones críticas sin borrar productos, usuarios o movimientos existentes. La migración v4 se realiza en el lugar y no crea un respaldo adicional.

Desde v4 en adelante el esquema lo gestiona **Alembic**, en `backend/migrations/`. La aplicación aplica las migraciones pendientes sola al arrancar, así que no hay que ejecutar nada a mano:

- Una base nueva se crea con los modelos actuales y se sella en la última revisión.
- Una base anterior a Alembic se sella en la línea base (`0001_baseline_v4`) y se le aplica lo que vino después.

Para trabajar sobre el esquema durante el desarrollo:

```powershell
cd backend
python -m alembic current              # revisión aplicada
python -m alembic history              # revisiones disponibles
python -m alembic revision -m "cambio" # crear una nueva
```

Las migraciones corren en modo *batch* porque SQLite no soporta `ALTER COLUMN`: cada cambio de tipo reconstruye la tabla conservando datos, índices y restricciones.

SQLite funciona con claves foráneas, restricciones de stock, espera ante bloqueos y modo WAL. No copies ni manipules la base mientras la aplicación esté ejecutándose.

## Roles y permisos

- `owner`: acceso total, administración de cuentas y cierre remoto de sesiones.
- `admin`: catálogo, dashboard, compras, movimientos, aprobación de conteos y reportes.
- `empleado`: consulta del inventario, registro de entradas/salidas, solicitud de productos nuevos y envío de conteos físicos. No crea fichas directamente ni aprueba ajustes.

Crea una cuenta distinta para cada trabajador desde **Equipo**. Las cuentas nuevas deben cambiar su contraseña temporal antes de operar. Cada empleado conserva una sola sesión activa; iniciar en otro dispositivo revoca la anterior.

La sesión del navegador vive en una cookie `HttpOnly` y `SameSite=Strict`: ningún script de la página puede leerla, y se borra al cerrar el navegador. Las escrituras exigen además la cabecera `X-Requested-With: InvAI`, que un formulario de otro sitio no puede enviar. Tras 5 intentos fallidos sobre una cuenta, o 20 desde un mismo equipo, el ingreso se bloquea 5 minutos. La documentación de la API y los scripts siguen pudiendo autenticarse con `Authorization: Bearer`.

## Movimientos y conteos

Las salidas múltiples usan una clave de idempotencia: un reintento por corte de red no duplica el descuento. El servidor valida todas las líneas dentro de una única transacción y rechaza la operación completa si alguna cantidad no está disponible.

Un conteo no modifica stock al enviarse. La diferencia se aplica solamente cuando un administrador lo aprueba. Si hubo entradas o salidas entre el conteo y la aprobación, se aplica la diferencia detectada sin sobrescribir esos movimientos posteriores.

La revisión muestra producto, ubicación, stock esperado, cantidad física, diferencia y stock resultante. Rechazar exige una explicación visible para quien hizo el conteo.

## Respaldo, restauración y vaciado

La sección **Datos** solo aparece al propietario (`owner`). El respaldo operativo incluye productos, movimientos, salidas, conteos y solicitudes; no incluye contraseñas ni sesiones. El archivo posee versión de formato y checksum SHA-256, por lo que una copia modificada o corrupta es rechazada.

Para vaciar datos se exige contraseña actual, frase exacta, confirmación explícita, un respaldo descargado durante la sesión y una segunda revisión. Usuarios y credenciales se conservan. Antes de restaurar o vaciar, el servidor crea además una copia operativa de seguridad en `backend/backups/`. Esas acciones quedan en la auditoría interna.

La exportación desde la aplicación complementa, pero no sustituye, el respaldo técnico del servidor y de sus cuentas.

## Moneda

Costos, precios, gráficas y reportes se muestran explícitamente en **pesos chilenos (CLP)**, por ejemplo `$70.475 CLP`. La aplicación no convierte automáticamente cifras históricas almacenadas en otra moneda.

La valorización principal utiliza `stock × costo de compra`; el precio de venta es opcional y se usa para calcular el valor potencial.

## Códigos de barras

El SKU y el código de barras son identificadores únicos opcionales. Por ahora pueden escribirse o capturarse con un lector USB/Bluetooth que funcione como teclado. El escaneo mediante cámara del teléfono requiere HTTPS y se considera una mejora posterior.

## Acceso desde teléfonos

`python start.py` escucha en la red local. Conecta el teléfono al mismo router y abre `http://IP-DEL-PC:8000`; consulta la dirección con `ipconfig`.

El acceso HTTP local sirve para probar la aplicación. Para una instalación PWA estable, acceso con cámara o uso fuera del local se requiere HTTPS. No publiques directamente el puerto 8000 en Internet.

## PWA y actualizaciones

El Service Worker nunca intercepta `/api`. Las navegaciones usan red primero, los recursos versionados utilizan caché controlada y las versiones anteriores se eliminan al activar una actualización.

Cuando cambies el frontend, incrementa de forma coordinada `CACHE_NAME` y `PRECACHE` en `frontend/sw.js`, los sufijos `?v=` en `frontend/index.html` y la expectativa en `frontend/security.test.js`. La versión actual es `v47`.

## Despliegue recomendado

- **Demostración académica:** un computador con `python start.py` y teléfonos en la misma red. No exponer el puerto 8000 directamente a Internet.
- **Piloto con una empresa:** una instancia aislada para esa empresa, dominio HTTPS, volumen persistente, respaldo diario fuera del servidor y responsable asignado. Puede comenzar en un solo servidor; Docker Compose facilita separar configuración de producción y aplicar políticas de reinicio.
- **Uso comercial sostenido:** migrar SQLite a PostgreSQL, usar un proxy HTTPS, monitoreo, rotación de secretos, copias técnicas automatizadas y pruebas periódicas de restauración.

La forma práctica de evitar que una empresa entregue el sistema a otra es operar InvAI como servicio alojado: el cliente recibe cuentas y acceso, no el código fuente. Una licencia instalada dentro de un servidor del cliente solo puede disuadir; no existe una protección técnica absoluta si se entrega el código y el control total de la máquina. Complementa la solución con contrato, identificación de empresa y condiciones de soporte.
## Pruebas

```powershell
cd backend
python -m pip install -r requirements-dev.txt
cd ..
python -m pytest -q
node --test frontend\planilla.test.js frontend\security.test.js
```

`backend/conftest.py` crea automáticamente una base temporal aislada y la elimina al terminar. Las pruebas nunca deben utilizar `backend/inventory.db`.

## Endpoints principales

- `GET/POST /api/products`: catálogo.
- `POST /api/products/import`: alta en bloque desde la planilla, en una sola transacción.
- `GET/POST /api/purchases`, `POST /api/purchases/import`, `GET/DELETE /api/purchases/{id}`: compras (solo administradores).
- `GET/POST /api/product-requests`: solicitudes de productos.
- `POST /api/product-requests/{id}/approve`: crea una ficha revisada a partir de una solicitud.
- `POST /api/product-requests/{id}/reject`: devuelve una solicitud con explicación.
- `GET/POST /api/transactions`: historial y movimiento individual.
- `POST /api/sales`: salida múltiple atómica del flujo rápido.
- `GET /api/assistant/status`, `POST /api/assistant/chat`: asistente de inventario (el detalle de fallas solo llega a administradores).
- `GET/PUT /api/admin/ai-settings`, `POST /api/admin/ai-settings/test`: configuración del asistente y prueba de conexión (administradores).
- `GET/POST /api/inventory-counts`: conteos físicos.
- `POST /api/inventory-counts/{id}/approve`: aprobación y ajuste.
- `GET /api/reports/inventory`: indicadores determinísticos.
- `GET /api/reports/kardex/{product_id}`: historial de un producto.
- `GET /api/reports/inventory.csv`: exportación para Excel.
- `GET /api/reports/movements-by-user`: trazabilidad por trabajador.
- `GET /api/admin/data-summary`: resumen previo a una operación crítica.
- `POST /api/admin/backup/export`: exportación operativa protegida.
- `POST /api/admin/backup/restore`: restauración transaccional validada.
- `POST /api/admin/data/reset`: vaciado operativo protegido.

## Docker

```powershell
docker build -t invai .
docker run --rm -p 8000:8000 `
  -v invai-data:/app/data `
  -e SECRET_KEY="una-clave-larga-y-aleatoria" `
  -e INITIAL_ADMIN_PASSWORD="una-clave-inicial-segura" `
  invai
```

La imagen corre sin privilegios, persiste SQLite en `/app/data` y publica un healthcheck en `/api/health`. No incluyas `.env`, bases SQLite ni credenciales dentro de la imagen.

## Estructura

```text
inventory-ai/
├── backend/
│   ├── main.py              API, roles, movimientos, conteos, respaldos y reportes
│   ├── database.py          esquema base, conexión SQLite y arranque de migraciones
│   ├── migrations/          revisiones de Alembic (esquema desde v4 en adelante)
│   ├── ai_settings.py       configuración del asistente: clave cifrada, URL y modelo
│   ├── ai_service.py        consulta al proveedor de IA y estados de falla
│   ├── auth.py              bcrypt, JWT y sesiones
│   ├── models.py            contratos Pydantic
│   └── start.py             inicio local y acceso LAN
├── frontend/
│   ├── index.html           SPA accesible
│   ├── style.css            diseño responsive
│   ├── app.js               flujos operativos
│   ├── planilla.js          lectura de la planilla .xlsx sin dependencias
│   ├── sw.js                actualizaciones PWA
│   └── manifest.json
├── docs/superpowers/
│   ├── specs/               diseño de la configuración del asistente de IA
│   └── plans/               plan de implementación de esa función
├── .claude/skills/          skills de diseño del proyecto (se versionan)
└── Dockerfile
```
