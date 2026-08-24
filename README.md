# Yield monitor data cleaning

Scripts en R para la limpieza y preparación de datos brutos procedentes de monitores de rendimiento de cosechadoras, especialmente orientados a ensayos en finca y comparaciones entre estrategias de manejo agrícola.

Este repositorio recoge un flujo de trabajo reproducible para depurar datos espaciales de rendimiento antes de su análisis agronómico. La metodología parte del procedimiento automatizado propuesto por Vega et al. y lo complementa con filtros contextuales inspirados en Marchant et al.

El flujo incluido en este repositorio corresponde a una versión mejorada respecto a la propuesta inicialmente descrita en la publicación **“Depuración de Datos de Monitores de Rendimiento de Cosechadoras: una Herramienta para Comparar Estrategias de Campo”**. Las principales mejoras incorporadas son el uso obligatorio de sistemas de referencia proyectados en metros, la exportación en formato GeoPackage y la incorporación de una comprobación local robusta basada en IQR para reducir la eliminación de variabilidad agronómica válida.

## Objetivo

El objetivo del flujo es eliminar registros erróneos o poco fiables de los datos brutos del monitor de rendimiento, conservando al mismo tiempo la variabilidad espacial real del cultivo.

El procedimiento está diseñado para ser aplicado por personal técnico, agricultores o investigadores que trabajen con datos de cosechadora en ensayos en finca, agricultura de precisión o evaluaciones comparativas de estrategias de manejo.

## Requisitos principales

Los datos de entrada deben cumplir los siguientes requisitos:

- Las capas espaciales deben estar en un CRS proyectado en metros.
- No debe ejecutarse el flujo con EPSG:4326 / WGS 84.
- Las capas utilizadas conjuntamente deben estar en el mismo CRS.
- Las salidas vectoriales se generan en formato GeoPackage (.gpkg).
- El campo de rendimiento y, cuando proceda, los campos de velocidad o dirección deben existir en la tabla de atributos.

El uso de GeoPackage evita problemas habituales del formato shapefile, como el recorte o modificación de nombres de columnas.

## Orden general de ejecución

El flujo de limpieza está organizado en varios scripts:

1. **Filtros 1, 2, 3 y 4**  
   Protocolo base automatizado inspirado en Vega et al.  
   Incluye eliminación de rendimientos nulos o no válidos, puntos próximos al borde, valores extremos globales, Local Moran’s I y Cook’s D.

2. **Filtro 5**  
   Eliminación de puntos próximos a caminos interiores.

3. **Filtro 6**  
   Eliminación de puntos próximos a líneas de separación entre tratamientos o estrategias.

4. **Filtro 7**  
   Eliminación de registros con velocidad de cosechadora no operativa o fuera del rango esperado.

5. **Filtro 8**  
   Eliminación de registros con direcciones de avance no esperadas.

Extra. **Gráficos de auditoría**  
   Generación de gráficos comparativos entre la capa anterior y posterior a cada filtro.

Algunos filtros contextuales pueden omitirse si no son aplicables a una parcela concreta. Por ejemplo, el Filtro 5 puede omitirse si no existen caminos interiores en el campo o el Filtro 7 de velocidad si los registros que elimina son similares a los anteriores y posteriores según el orden de cosecha.

## Identificador row_number

El flujo genera y conserva un identificador único denominado row_number.

Este campo permite comparar capas antes y después de cada filtro, identificando qué puntos se han mantenido y cuáles se han eliminado. Es especialmente importante para los gráficos de auditoría.

El archivo generado por los Filtros 1, 2, 3 y 4 incluye una copia de la capa original en GeoPackage con row_number, que debe utilizarse como capa inicial cuando se quieran representar los puntos eliminados por el primer bloque de filtros.

## Criterio local basado en IQR y regla de Tukey

Los pasos basados en Local Moran’s I y Cook’s D permiten detectar puntos espacialmente anómalos o influyentes. Sin embargo, en datos agrícolas no todo punto espacialmente diferente debe considerarse necesariamente erróneo. Parte de esa variabilidad puede reflejar diferencias reales de suelo, cultivo, manejo o rendimiento.

Por este motivo, en esta versión se añade una condición local robusta basada en el rango intercuartílico, o IQR.

El IQR se calcula como:

```text
IQR = Q3 - Q1
```

donde `Q1` es el primer cuartil y `Q3` el tercer cuartil.

A partir de ese valor se aplica la regla clásica de Tukey:

```text
Límite inferior = Q1 - 1.5 × IQR
Límite superior = Q3 + 1.5 × IQR
```

En este flujo, esa regla se aplica localmente, usando el vecindario espacial de cada punto. De este modo, los puntos detectados por Local Moran’s I o Cook’s D se consideran primero candidatos a eliminación, pero solo se eliminan si además son extremos respecto a su entorno local.

Esta doble condición hace que el filtrado sea más conservador y reduce el riesgo de eliminar variabilidad espacial válida.

## Formato de salida

Las salidas principales del flujo se generan en formato GeoPackage:

```text
yield_original_row_number.gpkg
yield_cleaned.gpkg
yield_no_roads.gpkg
yield_no_treatchange.gpkg
yield_speed_filtered.gpkg
yield_unexpected_direction.gpkg
```

El formato shapefile puede usarse como entrada, pero no se recomienda como salida, ya que puede modificar los nombres de los campos de la tabla de atributos.

## Advertencias importantes

Antes de ejecutar los filtros, se recomienda revisar que:

- El CRS de todas las capas sea proyectado y esté en metros.
- Las capas que se comparan estén en el mismo CRS.
- Las capas de caminos y líneas entre tratamientos sean realmente capas de líneas.
- Las capas convertidas desde polígonos se revisen antes de usarse, ya que pueden incluir perímetros exteriores u otras líneas no deseadas.
- Los nombres de los campos de rendimiento, velocidad y dirección coincidan con los definidos en cada script.
- El Filtro 5 solo debe aplicarse cuando existan caminos interiores u otras líneas equivalentes que justifiquen su uso.
- El Filtro 6 debe aplicarse con líneas de separación entre tratamientos, no con polígonos.
- El Filtro 7 de velocidad no debe aplicarse si los registros que elimina son similares en rendimiento a los anteriores y posteriores, según el orden de cosecha.

## Uso previsto

Este repositorio está pensado para apoyar la limpieza reproducible de datos de rendimiento en agricultura de precisión y ensayos en finca. Los scripts pueden adaptarse a diferentes monitores, cultivos y parcelas, siempre que se revisen adecuadamente los nombres de campos, unidades y capas espaciales de entrada.

## Referencias metodológicas

Este flujo se basa en el procedimiento automatizado propuesto por Vega et al. para la depuración de datos de monitores de rendimiento, y se complementa con filtros contextuales inspirados en Marchant et al.

También toma como punto de partida la publicación **“Depuración de Datos de Monitores de Rendimiento de Cosechadoras: una Herramienta para Comparar Estrategias de Campo”**, incorporando mejoras prácticas derivadas de su aplicación en ensayos en finca.
