
# Análisis Exploratorio de Datos - Nacidos Vivos en Ecuador 2023

## 1. Descripción general del proyecto
Este proyecto realiza un análisis exploratorio sobre datos de nacimientos en Ecuador del año 2023 usando Apache Spark. Se trabaja con un DataFrame `dfNacidos` que proviene de un archivo CSV delimitado por `;` y contiene 241,295 registros y 47 variables, muchas de ellas demográficas, clínicas y geográficas.

---

## 2. Carga de datos
```scala
val dfNacidos = spark
  .read
  .option("header", true)
  .option("sep", ";")
  .option("inferSchema", true)
  .csv("ENV_2023.csv")
```
**Función:** Carga el archivo CSV y detecta automáticamente los tipos de datos.

---

## 3. Exploración inicial

### Conteo de filas y columnas
```scala
print(s"Registros (filas): ${dfNacidos.count}, Variables (columnas): ${dfNacidos.columns.length}")
```

### Esquema del DataFrame
```scala
dfNacidos.printSchema
```
Muestra los tipos de datos de cada columna, muchas de las cuales están inicialmente como `String`, aunque algunas numéricas (`anio_nac`, `dia_nac`, `hij_viv`) ya fueron correctamente inferidas.

---

## 4. Estadísticas descriptivas

### Estadísticas generales
```scala
dfNacidos.describe().show
```

### Filtrado de columnas numéricas
```scala
val numericCols = dfNacidos.schema.fields.filter {
  case StructField(_, IntegerType | LongType | FloatType | DoubleType | ShortType | DecimalType(), _, _) => true
  case _ => false
}.map(_.name)
```

### Estadísticas por columnas numéricas
```scala
dfNacidos.select(numericCols.map(col): _*).summary().show()
```

Incluye percentiles (25%, 50%, 75%) y desviación estándar para `anio_nac`, `dia_nac` y `hij_viv`.

---

## 5. Limpieza y transformación de fechas

### Conversión de strings a tipo `Date`
```scala
val dfNacidosClean = dfNacidos
  .withColumn("fecha_insc_date", to_date(col("fecha_insc"), "yyyy/MM/dd"))
  .withColumn("fecha_nac_date", to_date(col("fecha_nac"), "yyyy/MM/dd"))
  .withColumn("fecha_mad_date", to_date(col("fecha_mad"), "yyyy/MM/dd"))
```

---

## 6. Análisis de valores nulos
```scala
dfNacidosClean.select(
  count(when(col("fecha_insc_date").isNull, 1)).alias("nulos_insc"),
  count(when(col("fecha_nac_date").isNull, 1)).alias("nulos_nac"),
  count(when(col("fecha_mad_date").isNull, 1)).alias("nulos_mad")
).show()
```

---

## 7. Resumen estadístico de fechas
```scala
val dateCols = dfNacidosClean.schema.fields
  .filter(f => f.dataType == DateType || f.dataType == TimestampType)
  .map(_.name)

val statsDate = dateCols.flatMap { colName =>
  Seq(
    count(col(colName)).alias(s"${colName}_count"),
    min(col(colName)).alias(s"${colName}_min"),
    max(col(colName)).alias(s"${colName}_max"),
    countDistinct(col(colName)).alias(s"${colName}_countDistinct")
  )
}

val dfStatsDateCols = dfNacidosClean.select(statsDate: _*)
dfStatsDateCols.show()
```

---

## 8. Exploración de columnas numéricas mal tipadas
```scala
val columnasStringNumericas = Seq("anio_insc", "dia_insc", "talla", "peso", "sem_gest", "apgar1", "apgar5", "anio_mad", "dia_mad", "edad_mad", "con_pren", "num_emb", "num_par", "hij_vivm", "hij_nacm")
```

Se exploran sus valores distintos con `.distinct()` y `.countDistinct()`.

---

## 9. Estadísticas específicas

### Mediana, moda y desviaciones
```scala
dfNacidosClean.select(
  mode($"anio_nac"),
  median($"anio_nac"),
  stddev($"anio_nac"),
  stddev_pop($"anio_nac"),
  stddev_samp($"anio_nac")
).show()
```

---

## 10. Recomendaciones
- Convertir las columnas `String` numéricas a tipos numéricos reales (`Double` o `Int`) para análisis estadísticos más precisos.
- Eliminar o imputar valores nulos según contexto del análisis.
- Normalizar valores como `"Sin información"` antes de análisis cualitativos.

---

## 11. Conclusión
Este análisis inicial establece las bases para profundizar en temas de salud materno-infantil en Ecuador. Spark permite escalar el procesamiento sobre grandes volúmenes de datos, y este script demuestra un uso eficaz de herramientas como `describe`, `summary`, `withColumn`, y `select`.

# Analisís 2
## Estado Civil
### Distribución por estado civil

| Estado Civil   | Recién nacidos |
| -------------- | -------------- |
| Soltera        | 139,799        |
| Casada         | 56,893         |
| Unida          | 32,470         |
| Unión de hecho | 5,489          |
| Divorciada     | 4,510          |

Se observa que más de la mitad de los nacimientos registrados en Ecuador durante 2023 provienen de madres solteras (58%). Esto puede reflejar tanto cambios culturales como posibles desafíos sociales en el acompañamiento prenatal. Deberiamos realizar un cruce con las variables como la edad materna, la ubicacion geólogica de las madres y el nivel de instrucción

## Etnia de las madres
### Distribución por grupo étnico
| Etnia                              | Recién nacidos |
| ---------------------------------- | -------------- |
| Mestiza                            | 210,051        |
| Indígena                           | 14,289         |
| Sin información                    | 7,241          |
| Afroecuatoriana / Afrodescendiente | 2,747          |
| Negra                              | 2,634          |


El 87% de los nacimientos provienen de madres que se identifican como mestizas, lo cual concuerda con la distribución étnica general del país. Sin embargo, se detecta un 3% de registros sin información étnica, lo que sugiere un área de mejora en la calidad de los datos. Podemos buscar relaciones con la zona de las madres, ya sea rural o urbana y tambien podriamos ver en que institución el niño fue dado a luz, publica, privada, en casa, etc


## Nivel de Alfabetización de las Madres
### Distribución por nivel de alfabetización
| Sabe Leer       | Recién nacidos |
|-----------------|----------------|
| Sí              | 239,519        |
| No              | 1,301          |
| Sin información | 475            |

El 99.4% de los nacimientos provienen de madres que declararon saber leer, lo cual refleja una cobertura amplia en alfabetización femenina.  
Sin embargo, existen 1,301 casos (aproximadamente 0.5%) de madres que no saben leer, un grupo minoritario pero relevante desde una perspectiva de política pública.  
Además, hay 475 registros sin información, lo que representa oportunidades de mejora en la recolección de datos.  

Para tener una imagen más completa de la alfabetización de las madres, proponemos combinar estos datos con información sobre su **nivel educativo** y su **lugar de residencia**. Así podremos descubrir si existen patrones, por ejemplo, si la falta de alfabetización es más común en ciertas regiones o entre madres con menor nivel de estudios, lo que nos daría pistas importantes para futuras intervenciones.


## Provincias de Residencia
### Distribución por provincia de residencia
| Provincia                            | Nacimientos |
|--------------------------------------|-------------|
| Guayas                               | 62,640      |
| Pichincha                            | 34,259      |
| Manabí                               | 24,423      |
| Los Ríos                             | 15,233      |
| Esmeraldas                           | 11,324      |
| El Oro                               | 9,982       |
| Azuay                                | 9,645       |
| Santo Domingo de los Tsáchilas      | 8,430       |
| Tungurahua                           | 6,855       |
| Santa Elena                          | 6,547       |
| Cotopaxi                             | 6,499       |
| Imbabura                             | 6,069       |
| Chimborazo                           | 6,038       |
| Loja                                 | 5,974       |
| Morona Santiago                      | 4,440       |
| Orellana                             | 3,742       |
| Sucumbíos                            | 3,721       |
| Cañar                                | 3,251       |
| Napo                                 | 2,878       |
| Bolívar                              | 2,529       |
| Pastaza                              | 2,244       |
| Carchi                               | 2,124       |
| Zamora Chinchipe                     | 1,979       |
| Galápagos                            | 373         |
| Exterior                             | 96          |
 
Las provincias con mayor número de nacimientos registrados en 2023 fueron **Guayas** (26%) y **Pichincha** (14%), que coinciden con las zonas más pobladas del país.  
Este patrón es coherente con la distribución demográfica nacional y puede estar relacionado con el acceso a servicios de salud, urbanización y concentración de hospitales públicos.

Para entender mejor las diferencias en la salud de las madres y sus bebés a lo largo del país, sería muy valioso ver cómo cambian aspectos como el tipo de parto, el origen étnico de la madre o su edad promedio, provincia por provincia. Esto nos ayudaría a identificar dónde hay mayores necesidades y dónde los servicios de salud perinatal podrían mejorar.

