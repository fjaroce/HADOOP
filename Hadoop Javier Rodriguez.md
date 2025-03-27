# PRÁCTICA HADOOP MOVIEBUSTER

El presente documento es una memoria explicativa que contiene la solución a los diferentes ejercicios de la práctica

### 1. Parte práctica

Como punto de partida previo a la explotación de los datos, ha de crearse el entorno para ello:

- En primer lugar descargar del repositorio externo los datasets. 
- Iniciar HIVE y crear los directorios en HDFS

        
            hdfs dfs -mkdir -p /user/cloudera/moviebuster/movies
            hdfs dfs -mkdir -p /user/cloudera/moviebuster/ratings
            hdfs dfs -mkdir -p /user/cloudera/moviebuster/users

- Subir los datos de la máquina local al entorno de HDFS

                hdfs dfs -put /home/cloudera/Documents/moviebuster_data/movies.dat /user/cloudera/moviebuster/movies/
                hdfs dfs -put /home/cloudera/Documents/moviebuster_data/ratings.dat /user/cloudera/moviebuster/ratings/
                hdfs dfs -put /home/cloudera/Documents/moviebuster_data/users.dat /user/cloudera/moviebuster/users/

- Crear las tablas, para ello he optado por añadir cláusula para espicificar los separadores y otra para determinar en qué directorio se encuentran los datos referenciados de la tabla, de manera que no hay que hacer el paso propio de importación pues he indicado que el directorio físico ('tablespace') es el mismo que el directorio al que he subido los datos en hdfs.

                CREATE TABLE movies (
                    movieId INT,
                    title STRING,
                    genres STRING
                )
                ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
                WITH SERDEPROPERTIES (
                    "input.regex" = "^([^:]+)::([^:]+)::([^:]+)$"
                )
                LOCATION '/user/cloudera/moviebuster/movies';

                CREATE TABLE ratings (
                    userId INT,
                    movieId INT,
                    rating FLOAT,
                    timestamp BIGINT
                )
                ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
                WITH SERDEPROPERTIES (
                    "input.regex" = "^([^:]+)::([^:]+)::([^:]+)::([^:]+)$"
                )
                LOCATION '/user/cloudera/moviebuster/ratings';

                CREATE TABLE users (
                    userId INT,
                    gender STRING,
                    age INT,
                    occupation INT,
                    zipcode STRING
                )
                ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
                WITH SERDEPROPERTIES (
                    "input.regex" = "^([^:]+)::([^:]+)::([^:]+)::([^:]+)::([^:]+)$"
                )
                LOCATION '/user/cloudera/moviebuster/users';

- Una vez he verificado que los datos se han importado correctamente y ya están alojado en las tablas, procedo a la realización de las consultas:

1.1 Ejercicio #1 

1.  ¿Cuál es la película con más opiniones?

        SELECT m.title, COUNT(r.rating) as num_ratings
        FROM movies m
        JOIN ratings r ON m.movieId = r.movieId
        GROUP BY m.title
        ORDER BY num_ratings DESC
        LIMIT 1;

    Resultado:
        American Beauty (1999)	3428


2.  ¿Qué 10 usuarios son los más activos a la hora de puntuar películas? 

        SELECT u.userId, COUNT(r.rating) as num_ratings
        FROM users u
        JOIN ratings r ON u.userId = r.userId
        GROUP BY u.userId
        ORDER BY num_ratings DESC
        LIMIT 10;
    
    Resultado:
        4169	2314
        1680	1850
        4277	1743
        1941	1595
        1181	1521
        889	    1518
        3618	1344
        2063	1323
        1150	1302
        1015	1286


3.  ¿Cuáles son las tres mejores películas según los scores? ¿Y las tres peores?

        --Tres mejores películas
        SELECT m.title, AVG(r.rating) as avg_score, COUNT(r.rating) as num_ratings
        FROM movies m
        JOIN ratings r ON m.movieId = r.movieId
        GROUP BY m.title
        HAVING num_ratings > 10 -- QUITAR PELICULAS CON POCAS OPINIONS 
        ORDER BY avg_score DESC
        LIMIT 3;
    
    Resultado(title):
        Sanjuro (1962)
        Seven Samurai (The Magnificent Seven) (Shichinin no samurai)
        Shawshank Redemption, The (1994)


        --Tres peores películas
        SELECT m.title, AVG(r.rating) as avg_score, COUNT(r.rating) as num_ratings
        FROM movies m
        JOIN ratings r ON m.movieId = r.movieId
        GROUP BY m.title
        HAVING num_ratings > 10 --QUITAR PELICULAS CON POCAS OPINIONS 
        ORDER BY avg_score ASC
        LIMIT 3;

    Resultado(title):
        Amityville 3-D (1983)
        Carnosaur 2 (1995)	
        Kazaam (1996)

4. ¿Hay alguna profesión en la que deberíamos enfocar nuestros esfuerzos en publicidad? ¿Por qué?   

        SELECT u.occupation, COUNT(DISTINCT u.userId) as num_users, 
            COUNT(r.rating) as num_ratings,
            AVG(r.rating) as avg_rating
        FROM users u
        JOIN ratings r ON u.userId = r.userId
        GROUP BY u.occupation
        ORDER BY num_ratings DESC;

    Recomendaría enfocar los esfuerzos publicitarios principalmente en estudiantes universitarios/graduados (código 4) por las siguientes razones:

    1. Mayor volumen: Este grupo representa el mayor número de usuarios activos y ha generado la mayor cantidad de ratings, lo que indica un alto nivel de compromiso con la plataforma.
    2. Potencial de crecimiento: Los estudiantes universitarios están formando sus hábitos de consumo , que podrían mantener a largo plazo.
    3. Efecto viral: Este grupo tiene más probabilidades de compartir recomendaciones entre sus círculos sociales, aumentando la visibilidad orgánica.
    4. Receptividad a nuevas tecnologías: Los estudiantes universitarios suelen ser early adopters de nuevas plataformas y servicios digitales.


5. ¿Se te ocurre algún otro insight valioso que pudiéramos extraer de los datos procesados? ¿Cómo? 

    Me parece que sería interesante la realización de un análisis de ratings por edad, que además complementa la información de la consulta anterior.

        SELECT 
        CASE
            WHEN u.age < 18 THEN 'Menos de 18'
            WHEN u.age BETWEEN 18 AND 24 THEN '18-24'
            WHEN u.age BETWEEN 25 AND 34 THEN '25-34'
            WHEN u.age BETWEEN 35 AND 44 THEN '35-44'
            WHEN u.age BETWEEN 45 AND 54 THEN '45-54'
            ELSE 'Más de 55'
        END as grupo_edad,
        COUNT(r.rating) as num_ratings,
        AVG(r.rating) as rating_promedio
    FROM 
        users u
    JOIN 
        ratings r ON u.userId = r.userId
    GROUP BY 
        CASE
            WHEN u.age < 18 THEN 'Menos de 18'
            WHEN u.age BETWEEN 18 AND 24 THEN '18-24'
            WHEN u.age BETWEEN 25 AND 34 THEN '25-34'
            WHEN u.age BETWEEN 35 AND 44 THEN '35-44'
            WHEN u.age BETWEEN 45 AND 54 THEN '45-54'
            ELSE 'Más de 55'
        END
    ORDER BY 
        rating_promedio DESC;


    Este análisis es valioso porque permite a MovieBuster entender cómo difieren las preferencias por edad, para por ejemplo facilitar el desarrollo de estrategias de marketing dirigidas por edad. Además, ayuda a identificar qué grupo de edad es más crítico o más generoso en sus valorciones, para favorecer que ciertos grupos tengan mayor facilidad para realizar valoraciones. 
    También puede orientar la selección de contenido para captar audiencias específicas

1.2 Ejercicio #2 

Para la ejecución del ejercicio 2 el primer paso es crear las tablas con el contenido de las consultas previamenete realizadas:

    CREATE TABLE top_movie_by_opinions AS
    SELECT m.title, COUNT(r.rating) as num_ratings
    FROM movies m
    JOIN ratings r ON m.movieId = r.movieId
    GROUP BY m.title
    ORDER BY num_ratings DESC
    LIMIT 1;

    CREATE TABLE top_active_users AS
    SELECT u.userId, COUNT(r.rating) as num_ratings
    FROM users u
    JOIN ratings r ON u.userId = r.userId
    GROUP BY u.userId
    ORDER BY num_ratings DESC
    LIMIT 10;

Una vez creadas las tablas en Hive, creo la base de datos en MySQL y las tablas que van a contener la información de las consultas anteriores:

    CREATE DATABASE moviebuster;
    USE moviebuster;

    CREATE TABLE top_movie_by_opinions (
    title VARCHAR(255),
    num_ratings INT,
    PRIMARY KEY (title)
    );

    CREATE TABLE top_active_users (
    userId INT,
    num_ratings INT,
    PRIMARY KEY (userId)
    );

Una vez creadas, solo queda hacer la exporación/importación de los datos mediante los siguientes comandos:

    sqoop export \
    --connect "jdbc:mysql://localhost/moviebuster" \
    --username root \
    --password cloudera \
    --table top_movie_by_opinions \
    --export-dir /user/hive/warehouse/moviebuster.db/top_movie_by_opinions \
    --input-fields-terminated-by '\001' \
    --direct \
    --m 1


    sqoop export \
    --connect "jdbc:mysql://localhost/moviebuster" \
    --username root \
    --password cloudera \
    --table top_active_users \
    --export-dir /user/hive/warehouse/moviebuster.db/top_active_users \
    --input-fields-terminated-by '\001' \
    --direct \
    --m 1

Existen diferentes motivos para crear estas tablas y copiar los datos a MySQL en lugar de acceder directamente a los resultados de Hive, destacar:

   1. Eficiencia: Las consultas a Hadoop son computacionalmente mucho más costosas y lentas para una interfaz web interactiva. La web necesita tiempos de respuesta de milisegundos y Hadoop no 
   no puede garantizar esta velocidad.
   2. Sepración de entornos, es una buena práctica tener separado el entorno analítico (en este caso Hadoop) del entorno operativo (la web), esto tiene diversas ventajas, como por ejemplo no   saturar el cluster con consultas repetitivas evitando hacer un incorrecto uso de hadoop además de facilitar la integración de los datos con la web gracias a mysql
   


### 2. Teoría - Dimensionamiento clúster Hadoop

Para calcular el número de máquinas necesarias para poder almacenar todo el volumen de datos durante el próximo año primero hay que calcular el volumen de datos diario de las diferentes fuentes:

| Fuente  | Eventos por día | Tamaño por evento | Total Diario |
|---------|---------------|------------------|-------------|
| Fuente 1 | 10.000        | 15 KB            | 150 MB     |
| Fuente 2 | 120.000       | 300 B            | 36 MB      |
| Fuente 3 | 150.000       | 100 KB           | 15 GB      |
| Fuente 4 | 170.000       | 800 KB           | 136 GB     |
| Fuente 5 | 2.000         | 1500 KB          | 3 GB       |
| **Total** | **—**         | **—**            | **154,2 GB** |

Con el volumen de datos diario, he calculado el volumen de datos anual:

    154.2(GB/días) * 365(días) = 56.783(GB) = 56,78(TB)

Teniendo en cuenta el factor de replicación de Hadoop (3), pasan a ser 170,34 TB

Sabiendo que cada máquina tiene 22 discos de 2TB, es decir 44TB y teniendo en cuenta el volumen total de datos:

    170,34/44 = 3,84

Conclusión:  asumiendo un factor 3 de replicación, hacen falta 4 máquinas para asegurar el almacenamiento anual.


### 3. Estimación arquitecturas Hadoop

Herramienta de BI → Apache Impala para consultas en tiempo real y en general explotación de los datos. Como ventaja destacar que tiene muy baja latencia para consultas interactivas y proporciona compatibilidad con conectores JDBC/ODBC para integración directa con otras herramientas. Como principal desventaja, el elevado consumo de RAM, además de que su rendimiento se ve degradado con alta concurrencia de usuarios y no soporta transacciones ACID completas. En mi opinión, Impala es la mejor opción para entornos de BI porque está específicamente optimizada para consultas analíticas interactivas, ño que prmite a los usuarios realizar exploraciones de datos dinámicas con tiempos de respuesta casi inmediatos, lo que  es esencial para una buena experiencia de usuario en dashboards y reportes interactivos.

Web de consultas → HBase + Phoenix como alternativa para implementar una web de consultas sobre pedidos realizados. La principal ventaja de esta arquitectura es que combina el rendimiento de HBase para operaciones de lectura/escritura por clave (crucial para búsquedas de pedidos específicos) con la capa SQL que proporciona Phoenix, facilitando la  integración con aplicaciones web. HBase ofrece tiempos de respuesta en milisegundos para búsquedas por clave primaria y escalabilidad horizontal conforme crezca el volumen de pedidos. Como inconvenientes, destacar la complejidad en el diseño del modelo de datos (crítico para el rendimiento) y que no está optimizado para consultas analíticas complejas que involucren múltiples agregaciones. Esta solución es superior a mantener los datos en una base de datos relacional como MySQL, ya que proporciona escalabilidad nativa dentro del ecosistema Hadoop, manteniendo los datos cerca de donde se procesarán posteriormente para análisis más complejos.

Generación informes SQL usando R → Hacer uso directamente de Apache Hive con RHive ofrece buen equilibrio entre capacidad de procesamiento y eficiencia de recursos. Como ventaja principal, Hive está específicamente optimizado para consultas batch programadas, siendo ideal para informes mensuales donde la latencia no es un factor crítico. Además, la integración con R mediante RHive permite combinar el poder del procesamiento distribuido de Hadoop con las capcidades analíticas y estadísticas de R, sin necesidad de extraer los datos del ecosistema. Como inconvenientes, Hive presenta mayor latencia en la ejecución comparado con soluciones como Impala (minutos vs segundos), aunque no debe ser inconveniente para informes programados mensualmente. Para la ejecución automatizada, puede integrarse fácilmente con Apache Oozie, que proporciona un sistema robusto de programación de workflows con capacidades de recuperación ante fallos y dependencias entre tareas. 

Recopilación de información de redes sociales → Apache Flume + HDFS/HBase representa la solución óptima para capturar y procesar datos de redes sociales dentro del ecosistema Hadoop. Flume está específicamente diseñado para la recolección, agregación y movimiento de grandes volúmenes de datos de logs y eventos, lo que lo hace naturalmente apropiado para la captura de datos de redes sociales. Como principales ventajas, Flume ofrece una arquitectura  flexible y configurable que permite adaptarse a múltiples fuentes de datos, tiene baja sobrecarga de procesamiento y alta fiabilidad gracias a mecanismos  de recuperación ante fallos. Los datos capturados pueden almacenarse directamente en HDFS para análisis posteriores o en HBase si se requiere acceso en tiempo real. Como inconvenientes, Flume presenta capacidades limitadas de transformación en tiempo real y menos flexibilidad para procesamientos complejos comparado con otas soluciones como Kafka + Spark.