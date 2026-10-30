# Depósito Patagonia

Primera versión de un sistema web de depósito para ejecutar como contenedor Docker y administrar desde Portainer. La información se guarda en SQLite dentro de un volumen persistente.

## Funciones incluidas

- Inicio de sesión y cambio de contraseña desde **Cuenta**.
- Alta de contenedores y generación automática de un código numérico de 12 dígitos.
- Jerarquía de contenedores: desde una ficha se pueden crear cajas/subcontenedores, cada uno con etiqueta y código propios.
- Retiro de artículos de un contenedor y eliminación de contenedores o de toda una rama, con ajuste de stock registrado en el historial.
- Etiquetas imprimibles con código de barras Code 128. Los códigos importados se conservan y no se reciclan.
- Catálogo de productos con columnas de código, nombre, item, color, talle, descripción, unidades y ubicación. El listado permite ver Bariloche, Retiro o ambos; no incluye Centro de Distribución.
- El catálogo permite archivar referencias que no se tienen y restaurarlas si hace falta.
- Stock por contenedor, consulta y búsqueda por ubicación.
- Lector de código por teclado USB (los lectores configurados como teclado suelen enviar el código y Enter).
- Ingresos, salidas y traslados, con control de cantidad disponible y registro de movimientos.
- Importación y exportación de contenedores, productos y stock en Excel.

El importador admite la planilla original de productos con una hoja `elementos` y la plantilla de tres hojas. La planilla original trae cantidades, pero no identifica ubicación. Esas cantidades se guardan como stock importado sin ubicar y se trasladan a un contenedor desde **Movimientos** usando `Planilla importada sin ubicación` como origen. Así se conserva el total y queda disponible por Bariloche o Retiro después de asignarlo. La importación no carga el historial anterior de movimientos. La importación hace altas/actualizaciones de los códigos incluidos y conserva los códigos existentes; las filas que no aparezcan en un nuevo archivo no se eliminan automáticamente.

## Despliegue desde Portainer

### 1. Preparar la imagen

Copiá la carpeta `deposito-stock` a un equipo con acceso al Docker Engine del servidor. Desde esa carpeta, construí la imagen:

```sh
docker build -t deposito-stock:1.0 .
```

Si el comando se ejecuta desde otro equipo, ese mismo Docker Engine debe ser el del servidor o la imagen tiene que publicarse en un registry accesible por Portainer.

### 2. Crear el Stack

En Portainer, creá un Stack y pegá el contenido de `compose-portainer.yaml`. En **Environment variables**, definí:

- `SECRET_KEY`: valor aleatorio largo usado para firmar sesiones.
- `ADMIN_USERNAME`: usuario inicial (por defecto `admin`).
- `ADMIN_PASSWORD`: contraseña inicial larga.
- `PORT`: puerto HTTP publicado (por defecto `8000`).
- `SESSION_COOKIE_SECURE`: dejalo `false` si entrás por HTTP directo. Ponelo en `true` cuando el sitio quede detrás de HTTPS.

Portainer crea el volumen `deposito_data`; ahí persiste la base SQLite aunque se reemplace el contenedor. No borres ese volumen al actualizar.

### 3. Primer ingreso

Abrí `http://IP-DEL-SERVIDOR:8000` (o el puerto definido), iniciá sesión con el usuario y contraseña configurados en el Stack y cambiá la contraseña en **Cuenta**. Para exponerlo fuera de la red local, configurá HTTPS en el proxy inverso antes de usar `SESSION_COOKIE_SECURE=true`.

## Alternativa con Compose

Copiá `.env.example` a `.env`, reemplazá los secretos y ejecutá:

```sh
docker compose up -d --build
```

## Importación de datos

Desde **Importar Excel**, descargá `plantilla_deposito.xlsx`. El archivo contiene tres hojas con estas columnas:

- `Contenedores`: `codigo`, `nombre`, `ubicacion`, `sucursal` (Bariloche o Retiro)
- `Productos`: `sku`, `nombre`, `item`, `color`, `talle`, `descripcion`, `unidades_sin_ubicar`, `ubicaciones`
- `Stock`: `codigo_contenedor`, `sku`, `cantidad`

La hoja `elementos` del Excel original se transforma directamente: `codigo` pasa a ser el código del producto, y se conservan nombre, item, color, talle, descripción, unidades y ubicaciones.

Las filas de stock relacionan cada SKU con un contenedor mediante sus códigos. Usá códigos de barras existentes en la columna `codigo` para mantenerlos. La sucursal debe ser Bariloche o Retiro. La aplicación guarda los códigos como texto y los representa en Code 128.

## Archivos

- `app.py`: servidor y reglas del inventario.
- `templates/` y `static/`: interfaz web.
- `compose.yaml`: Stack que construye la imagen desde la carpeta.
- `docker-compose.yml`: nombre predeterminado para la ruta Compose de Portainer; construye la imagen desde el repositorio.
- `compose.yaml`: misma configuración, con nombre Compose moderno.
- `compose-portainer.yaml`: Stack para Portainer usando una imagen ya construida.

## Consideraciones de esta primera entrega

- Usa un solo usuario administrador inicial; no incluye perfiles de permisos ni multiusuario.
- La consulta por código busca el código del contenedor o SKU exacto. El lector funciona como teclado USB.
- La impresión de etiquetas usa una medida inicial de 88 × 50 mm; puede ajustarse a la impresora y etiquetas disponibles.
- Los códigos nuevos son números secuenciales de 12 cifras impresos en Code 128. Si los códigos del sistema anterior usan otra simbología o longitud, el importador mantiene su texto, pero conviene comparar una etiqueta antes de imprimir lotes.
- SQLite funciona bien para un depósito chico. La base se mantiene en un único volumen para facilitar copias de seguridad y restauración.
