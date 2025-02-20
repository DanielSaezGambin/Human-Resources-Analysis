> **Base de datos utilizada:** `mi_base_de_datos`
> **Descripción:** Conjunto de datos de clientes con información de compras.

## 1. **Renombramos la tabla "human resources" para evitar problemas con los espacios:**

   Utilizamos el siguiente comando SQL para renombrar la tabla:

   ```sql
   ALTER TABLE `proyecto`.`human resources` 
   RENAME TO `proyecto`.`human_resources`;
   ```

## 2. **Verificamos el número de registros de la tabla human_resources:**

Usamos el siguiente comando SQL para contar las filas:

```sql 
SELECT count(*) FROM human_resources;
```

Esto nos da un total de *22,214* registros.

## 3. **Comprobamos la estructura de la tabla:**

![alt text](image.png)

Un vistazo rápido revela algunos problemas en la codificación de los datos.

## 4. **Recodificaciones**

### 4.1 **Columna id**
   
   Puesto que será nuestra clave primaria, comenzamos comprobando que no existen valores nulos:

```sql 
   SELECT * FROM human_resources WHERE id IS NULL;
```

Por el mismo motivo comprobamos que no haya más de un id duplicado:

```sql
    SELECT count(id) "id_duplicados" FROM human_resources 
    GROUP BY id 
    HAVING count(id)>1;
```

Definimos id como primary key, con valores únicos y no nulos (además debemos cambiar su tipo de dato de texto a varchar):

```sql
    ALTER TABLE human_resources 
	MODIFY id VARCHAR(20) NOT NULL UNIQUE, 
	ADD PRIMARY KEY (id);
```

### 4.2 **Columnas first_name y last_name**

Comprobamos que no hay dos registros con nombre o apellidos vacíos:

```sql
    SELECT * FROM human_resources 
    WHERE first_name OR last_name IS NULL;
```  

Modificamos estas variables:

```sql
    ALTER TABLE human_resources 
	MODIFY first_name VARCHAR(30) NOT NULL, 
    MODIFY last_name VARCHAR(60) NOT NULL, 
```

### 4.3 **Columnas birthday y hire_date**

![alt text](image-1.png)

La columna birthday muestra fechas formato MM-DD-AAAA, y del tipo MM/DD/AAAA, con el siguiente código recodificamos ambas opciones al tipo DD-MM-AAAA. 

**NOTA: Al tratarse de un cambio de datos de cierta sensibilidad, se utilizará siempre en un contexto de TRANSACTION, realizando el commit si los cambios se realizan de manera satisfactoria y un rollback si encontramos fallas en la propuesta.**

```sql
UPDATE human_resources
SET birthdate = CASE
    WHEN birthdate LIKE '%/%' THEN DATE_FORMAT(STR_TO_DATE(birthdate, '%m/%d/%Y'), '%Y-%m-%d')
    WHEN birthdate LIKE '%-%' THEN DATE_FORMAT(STR_TO_DATE(birthdate, '%m-%d-%Y'), '%Y-%m-%d')
    ELSE birthdate  
END;
```

Tras ejecutar esta orden, homogeneizamos la columna, obteniendo el siguiente resultado:

![alt text](image-2.png)


Algo similar sucede con los valores de la columna hire_date, con un código equivalente realizamos la operación correspondiente:

```sql
UPDATE human_resources
SET hire_date = CASE
    WHEN hire_date LIKE '%/%' THEN DATE_FORMAT(STR_TO_DATE(hire_date, '%m/%d/%Y'), '%Y-%m-%d')
    WHEN hire_date LIKE '%-%' THEN DATE_FORMAT(STR_TO_DATE(hire_date, '%m-%d-%Y'), '%Y-%m-%d')
    ELSE hire_date  
END;
```

Por último, transformamos los valores a tipo DATE:

```sql
ALTER TABLE human_resources 
	MODIFY birthdate DATE, 
    MODIFY hire_date DATE;
```

### 4.4 **Columnas gender, race, department, jobtitle y location, location_city y location_state**

Para estas columnas, queremos comprobar que no haya errores de sintaxis en las categorías, para ello utilizamos el siguiente código:

```sql
SELECT DISTINCT gender FROM human_resources;
SELECT DISTINCT race FROM human_resources;
SELECT DISTINCT department FROM human_resources;
SELECT DISTINCT jobtitle FROM human_resources;
SELECT DISTINCT location FROM human_resources;
SELECT DISTINCT location_city FROM human_resources;
SELECT DISTINCT location_state FROM human_resources;
```
convertimos ahora los datos a tipo VARCHAR:

```sql
ALTER TABLE human_resources
MODIFY gender VARCHAR(40),
MODIFY race VARCHAR(100),
MODIFY department VARCHAR(100),
MODIFY jobtitle VARCHAR(100),
MODIFY location VARCHAR(100),
MODIFY location_city VARCHAR(100),
MODIFY location_state VARCHAR(100);
```


### 4.5 **Columna Age**

Es de interés crear una columna que exprese la edad del trabajador, para ello utilizaremos la siguiente orden:

```sql
START TRANSACTION;
ALTER TABLE human_resources
ADD COLUMN age int NULL;
UPDATE human_resources
SET age = TIMESTAMPDIFF(YEAR, birthdate, CURDATE());
```

Al hacerlo, comprobamos que algunas edades son menores a cero, esto es así porque sus fechas de nacimiento son mayores a la fecha actual. Mantendremos la fecha, pero la edad la borraremos, pues no tienen sentido en nuestros futuros análisis.

```sql
UPDATE human_resources
SET age = CASE
WHEN age<0 THEN NULL
ELSE age
END;
```

### 4.6 **Columna termdate**

Eliminamos la parte UTC, convertimos los valores vacíos a null y cambiamos el tipo de dato a DATETIME

```sql
UPDATE human_resources  
SET termdate = CASE
    WHEN termdate = '' THEN NULL
    ELSE termdate
END;
UPDATE human_resources  
SET termdate = STR_TO_DATE(SUBSTRING(termdate, 1, 19), '%Y-%m-%d %H:%i:%s')
WHERE termdate LIKE '%UTC%';
SELECT * FROM human_resources;
ALTER TABLE human_resources  
MODIFY COLUMN termdate DATETIME;
```

## 5. Algunos análisis:

### 5.1 Distribución por género de los empleados en la empresa.  

Puesto que será un colectivo que consultemos con frecuencia, creamos una vista para los empleados en activo (mayores de 18 años y con fecha de fin de contrato aún no especificada).

```sql 
CREATE VIEW active_workers AS
SELECT * FROM human_resources
WHERE age>18 AND termdate = "";
```

Ahora comprobamos el género:

```sql 
SELECT gender, COUNT(gender) "n", ROUND(COUNT(gender) * 100 / SUM(COUNT(gender)) OVER (), 2) "%" 
FROM active_workers
GROUP BY gender
```

![alt text](image-5.png)

Tenemos un total de 50.97% de hombres, 46.28% de mujeres y el resto No Binario.

### 5.2 Distribución por raza/etnia de los empleados en la empresa.  

```sql 
SELECT 
    race, 
    COUNT(race) AS "n", 
    ROUND(COUNT(race) * 100.0 / SUM(COUNT(race)) OVER (), 2) AS "percentage"
FROM active_workers 
GROUP BY race 
ORDER BY 3 DESC;
```

![alt text](image-6.png)

Se ha ordenado de mayor a menor presencia. 

### 5.3 Distribución por edad de los empleados en la empresa.  

Podemos comprobar el máximo y mínimo de la edad:

```sql 
SELECT 
   MAX(age) as age_max,
   MIN(age) as age_min
FROM active_workers;
```

![alt text](image-7.png)

Creamos categorías para ordenar los tramos de edad

```sql 
SELECT CASE
    WHEN age < 25 THEN "menor de 25 años"
    WHEN age >= 25 AND age < 35 THEN "de 25 a 34 años"
    WHEN age >= 35 AND age < 45 THEN "de 35 a 44 años"
    WHEN age >= 45 AND age < 55 THEN "de 45 a 54 años"
    WHEN age >= 55 AND age < 64 THEN "de 55 a 64 años"
    ELSE "65 años o más"
END AS age_cat,
COUNT(*)
FROM active_workers 
GROUP BY age_cat 
ORDER BY COUNT(*) DESC;
```

![alt text](image-8.png)

### 5.4 Comparación del número de empleados en la sede y en ubicaciones remotas.  

```sql 
SELECT location, COUNT(*) FROM active_workers
GROUP BY location
```
![alt text](image-9.png)

### 5.5 Duración promedio del empleo para los empleados que han sido despedidos.  

```sql
SELECT AVG(DATEDIFF(termdate, hire_date))/365 "Average_job_lenght"
FROM human_resources
WHERE termdate < curdate() AND age>=18;
```

![alt text](image-10.png)

### 5.6 Variación de la distribución de género entre departamentos y puestos de trabajo.  

```sql
SELECT department, gender, count(gender) "n", count(gender)/SUM(count(gender)) OVER () * 100 "perc" 
FROM human_resources 
GROUP BY department, gender;
```

![alt text](image-11.png)

### 5.7 Distribución de los puestos de trabajo en toda la empresa.  

```sql
SELECT jobtitle, count(jobtitle) "n", count(jobtitle)/SUM(count(jobtitle)) OVER () * 100 "perc" 
FROM active_workers
GROUP BY jobtitle;
```

![alt text](image-12.png)

### 5.8 Departamento con la tasa de rotación más alta.  

```sql
SELECT department,
	total_count,
    terminated_count,
    terminated_count/total_count AS termination_rate
FROM(
	SELECT department,
    count(*) AS total_count,
    SUM(CASE WHEN termdate IS NOT NULL AND termdate <= curdate() THEN 1 else 0 end) AS terminated_count
    FROM human_resources
    WHERE age >= 18
    GROUP BY department)
    AS subquery
ORDER BY termination_rate DESC;
```
![alt text](image-13.png)

### 5.9 Distribución de empleados por ubicación según el estado.  

```sql
SELECT location_state, COUNT(*) "n"
FROM active_workers
GROUP BY location_state
ORDER BY COUNT(*) DESC;
```
![alt text](image-14.png)


### 5.10 Evolución del número de empleados en la empresa según las fechas de contratación y despido.  

```sql
SELECT
	year,
    hires,
    terminations,
    hires-terminations AS net_change,
    round((hires-terminations)/hires*100,2) AS net_cange_perc
FROM(
	SELECT YEAR (hire_date) AS year,
    COUNT(*) AS hires,
    SUM(CASE WHEN termdate IS NOT NULL AND termdate<= CURDATE() THEN 1 else 0 END) AS terminations
    FROM human_resources
    WHERE age >= 18
    GROUP BY YEAR(hire_date)
    ) AS subquery
ORDER BY year ASC;
```

![alt text](image-15.png)

### 5.11 Distribución de la antigüedad en cada departamento.

```sql
SELECT
    department,
    AVG(years_worked) AS average_years_working
FROM(
	SELECT 
    department,
    ROUND(DATEDIFF(curdate(),hire_date)/360) AS "years_worked"
    FROM active_workers
) AS subquery
GROUP BY department
ORDER BY average_years_working DESC;
```

![alt text](image-16.png)