# DB-De-Mediciones — Documentación de la Base de Datos

## Resumen

Este repositorio contiene el diseño y documentación de la base de datos utilizada por un sistema IoT para monitoreo de energía eléctrica orientado a la supervisión del comportamiento de fusibles termomagnéticos y la detección de anomalías en circuitos derivados. El sistema opera en red local (no depende de internet) y prioriza seguridad mediante TLS. Los sensores usados son SCT013-030 (corriente) y ZMPT101B (voltaje) y el microcontrolador es un ESP32 Devkit V1. La lógica de detección de anomalías usa regresión lineal sobre una ventana de muestras (búfer circular) y referencia la NOM-001-SEDE-2012 para evaluar caídas o aumentos de tensión/corriente. Se registran medidas en tiempo real y anomalías detectadas para su consulta histórica y notificación hacia la aplicación cliente.

## Propósito de la base de datos

- Almacenar medidas periódicas de voltaje, corriente y potencia por dispositivo IoT.
- Registrar anomalías detectadas (ventana inicial/final, valor, mensaje y tipo).
- Mantener información de dispositivos y usuarios registrados, así como la relación entre ellos.
- Permitir consultas en tiempo real y análisis histórico de eventos.
- Notificar al servidor sobre actualizaciones de conteo de mediciones mediante `pg_notify`.

## Base de datos

Para la persistencia de los datos y el manejo eficiente de las mediciones y anomalías generadas por el módulo IoT, se diseñó e implementó una base de datos relacional. Debido al alto volumen de registros por parte del microcontrolador (muestras en milisegundos), se utilizó TimescaleDB (extensión sobre PostgreSQL) para optimizar el almacenamiento y las consultas de series temporales.

Se adoptó un modelo normalizado: la tabla dimensional `devices` centraliza metadatos estáticos de cada dispositivo y las tablas de hecho (`measures`, `anomalies`) se vinculan mediante la llave foránea `device_id`. Las tablas `users` y `user_devices` gestionan el acceso de usuarios a dispositivos. Esto minimiza la redundancia por cada medición (2NF/3NF), manteniendo los payloads pequeños y optimizando el costo de almacenamiento y la velocidad de escritura/lectura.

## Diagrama

El diagrama de las tablas se encuentra como imagen en `Imagenes/diagrama-db.png`. Las tablas principales son: `devices`, `measures`, `anomalies`, `users` y `user_devices`.

## Esquema real (extraído del dump de PostgreSQL 16 + TimescaleDB)

> **Nota:** ejecutar `CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;` antes de crear las hypertables.

```sql
-- Habilitar TimescaleDB (ejecutar una vez)
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- ─────────────────────────────────────────
-- Tabla de usuarios
-- ─────────────────────────────────────────
CREATE TABLE public.users (
    user_id   serial PRIMARY KEY,
    username  character varying(30)  NOT NULL,
    password  character varying(255) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

-- ─────────────────────────────────────────
-- Tabla de dispositivos (dimensional)
-- ─────────────────────────────────────────
CREATE TABLE public.devices (
    device_id              smallserial PRIMARY KEY,
    device_tag             text        NOT NULL UNIQUE,
    register_date          timestamptz NOT NULL DEFAULT now(),
    first_measurement_date timestamptz,
    count_of_measurements  integer     NOT NULL DEFAULT 0
);

-- ─────────────────────────────────────────
-- Relación usuario ↔ dispositivo (N:M)
-- ─────────────────────────────────────────
CREATE TABLE public.user_devices (
    user_id   integer  NOT NULL REFERENCES public.users(user_id)   ON DELETE CASCADE,
    device_id smallint NOT NULL REFERENCES public.devices(device_id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, device_id)
);

-- ─────────────────────────────────────────
-- Tabla de medidas (hecho) — alta frecuencia
-- ─────────────────────────────────────────
CREATE TABLE public.measures (
    "time"    timestamptz NOT NULL DEFAULT now(),
    current_a real        NOT NULL,
    voltage_v real        NOT NULL,
    power_w   real        NOT NULL,
    device_id smallint    NOT NULL REFERENCES public.devices(device_id) ON DELETE CASCADE
);

-- Convertir measures a hypertable (TimescaleDB)
SELECT create_hypertable('measures', 'time', if_not_exists => TRUE);

-- Compresión columnar segmentada por device_id, con política automática a 7 días
ALTER TABLE public.measures SET (
    timescaledb.compress          = TRUE,
    timescaledb.compress_segmentby = 'device_id'
);
SELECT add_compression_policy('measures', INTERVAL '7 days');

-- ─────────────────────────────────────────
-- Tabla de anomalías (hecho)
-- La dimensión temporal de la hypertable es first_timestamp.
-- ─────────────────────────────────────────
CREATE TABLE public.anomalies (
    device_id       smallint    NOT NULL REFERENCES public.devices(device_id),
    tag             text        NOT NULL,
    anomaly_type    text        NOT NULL,
    anomaly_message text,
    last_value      real        NOT NULL,
    first_value     real        NOT NULL,
    last_timestamp  timestamptz NOT NULL,
    first_timestamp timestamptz NOT NULL,
    is_read         boolean     NOT NULL DEFAULT false
);

-- Convertir anomalies a hypertable (dimensión: first_timestamp)
SELECT create_hypertable('anomalies', 'first_timestamp', if_not_exists => TRUE);

-- Compresión opcional para anomalies
ALTER TABLE public.anomalies SET (
    timescaledb.compress          = TRUE,
    timescaledb.compress_segmentby = 'device_id'
);
SELECT add_compression_policy('anomalies', INTERVAL '7 days');
```

## Función y trigger

Al insertar una nueva medida, el trigger `after_measure_insert` incrementa el contador `count_of_measurements` en `devices` y emite una notificación asíncrona via `pg_notify` para que la aplicación cliente actualice su estado en tiempo real.

```sql
CREATE OR REPLACE FUNCTION public.update_device_measurements_count()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    UPDATE public.devices
    SET count_of_measurements = count_of_measurements + 1
    WHERE device_id = NEW.device_id;

    PERFORM pg_notify(
        'device_measurements_update',
        json_build_object(
            'device_id',          NEW.device_id,
            'measurements_count', (
                SELECT count_of_measurements
                FROM public.devices
                WHERE device_id = NEW.device_id
            )
        )::text
    );

    RETURN NEW;
END;
$$;

CREATE TRIGGER after_measure_insert
AFTER INSERT ON public.measures
FOR EACH ROW EXECUTE FUNCTION public.update_device_measurements_count();
```

## Índices y optimizaciones

```sql
-- measures: índice por tiempo (hypertable default) y compuesto por dispositivo
CREATE INDEX measures_time_idx
    ON public.measures ("time" DESC);

CREATE INDEX idx_measures_device_time
    ON public.measures (device_id, "time" DESC);

-- anomalies: índice por tiempo (hypertable default)
CREATE INDEX anomalies_first_timestamp_idx
    ON public.anomalies (first_timestamp DESC);

-- anomalies: índice compuesto por dispositivo, etiqueta y tiempo
CREATE INDEX idx_anomalies_device_tag_time
    ON public.anomalies (device_id, tag, first_timestamp DESC);

-- anomalies: índice parcial para consultas de anomalías no leídas (acceso frecuente)
CREATE INDEX idx_anomalies_unread_device_tag_time
    ON public.anomalies (device_id, tag, first_timestamp DESC)
    WHERE is_read = false;
```

### Notas sobre índices

- Los índices compuestos por `device_id` agrupan consultas por dispositivo, que es el acceso más frecuente desde la aplicación cliente.
- El índice parcial `idx_anomalies_unread_device_tag_time` acelera la obtención de anomalías pendientes sin indexar las ya leídas, reduciendo su tamaño en escrituras frecuentes.
- `measures_time_idx` y `anomalies_first_timestamp_idx` son generados automáticamente por TimescaleDB al crear la hypertable; se documentan aquí por completitud.

## Retención, particionado y compresión

- Las hypertables de TimescaleDB particionan de forma transparente por tiempo en chunks semanales (intervalo por defecto), optimizando inserciones en la partición activa.
- La compresión columnar (`compress_segmentby = 'device_id'`) agrupa datos por dispositivo y los ordena cronológicamente, reduciendo espacio en disco para históricos.
- La política automática (`add_compression_policy`) comprime chunks con más de 7 días de antigüedad (configurable).
- La hypertable `measures` usa `time` como dimensión temporal; la hypertable `anomalies` usa `first_timestamp`.

## Políticas de mantenimiento recomendadas

- Definir retención/archivado: p. ej., mantener datos de alta resolución 30 días, luego generar agregados y purgar detalles antiguos con `add_retention_policy`.
- Monitorizar tamaño de chunks con `timescaledb_information.chunks` y ajustar parámetros de compresión según el patrón de acceso histórico.
- Usar roles de DB con privilegios mínimos para la API y para herramientas de mantenimiento.
- Respaldos periódicos con `pg_dump` y encriptación fuera del contenedor.

## Consultas de ejemplo

**Últimas N mediciones de un dispositivo:**
```sql
SELECT "time", current_a, voltage_v, power_w
FROM public.measures
WHERE device_id = 1
ORDER BY "time" DESC
LIMIT 100;
```

**Anomalías no leídas (todas):**
```sql
SELECT device_id, tag, anomaly_type, anomaly_message,
       first_value, last_value, first_timestamp, last_timestamp
FROM public.anomalies
WHERE is_read = false
ORDER BY first_timestamp DESC;
```

**Anomalías no leídas de un dispositivo específico:**
```sql
SELECT device_id, tag, anomaly_type, anomaly_message,
       first_value, last_value, first_timestamp, last_timestamp
FROM public.anomalies
WHERE device_id = 1
  AND is_read = false
ORDER BY first_timestamp DESC;
```

**Marcar una anomalía como leída:**
```sql
UPDATE public.anomalies
SET is_read = true
WHERE device_id = 1
  AND tag = 'circuita'
  AND first_timestamp = '2026-02-10 12:00:00+00';
```

> **Nota:** `anomalies` no tiene columna `id`. La clave lógica de una anomalía es la combinación `(device_id, tag, first_timestamp)`.

**Dispositivos de un usuario:**
```sql
SELECT d.device_id, d.device_tag, d.count_of_measurements
FROM public.devices d
JOIN public.user_devices ud USING (device_id)
WHERE ud.user_id = 1;
```

## Ejemplo mínimo de datos (pruebas)

```sql
-- Usuario
INSERT INTO public.users (username, password)
VALUES ('admin', 'hash_aqui');

-- Dispositivo
INSERT INTO public.devices (device_tag, first_measurement_date)
VALUES ('esp32-01', now());

-- Relación usuario-dispositivo
INSERT INTO public.user_devices (user_id, device_id) VALUES (1, 1);

-- Medidas (el trigger incrementa count_of_measurements automáticamente)
INSERT INTO public.measures ("time", current_a, voltage_v, power_w, device_id)
VALUES
    (now() - interval '2 minutes', 0.1, 120.3, 12.03, 1),
    (now() - interval '1 minute',  6.6,  98.2, 648.12, 1),
    (now(),                        6.8,  97.9, 666.20, 1);

-- Anomalía
INSERT INTO public.anomalies
    (device_id, tag, anomaly_type, anomaly_message,
     first_value, last_value, first_timestamp, last_timestamp)
VALUES
    (1, 'circuita', 'voltage_drop',
     'Caída de voltaje mayor a la NOM-001-SEDE-2012',
     120.3, 97.9,
     now() - interval '2 minutes', now());
```

## Integración con la aplicación y contenedores

- El ESP32 envía medidas al servidor; el servidor inserta en `measures`. El trigger `after_measure_insert` actualiza `count_of_measurements` en `devices` y emite `pg_notify('device_measurements_update', ...)` para notificar a los clientes suscritos.
- La lógica de detección de anomalías registra filas en `anomalies` cuando detecta una desviación respecto a la NOM-001-SEDE-2012.
- La base de datos se despliega dentro de un contenedor Docker, mapeando el puerto 5432 del contenedor al host. Los scripts SQL se montan como volúmenes y las variables de entorno se leen desde `.env`.
- Se eliminaron privilegios root dentro del contenedor y se aplicaron opciones de seguridad (p. ej. `security_opt: no-new-privileges`) para impedir elevación de privilegios.

## Buenas prácticas y recomendaciones

- Usar roles de DB con privilegios mínimos (uno para la API, otro para mantenimiento).
- Al no tener `id` en `measures` y `anomalies`, las consultas de actualización o referencia deben usar las claves naturales o rangos de tiempo.
- Considerar `double precision` si la detección requiere mayor precisión numérica que `real`.
- Validaciones adicionales (triggers/constraints) si se requiere verificar límites de NOM-001-SEDE-2012 en inserciones masivas.

## Referencias

- TimescaleDB — documentación oficial de hypertables, compresión y políticas.
- NOM-001-SEDE-2012 — Instalaciones Eléctricas (Utilización).
- Sensores: SCT013-030 (transformador de corriente), ZMPT101B (detección de voltaje).
- Microcontrolador: ESP32 Devkit V1.
