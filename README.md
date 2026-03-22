# DB-De-Mediciones — Documentación de la Base de Datos

Resumen
-------
Este repositorio contiene el diseño y documentación de la base de datos utilizada por un sistema IoT para monitoreo de energía eléctrica orientado a la supervisión del comportamiento de fusibles termomagnéticos y la detección de anomalías en circuitos derivados. El sistema opera en red local (no depende de internet) y prioriza seguridad mediante TLS. Los sensores usados son SCT013-030 (corriente) y ZMPT101B (voltaje) y el microcontrolador es un ESP32 Devkit V1. La lógica de detección de anomalías usa regresión lineal sobre una ventana de muestras (búfer circular) y referencia la NOM-001-SEDE-2012 para evaluar caídas o aumentos de tensión/corriente. Se registran medidas en tiempo real y anomalías detectadas para su consulta histórica y notificación hacia la aplicación cliente.

Propósito de la base de datos
-----------------------------
- Almacenar medidas periódicas de voltaje, corriente y potencia por dispositivo IoT.
- Registrar anomalías detectadas (ventana inicial/final, valor, mensaje y tipo).
- Mantener información básica de los dispositivos registrados.
- Permitir consultas en tiempo real y análisis histórico de eventos.

Base de datos
-------------
Para la persistencia de los datos y el manejo eficiente de las mediciones y anomalías generadas por el módulo IoT, se diseñó e implementó una base de datos relacional. Su esquema se compone de distintas tablas que se detallan en la Tabla correspondiente en la documentación del proyecto. Debido al alto volumen de registros por parte del microcontrolador (muestras en milisegundos), se utilizó TimescaleDB (extensión sobre PostgreSQL) para optimizar el almacenamiento y las consultas de series temporales.

Se adoptó un modelo normalizado en forma de esquema en estrella: la tabla dimensional `devices` centraliza metadatos estáticos y las tablas de hecho (`measures`, `anomalies`, `notifications`) se vinculan mediante la llave foránea `device_id`. Esto minimiza la redundancia por cada medición (2NF/3NF), manteniendo los payloads pequeños y optimizando el costo de almacenamiento y la velocidad de escritura/lectura.

Diagrama
--------
El diagrama de las tablas `measures`, `anomalies`, `notifications` y `devices` se encuentra como imagen en `Imagenes/diagrama-db.png` (Figura del documento).

Esquema (DDL sugerido para PostgreSQL + TimescaleDB)
---------------------------------------------------
Nota: ejecutar `CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;` antes de crear hypertables.

```sql
-- Habilitar TimescaleDB (ejecutar una vez)
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- Tabla de dispositivos (dimensional)
CREATE TABLE devices (
  device_id      smallserial PRIMARY KEY,
  device_tag     text NOT NULL UNIQUE,
  register_date  timestamptz NOT NULL DEFAULT now(),
  first_measurement_date timestamptz
);

-- Tabla de medidas (hecho) - alta frecuencia
CREATE TABLE measures (
  id        bigserial PRIMARY KEY,
  time      timestamptz NOT NULL,
  current_a real NOT NULL,
  voltage_v real NOT NULL,
  power_w   real,
  device_id smallint NOT NULL REFERENCES devices(device_id) ON DELETE CASCADE
);

-- Convertir measures a hypertable (TimescaleDB)
SELECT create_hypertable('measures', 'time', if_not_exists => TRUE);

-- Habilitar compresión columnar por device_id y agregar política de compresión a 7 días
ALTER TABLE measures SET (
  timescaledb.compress = TRUE,
  timescaledb.compress_segmentby = 'device_id'
);
SELECT add_compression_policy('measures', INTERVAL '7 days');

-- Tabla de anomalías (hecho). Se usa last_timestamp como dimensión temporal.
CREATE TABLE anomalies (
  id               bigserial PRIMARY KEY,
  device_id        smallint NOT NULL REFERENCES devices(device_id) ON DELETE CASCADE,
  tag              text,
  anomaly_type     text,
  anomaly_message  text,
  last_value       real,
  first_value      real,
  last_timestamp   timestamptz NOT NULL,
  first_timestamp  timestamptz,
  is_read          boolean DEFAULT false
);

-- Convertir anomalies a hypertable
SELECT create_hypertable('anomalies', 'last_timestamp', if_not_exists => TRUE);

-- Opcional: compresión para anomalies
ALTER TABLE anomalies SET (
  timescaledb.compress = TRUE,
  timescaledb.compress_segmentby = 'device_id'
);
SELECT add_compression_policy('anomalies', INTERVAL '7 days');

-- Tabla de notificaciones (transitorias/asíncronas)
CREATE TABLE notifications (
  id          bigserial PRIMARY KEY,
  device_id   smallint REFERENCES devices(device_id) ON DELETE CASCADE,
  message     text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  is_read     boolean DEFAULT false
);

-- Convertir notifications a hypertable si se desea historizar por tiempo
SELECT create_hypertable('notifications', 'created_at', if_not_exists => TRUE);
```

Índices y optimizaciones
------------------------
Para optimizar consultas sobre grandes volúmenes de datos se definieron índices, índices compuestos e índices parciales:

```sql
-- Índice compuesto para lecturas rápidas por dispositivo y tiempo (measures)
CREATE INDEX idx_measures_device_time ON measures(device_id, time DESC);

-- Índice compuesto para anomalías por dispositivo, etiqueta y tiempo
CREATE INDEX idx_anomalies_device_tag_time ON anomalies(device_id, tag, last_timestamp DESC);

-- Índice parcial para notificaciones no leídas (consulta frecuente en la app cliente)
CREATE INDEX idx_notifications_unread_device_time
  ON notifications(device_id, created_at DESC)
  WHERE is_read = FALSE;
```

Notas sobre índices
- Los índices mejoran lecturas pero impactan ligeramente escrituras/espacio; se eligieron compuestos por `device_id` para agrupar consultas por dispositivo.
- El índice parcial en `notifications` acelera la obtención de notificaciones pendientes sin indexar filas ya leídas.

Retención, particionado y compresión
------------------------------------
- Las hypertables de TimescaleDB permiten particionado transparente por tiempo, optimizando inserciones en la partición activa.
- La compresión columnar (configurada con `compress_segmentby = 'device_id'`) agrupa datos por dispositivo y ordena por fecha, reduciendo espacio en disco para históricos.
- La política de compresión automática (`add_compression_policy`) comprime datos más antiguos a 7 días por defecto (configurable).

Políticas de mantenimiento recomendadas
--------------------------------------
- Definir retención/archivado: p. ej., mantener datos de alta resolución 30 días, luego agregar agregados y purgar detalles antiguos.
- Particionado adicional o downsampling para cargas muy altas.
- Monitorizar tamaño de chunks en TimescaleDB y ajustar parámetros de compresión según acceso histórico.

Consultas de ejemplo
--------------------
- Últimas N mediciones de un dispositivo:
```sql
SELECT time, current_a, voltage_v, power_w
FROM measures
WHERE device_id = 1
ORDER BY time DESC
LIMIT 100;
```

- Anomalías no leídas:
```sql
SELECT id, device_id, anomaly_type, anomaly_message, first_timestamp, last_timestamp, is_read
FROM anomalies
WHERE is_read = false
ORDER BY last_timestamp DESC;
```

- Marcar una anomalía como leída:
```sql
UPDATE anomalies
SET is_read = true
WHERE id = 123;
```

Integración con la aplicación y contenedores
--------------------------------------------
- El ESP32 envía medidas al servidor; el servidor inserta en `measures` y la lógica de detección registra filas en `anomalies` y `notifications` cuando procede.
- La base de datos se despliega dentro de un contenedor Docker (ver sección Docker), mapeando el puerto 5432 del contenedor al host. Los scripts SQL se montan como volúmenes y las variables de entorno se leen desde `.env`.
- Se eliminaron privilegios root dentro del contenedor y se aplicaron opciones de seguridad (p. ej. `security_opt`) para impedir elevación de privilegios.

Buenas prácticas y recomendaciones
---------------------------------
- Usar roles de DB con privilegios mínimos para la API y para herramientas de mantenimiento.
- Respaldos periódicos y respaldo/encriptación fuera del contenedor.
- Validaciones adicionales (triggers/constraints) si se requiere comprobar límites de NOM-001-SEDE-2012 en inserciones masivas.
- Considerar `double precision` si la detección requiere mayor precisión numérica.

Ejemplo mínimo de datos (pruebas)
---------------------------------
```sql
INSERT INTO devices (device_tag, first_measurement_date) VALUES ('esp32-01', now());

INSERT INTO measures (time, current_a, voltage_v, power_w, device_id)
VALUES
  (now() - interval '2 minutes', 0.1, 120.3, 12.03, 1),
  (now() - interval '1 minute', 6.6, 98.2, 648.12, 1),
  (now(), 6.8, 97.9, 666.2, 1);

INSERT INTO anomalies (device_id, tag, anomaly_type, anomaly_message, first_value, last_value, first_timestamp, last_timestamp)
VALUES
  (1, 'circuita', 'voltage_drop', 'Caída de voltaje mayor a la NOM-001-SEDE-2012', 120.3, 97.9, now() - interval '2 minutes', now());
```

Referencias
-----------
- TimescaleDB — documentación oficial de hypertables, compresión y políticas.
- NOM-001-SEDE-2012 — Instalaciones Eléctricas.
- Sensores: SCT013-030 (transformador de corriente), ZMPT101B (detección de voltaje).
- Microcontrolador: ESP32 Devkit V1.

Contacto y próximos pasos
------------------------
He actualizado el README para reflejar la arquitectura de base de datos descrita en la subsección: TimescaleDB, hypertables, compresión con política a 7 días, índices compuestos y parciales, tabla `notifications` y despliegue en Docker con medidas de seguridad. Puedo ahora:
- Generar scripts de migración / archivo SQL listo para ejecutar.
- Crear un `docker-compose.yml` de ejemplo que monte el volumen con los scripts y habilite la extensión TimescaleDB.
- Proveer versiones del DDL para SQLite/MySQL (si se requiere).
Dime cuál de estas tareas quieres que haga a continuación.
