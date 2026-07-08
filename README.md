# Proyecto_reporte_Premios_prueba_tecnica

Introducción del Proyecto: Proyecto_Premios
El presente proyecto, denominado Proyecto_Premios, tiene como propósito el diseño e implementación de una solución integral de inteligencia de negocios (Business Intelligence) orientada al análisis y gestión de información relacionada con premios. Para su desarrollo se empleó un conjunto de herramientas y tecnologías de Microsoft ampliamente utilizadas en el ámbito empresarial para el manejo de datos, la automatización de procesos ETL, el modelado analítico y la generación de reportes interactivos.
En primer lugar, se utilizó SQL Server como motor de base de datos relacional, permitiendo el almacenamiento, estructuración y administración eficiente de la información fuente del proyecto. Sobre esta base, se trabajó con Visual Studio como entorno de desarrollo integrado (IDE), el cual facilitó la creación y gestión de los distintos proyectos de Business Intelligence a través de las herramientas de SQL Server Data Tools (SSDT).
Para la extracción, transformación y carga de datos (ETL), se implementó SQL Server Integration Services (SSIS), lo que permitió automatizar los procesos de consolidación de información proveniente de distintas fuentes, garantizando su limpieza, transformación y disponibilidad para su posterior análisis.
Posteriormente, mediante SQL Server Analysis Services (SSAS), se construyó un modelo multidimensional (cubo OLAP) que permitió organizar la información en dimensiones y medidas, facilitando el análisis eficiente de los datos relacionados con los premios, sus categorías, beneficiarios, periodos y demás variables relevantes para la toma de decisiones.
Finalmente, con el fin de comunicar los resultados obtenidos de manera clara, visual y accesible, se desarrollaron reportes interactivos en Power BI, herramienta que permitió la creación de dashboards dinámicos con indicadores clave (KPIs), gráficos y filtros, facilitando la interpretación de la información y apoyando la toma de decisiones estratégicas.
De esta manera, el Proyecto_Premios integra de forma completa el ciclo de vida de una solución de inteligencia de negocios: desde el almacenamiento y transformación de los datos, pasando por su modelado analítico, hasta la visualización final de resultados, evidenciando la aplicación práctica de herramientas del ecosistema Microsoft BI en un caso de uso real.

#DIAGRAMA PREMIOS

<img width="1210" height="812" alt="image" src="https://github.com/user-attachments/assets/070213f9-3ff0-4c93-a8de-d0b34096d597" />

Cómo integrarlas en el paquete:

Dim_Fecha — se genera una sola vez con una CTE recursiva (rango 2020-2030). Va en un Execute SQL Task, no en un Data Flow, porque no depende de datos transaccionales.
Dim_Cliente, Dim_Premio, Dim_Sorteo, Dim_Comercio, Dim_Usuario — cada bloque "ORIGEN" es la query que pones en el OLE DB Source (modo "SQL Command") dentro de un Data Flow Task.
Orden de carga: primero todas las dimensiones, después fact_ganadores y fact_transacciones, porque los hechos resuelven sus llaves subrogadas contra las dimensiones ya cargadas (por eso llevan INNER JOIN a dim_*).

Dos estrategias para evitar duplicados en cargas repetidas:

Dentro del Data Flow: usa un Lookup Transformation contra la dimensión destino comparando la llave natural (ej. cod_premio), y enruta solo el "No Match Output" al OLE DB Destination.
Fuera del Data Flow: usa las sentencias MERGE que incluí para cada dimensión, dentro de un Execute SQL Task — hacen upsert (insertan lo nuevo, actualizan lo cambiado) en un solo paso.


1.DATA FLOW CONJUNTO DE DATOS 

<img width="658" height="385" alt="image" src="https://github.com/user-attachments/assets/25505bb2-6b51-4fea-a6c0-7ba7b4f98ca6" />

2.DATA ORIGEN Y DESTINO 

<img width="667" height="387" alt="image" src="https://github.com/user-attachments/assets/0fb7a569-c29a-4e76-9734-e58ce8d6be01" />

3.DATA ORIGEN OLE DB 

<img width="756" height="650" alt="image" src="https://github.com/user-attachments/assets/9c13db4e-8737-473b-9eca-e027839efb9d" />

4.DATA DESTINO OLE DB 

<img width="750" height="630" alt="image" src="https://github.com/user-attachments/assets/91b9f180-d748-4e45-8890-dd6f138c5b14" />



/* ============================================================================
   QUERIES DE CARGA (ETL) PARA SQL SERVER INTEGRATION SERVICES (SSIS)
   Cubo de Sorteos y Premios

   Cómo usarlas en SSIS:
   - Cada bloque "ORIGEN" es la query que va en un componente OLE DB Source
     (Data Access Mode = "SQL Command") dentro de un Data Flow Task.
   - Cada dimensión debe cargarse ANTES que las tablas de hechos, porque los
     hechos necesitan las llaves subrogadas (id_*_dim) vía Lookup.
   - Para evitar duplicados en cargas repetidas, usa un Lookup Transformation
     contra la dimensión de destino (comparando la llave natural) y así
     redirigir solo los registros "No Match Output" al OLE DB Destination.
     Alternativamente, usa las sentencias MERGE incluidas al final de cada
     bloque dentro de un Execute SQL Task.
   ============================================================================ */




/* ============================================================================
   1. DIM_FECHA
   Esta dimensión normalmente NO se llena desde el origen transaccional,
   sino que se genera una sola vez cubriendo el rango de fechas necesario.
   Se puede ejecutar como Execute SQL Task (no necesita Data Flow).
   ============================================================================ */

  

DECLARE @FechaInicio DATE = '2020-01-01';
DECLARE @FechaFin    DATE = '2030-12-31';

;WITH Calendario AS (
    SELECT @FechaInicio AS fecha
    UNION ALL
    SELECT DATEADD(DAY, 1, fecha)
    FROM Calendario
    WHERE fecha < @FechaFin
)
INSERT INTO [dbo].[dim_fecha]
    ([id_fecha], [fecha], [dia], [mes], [nombre_mes], [trimestre], [anio],
     [dia_semana], [nombre_dia_semana], [es_fin_semana])
SELECT
    CONVERT(INT, FORMAT(fecha, 'yyyyMMdd'))              AS id_fecha,
    fecha,
    DAY(fecha)                                           AS dia,
    MONTH(fecha)                                          AS mes,
    DATENAME(MONTH, fecha)                                AS nombre_mes,
    DATEPART(QUARTER, fecha)                              AS trimestre,
    YEAR(fecha)                                           AS anio,
    DATEPART(WEEKDAY, fecha)                              AS dia_semana,
    DATENAME(WEEKDAY, fecha)                              AS nombre_dia_semana,
    CASE WHEN DATEPART(WEEKDAY, fecha) IN (1,7) THEN 1 ELSE 0 END AS es_fin_semana
FROM Calendario
OPTION (MAXRECURSION 0);

/* ============================================================================
   2. DIM_CLIENTE
   ORIGEN (query para el OLE DB Source en SSIS):
   Unifica clientes de transacciones_clientes y ganadores (dimensión conformada).
   ============================================================================ */

-- ORIGEN:
SELECT cod_cliente, nombre_cliente
FROM (
    SELECT cod_cliente, nombre_cliente FROM [dbo].[transacciones_clientes]
    UNION
    SELECT cod_clientes AS cod_cliente, nombre_cliente FROM [dbo].[ganadores]
) AS clientes
GROUP BY cod_cliente, nombre_cliente;

-- CARGA INCREMENTAL (alternativa vía MERGE, en Execute SQL Task):
MERGE [dbo].[dim_cliente] AS destino
USING (
    SELECT cod_cliente, nombre_cliente
    FROM (
        SELECT cod_cliente, nombre_cliente FROM [dbo].[transacciones_clientes]
        UNION
        SELECT cod_clientes AS cod_cliente, nombre_cliente FROM [dbo].[ganadores]
    ) AS c
    GROUP BY cod_cliente, nombre_cliente
) AS origen
ON destino.cod_cliente = origen.cod_cliente
WHEN MATCHED AND destino.nombre_cliente <> origen.nombre_cliente THEN
    UPDATE SET destino.nombre_cliente = origen.nombre_cliente
WHEN NOT MATCHED BY TARGET THEN
    INSERT (cod_cliente, nombre_cliente)
    VALUES (origen.cod_cliente, origen.nombre_cliente);

/* ============================================================================
   3. DIM_PREMIO
   ============================================================================ */

-- ORIGEN:
SELECT cod_premio, descripcion, esactivo
FROM [dbo].[premios];

-- CARGA INCREMENTAL:
MERGE [dbo].[dim_premio] AS destino
USING (SELECT cod_premio, descripcion, esactivo FROM [dbo].[premios]) AS origen
ON destino.cod_premio = origen.cod_premio
WHEN MATCHED AND (destino.descripcion <> origen.descripcion OR destino.esactivo <> origen.esactivo) THEN
    UPDATE SET destino.descripcion = origen.descripcion,
               destino.esactivo    = origen.esactivo
WHEN NOT MATCHED BY TARGET THEN
    INSERT (cod_premio, descripcion, esactivo)
    VALUES (origen.cod_premio, origen.descripcion, origen.esactivo);

/* ============================================================================
   4. DIM_SORTEO
   ============================================================================ */

-- ORIGEN:
SELECT cod_sorteo, nombre, fecha_ejecucion, fecha_rango_incial AS fecha_rango_inicial,
       fecha_rango_final, fecha_creacion, esactivo
FROM [dbo].[sorteo];

-- CARGA INCREMENTAL:
MERGE [dbo].[dim_sorteo] AS destino
USING (
    SELECT cod_sorteo, nombre, fecha_ejecucion, fecha_rango_incial AS fecha_rango_inicial,
           fecha_rango_final, fecha_creacion, esactivo
    FROM [dbo].[sorteo]
) AS origen
ON destino.cod_sorteo = origen.cod_sorteo
WHEN MATCHED THEN
    UPDATE SET destino.nombre              = origen.nombre,
               destino.fecha_ejecucion     = origen.fecha_ejecucion,
               destino.fecha_rango_inicial = origen.fecha_rango_inicial,
               destino.fecha_rango_final   = origen.fecha_rango_final,
               destino.fecha_creacion      = origen.fecha_creacion,
               destino.esactivo            = origen.esactivo
WHEN NOT MATCHED BY TARGET THEN
    INSERT (cod_sorteo, nombre, fecha_ejecucion, fecha_rango_inicial, fecha_rango_final, fecha_creacion, esactivo)
    VALUES (origen.cod_sorteo, origen.nombre, origen.fecha_ejecucion, origen.fecha_rango_inicial,
            origen.fecha_rango_final, origen.fecha_creacion, origen.esactivo);

/* ============================================================================
   5. DIM_COMERCIO
   ============================================================================ */

-- ORIGEN:
SELECT DISTINCT comercio
FROM [dbo].[transacciones_clientes];

-- CARGA INCREMENTAL:
MERGE [dbo].[dim_comercio] AS destino
USING (SELECT DISTINCT comercio FROM [dbo].[transacciones_clientes]) AS origen
ON destino.comercio = origen.comercio
WHEN NOT MATCHED BY TARGET THEN
    INSERT (comercio) VALUES (origen.comercio);

/* ============================================================================
   6. DIM_USUARIO
   ============================================================================ */

-- ORIGEN:
SELECT id_usuario, nombre_user, esactivo
FROM [dbo].[usuario];

-- CARGA INCREMENTAL:
MERGE [dbo].[dim_usuario] AS destino
USING (SELECT id_usuario, nombre_user, esactivo FROM [dbo].[usuario]) AS origen
ON destino.id_usuario = origen.id_usuario
WHEN MATCHED AND destino.esactivo <> origen.esactivo THEN
    UPDATE SET destino.esactivo = origen.esactivo
WHEN NOT MATCHED BY TARGET THEN
    INSERT (id_usuario, nombre_user, esactivo)
    VALUES (origen.id_usuario, origen.nombre_user, origen.esactivo);

/* ============================================================================
   7. FACT_GANADORES
   Se carga DESPUÉS de las dimensiones. La query resuelve las llaves
   subrogadas (id_*_dim) haciendo JOIN contra cada dimensión.
   ============================================================================ */

-- ORIGEN:
SELECT
    df.id_fecha,
    ds.id_sorteo_dim,
    dp.id_premio_dim,
    dc.id_cliente_dim,
    sp.cod_sorteo_premio,
    sp.maximo_ganadores,
    1 AS cantidad_ganadores
FROM [dbo].[ganadores] g
INNER JOIN [dbo].[sorteo_premio] sp ON sp.cod_sorteo_premio = g.cod_sorteo_premio
INNER JOIN [dbo].[sorteo] s         ON s.cod_sorteo = sp.cod_sorteo
INNER JOIN [dbo].[dim_sorteo]  ds   ON ds.cod_sorteo = s.cod_sorteo
INNER JOIN [dbo].[dim_premio]  dp   ON dp.cod_premio = sp.cod_premio
INNER JOIN [dbo].[dim_cliente] dc   ON dc.cod_cliente = g.cod_clientes
INNER JOIN [dbo].[dim_fecha]   df   ON df.fecha = s.fecha_ejecucion;

/* ============================================================================
   8. FACT_TRANSACCIONES
   ============================================================================ */

-- ORIGEN:
SELECT
    df.id_fecha,
    dc.id_cliente_dim,
    dcom.id_comercio_dim,
    t.numero_autorizacion,
    t.monto_transac,
    1 AS cantidad_transac
FROM [dbo].[transacciones_clientes] t
INNER JOIN [dbo].[dim_cliente]  dc   ON dc.cod_cliente = t.cod_cliente
INNER JOIN [dbo].[dim_comercio] dcom ON dcom.comercio = t.comercio
INNER JOIN [dbo].[dim_fecha]    df   ON df.fecha = t.fecha_transac;


#

#CUBO PREMIOS

<img width="949" height="871" alt="image" src="https://github.com/user-attachments/assets/2f5230de-7977-453b-8edb-1fd7d667dbd0" />


Guía Power BI — Dashboard de Sorteos y Premios

Página 1 — Resumen ejecutivo


Tarjetas: Total Ganadores, Sorteos Activos, Monto Total Transacciones, % Cupo Utilizado
Gráfico de líneas: Total Ganadores por mes (eje = dim_fecha[nombre_mes])
Gráfico de barras horizontales: Top 5 premios por cantidad de ganadores
<img width="749" height="591" alt="image" src="https://github.com/user-attachments/assets/0cfdd2a3-ea1e-4476-86c9-e225bb124dae" />


Página 2 — Análisis de sorteos


Tabla matricial: Sorteo × Premio, valores = Total Ganadores y % Cupo Utilizado
Gráfico de barras apiladas: cupo máximo vs. ganadores reales por sorteo
Segmentador (slicer): dim_sorteo[esactivo], rango de fecha_ejecucion
<img width="749" height="630" alt="image" src="https://github.com/user-attachments/assets/03bcef4b-0aa1-4ee3-b61a-1e6d26790d81" />


Página 3 — Análisis de clientes


Tabla: Top clientes por número de premios ganados
Tarjeta: Clientes Únicos Ganadores
Gráfico de dispersión: monto transaccionado vs. cantidad de premios ganados por cliente (para ver si hay relación)
<img width="763" height="557" alt="image" src="https://github.com/user-attachments/assets/9293eb6a-6b49-47e0-a860-9efd0bd82127" />


Página 4 — Análisis de comercios


Gráfico de barras: Monto Total Transacciones por comercio
Gráfico de líneas: tendencia de transacciones en el tiempo
Tarjeta: Ticket Promedio
<img width="768" height="688" alt="image" src="https://github.com/user-attachments/assets/d95b00a5-4cf8-4de9-9058-54b3868caaac" />




