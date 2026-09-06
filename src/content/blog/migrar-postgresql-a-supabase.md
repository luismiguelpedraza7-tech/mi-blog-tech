---
title: 'Tutorial: Cómo migrar PostgreSQL a Supabase sin tiempo de inactividad (Zero Downtime)'
description: 'Aprende paso a paso cómo migrar tu base de datos PostgreSQL de producción hacia Supabase sin afectar a tus usuarios utilizando replicación lógica.'
pubDate: 'Aug 31 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

Migrar una base de datos relacional en producción es una de las tareas más delicadas y con mayor responsabilidad para cualquier equipo de desarrollo o ingeniería de datos. Cuando una plataforma crece y requiere delegar la administración del motor de bases de datos hacia un servicio gestionado como **Supabase**, la prioridad número uno es evitar interrupciones del servicio (*downtime*) y prevenir la pérdida de registros transaccionales.

En esta guía exhaustiva aprenderás a realizar una migración completa de un servidor **PostgreSQL** hacia la infraestructura de **Supabase**, aplicando técnicas de **Replicación Lógica (Logical Replication)** para lograr cero tiempo de inactividad.

---

## 1. ¿Por qué Replicación Lógica en lugar de un Dump SQL Tradicional?

El método clásico para mover bases de datos consiste en ejecutar `pg_dump`, exportar un archivo `.sql` gigante, detener la aplicación web, e importar dicho archivo en el nuevo servidor con `pg_restore`.

Sin embargo, para bases de datos transaccionales de producción con un volumen activo de escritura, este enfoque presenta serias desventajas:
- **Tiempo de Inactividad Elevado:** Si el archivo dump pesa varios gigabytes y tarda horas en transferirse, tu servicio debe permanecer completamente fuera de línea durante todo ese tiempo.
- **Riesgo de Desincronización y Pérdida de Datos:** Cualquier registro insertado por usuarios durante el proceso de exportación se perderá si no congelas la base de datos origen.

Con la **Replicación Lógica basada en suscripciones (Publisher/Subscriber)**, la base de datos original continúa respondiendo peticiones con normalidad mientras copia progresivamente todos los datos iniciales hacia Supabase.

---

## 2. Fase de Preparación en el Servidor PostgreSQL Origen

Para que PostgreSQL pueda transmitir sus cambios mediante replicación lógica, debemos verificar y ajustar los parámetros del servidor de origen.

### Paso 2.1: Activar el nivel de WAL adecuado
Abre el archivo de configuración de tu servidor PostgreSQL local o VPS (habitualmente ubicado en `/etc/postgresql/15/main/postgresql.conf`):

```ini
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Guarda los cambios y reinicia el servicio de PostgreSQL en tu servidor ejecutando:

```bash
sudo systemctl restart postgresql
```

### Paso 2.2: Configurar REPLICA IDENTITY en tus tablas
Para que la replicación lógica pueda procesar operaciones de modificación (`UPDATE`) y borrado (`DELETE`), cada tabla debe tener una clave primaria identificable. Si tienes tablas sin clave primaria, debes establecer su `REPLICA IDENTITY` en `FULL`:

```sql
ALTER TABLE nombre_de_tabla REPLICA IDENTITY FULL;
```

### Paso 2.3: Crear el rol de migración con permisos de Replicación
Accede a la consola de PostgreSQL (`psql`) como superusuario y ejecuta los siguientes comandos para crear un usuario dedicado exclusivamente al proceso de migración:

```sql
CREATE USER replicador_migracion WITH REPLICATION PASSWORD 'TuPasswordSegura123!';
GRANT USAGE ON SCHEMA public TO replicador_migracion;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicador_migracion;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO replicador_migracion;
```

---

## 3. Preparación del Esquema en Supabase

Antes de transferir los datos, es indispensable replicar únicamente la **estructura de la base de datos (el Esquema DDL)** en tu nuevo proyecto de Supabase.

### Paso 3.1: Exportar solo la estructura (DDL) sin datos
Ejecuta en tu terminal el siguiente comando `pg_dump`:

```bash
pg_dump -h ip_servidor_origen -U mi_usuario -d mi_base_datos --schema-only --no-owner --no-privileges > esquema_base.sql
```

### Paso 3.2: Aplicar el esquema en Supabase
1. Inicia sesión en tu panel de control de **Supabase**.
2. Crea un nuevo proyecto y guarda la contraseña de la base de datos en un lugar seguro.
3. Dirígete a la sección **SQL Editor** del panel de Supabase.
4. Pega o carga el contenido de tu archivo `esquema_base.sql` y presiona **RUN**.

---

## 4. Configuración de Publicación y Suscripción

Una vez alineados los esquemas entre el servidor de origen y el destino en Supabase, iniciamos el flujo de replicación en vivo.

### Paso 4.1: Crear la Publicación en el Servidor Origen
Conéctate vía `psql` a tu base de datos de origen y define una publicación que abarque todas las tablas de tu dominio:

```sql
CREATE PUBLICATION pub_migracion_supabase FOR ALL TABLES;
```

### Paso 4.2: Crear la Suscripción en Supabase
Conéctate a tu proyecto de Supabase mediante un cliente SQL (como DBeaver, TablePlus o psql) y ejecuta el comando para suscribirte a la publicación del servidor de origen:

```sql
CREATE SUBSCRIPTION sub_migracion_origen CONNECTION 'host=ip_servidor_origen port=5432 dbname=mi_base_datos user=replicador_migracion password=TuPasswordSegura123!' PUBLICATION pub_migracion_supabase WITH (copy_data = true);
```

---

## 5. Resincronización de Secuencias (BIGSERIAL / Auto-Increment)

La replicación lógica de PostgreSQL copia los datos de las tablas, **pero no sincroniza automáticamente el estado de las secuencias** (los IDs autonumerados). Si no resincronizas las secuencias antes de mover tu aplicación a Supabase, las nuevas inserciones fallarán por duplicación de Clave Primaria.

Una vez terminada la copia de datos, ejecuta el siguiente script en el editor SQL de Supabase para ajustar todas las secuencias al valor máximo actual de cada tabla:

```sql
DO $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN 
        SELECT table_name, column_name, column_default 
        FROM information_schema.columns 
        WHERE table_schema = 'public' 
          AND column_default LIKE 'nextval%'
    LOOP
        EXECUTE format(
            'SELECT setval(pg_get_serial_sequence(%L, %L), COALESCE(MAX(%I), 1)) FROM %I',
            rec.table_name, rec.column_name, rec.column_name, rec.table_name
        );
    END LOOP;
END $$;
```

---

## 6. Monitoreo de Sincronización y Procedimiento Cutover

Para verificar la latencia de replicación y asegurar que ambas instancias tengan exactamente los mismos datos:

### Consulta de estado de la replicación
Ejecuta la consulta de estado de replicación en el servidor origen revisando la tabla `pg_stat_replication`:

```sql
SELECT client_addr, application_name, state, sync_state, (pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS bytes_retraso FROM pg_stat_replication;
```

Cuando el valor de `bytes_retraso` sea **0** o de apenas unos pocos bytes, la base de datos en Supabase estará perfectamente al día.

### Procedimiento de Conmutación Final (Cutover)
1. **Cambio de Variables de Entorno:** Actualiza la variable `DATABASE_URL` en tu hosting para apuntar a Supabase.
2. **Reinicia la Aplicación:** Reinicia tus servidores API para forzar la creación de nuevas conexiones en el pool de Supabase.
3. **Limpieza de Recursos:** Ejecuta `DROP SUBSCRIPTION sub_migracion_origen;` en Supabase y `DROP PUBLICATION pub_migracion_supabase;` en el origen.

---

## Conclusión

Migrar una infraestructura de datos en producción hacia **Supabase** utilizando Replicación Lógica es la estrategia recomendada para equipos que no pueden darse el lujo de apagar sus sistemas. Garantiza una migración transparente, medible, segura y con cero tiempo de inactividad para tus usuarios finales.