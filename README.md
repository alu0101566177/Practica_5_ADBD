# Práctica 5 ADBD, Vistas y disparadores

## Consultas para vistas

### Obtenga las ventas totales por categoría de películas ordenadas descendentemente.

```sql
CREATE VIEW view_sales_per_category AS SELECT c.name, COUNT(*) AS total_sales
from rental r
  inner join inventory i on r.inventory_id =  i.inventory_id
  inner join film_category f on i.film_id = f.film_id
  inner join category c on f.category_id = c.category_id
GROUP BY c.name
ORDER BY total_sales DESC;
```

### Obtenga las ventas totales por tienda, donde se refleje la ciudad, el país (concatenar la ciudad y el país empleando como separador la “,”), y el encargado.

```sql
CREATE VIEW view_sales_per_store AS WITH sales_per_store AS (
	SELECT s.store_id, COUNT(*) AS total_sales
	FROM rental r
  		INNER JOIN staff st ON r.staff_id = st.staff_id
  		INNER JOIN store s ON st.store_id = s.store_id
	GROUP BY s.store_id
)

SELECT s.total_sales,
	CONCAT(c.city, ':', co.country) AS location,
	CONCAT(m.first_name, ' ', m.last_name) AS manager
FROM sales_per_store s
	INNER JOIN store st ON s.store_id = st.store_id
	INNER JOIN staff m ON st.manager_staff_id = m.staff_id
	INNER JOIN address a ON st.address_id = a.address_id
	INNER JOIN city c ON a.city_id = c.city_id
	INNER JOIN country co ON c.country_id = co.country_id
;
```

### Obtenga una lista de películas, donde se reflejen el identificador, el título, descripción, categoría, el precio, la duración de la película, clasificación, nombre y apellidos de los actores (puede realizar una concatenación de ambos).

```sql
CREATE VIEW view_film_info AS SELECT f.film_id AS id,
	   f.title,
	   f.description,
	   STRING_AGG(DISTINCT c.name, ', ') AS categories,
	   f.replacement_cost AS price,
	   f.length,
	   f.rating,
	   STRING_AGG(DISTINCT (a.first_name || ' ' || a.last_name), ', ') AS actor
FROM film f
	LEFT JOIN film_category fc ON f.film_id = fc.film_id
	LEFT JOIN category c ON fc.category_id = c.category_id
	LEFT JOIN film_actor fa ON fa.film_id = f.film_id
	LEFT JOIN actor a ON fa.actor_id = a.actor_id
GROUP BY
    f.film_id, f.title, f.description,
    price, f.length, f.rating;
```

### Obtenga la información de los actores, donde se incluya sus nombres y apellidos, las categorías y sus películas. Los actores deben de estar agrupados y, las categorías y las películas deben estar concatenados por “:”

```sql
CREATE VIEW view_actor_info AS SELECT
	CONCAT(a.first_name, ' ', a.last_name) AS name,
	STRING_AGG(DISTINCT c.name, ':') AS categories,
	STRING_AGG(DISTINCT f.title, ':') AS films
FROM actor a
	LEFT JOIN film_actor fa ON a.actor_id = fa.actor_id
	LEFT JOIN film f ON fa.film_id = f.film_id
	LEFT JOIN film_category fc ON f.film_id = fc.film_id
	LEFT JOIN category c ON fc.category_id = c.category_id
GROUP BY a.first_name, a.last_name;
```

## Análisis de restricciones de la base de datos

Se observa que no existen restricciones adicionales en la base de datos (excepto las claves primarias y foráneas).

Un buen CHECK a añadir sería en los precios para evitar montos negativos, salvo en la tabla `payment` donde se podrían contemplar devoluciones:

```sql
ALTER TABLE film
ADD CONSTRAINT chk_film_replacement_cost
CHECK (replacement_cost > 0);
```

También es conveniente añadir restricciones a los datos de contacto con expresiones regulares:

```sql
-- Tabla address
ALTER TABLE address
ADD CONSTRAINT chk_address_postal_code
CHECK (postal_code ~ '^[0-9]{5}$'); -- (No incluida)

ALTER TABLE address
ADD CONSTRAINT chk_address_phone
CHECK (phone ~ '^\+?[0-9]{7,15}$'); -- (No incluida)

-- Tablas `staff` y `customer`
ALTER TABLE staff
ADD CONSTRAINT chk_staff_email
CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

ALTER TABLE customer
ADD CONSTRAINT chk_customer_email
CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
```

## Explique la sentencia que aparece en la tabla `customer`

```sql
last_updated BEFORE UPDATE ON customer
FOR EACH ROW EXECUTE PROCEDURE last_updated()
```

La sentencia define un trigger que se ejecuta antes de cada actualización (BEFORE UPDATE) en la tabla `customer`.
Su función es llamar al procedimiento almacenado `last_updated()`, el cual normalmente actualiza automáticamente
la columna `last_update` con la fecha y hora actuales, garantizando que siempre se registre cuándo se modificó por última vez un registro.

Una solución similar puede encontrarse en tablas como `rental`, donde también se requiere mantener un control automático
del momento en que se actualizan los datos.

## Creamos un trigger para hacer seguimiento de las inserciones de nuevas películas

1. Creamos una tabla para insertar las fechas

```sql
CREATE TABLE film_audit (
	audit_id SERIAL PRIMARY KEY,
	film_id INTEGER NOT NULL REFERENCES film(film_id),
	insertion_date TIMESTAMP NOT NULL
);
```

2. Definimos una función para calcular la fecha actual e insertar un nuevo registro en `film_audit`

```sql
CREATE OR REPLACE FUNCTION register_film_insertion()
RETURNS TRIGGER AS $$
BEGIN
	INSERT INTO film_audit (film_id, insertion_date)
	VALUES (NEW.film_id, NOW());
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

3. Asociamos la función al evento producido después de un INSERT en la tabla `film`

```sql
CREATE TRIGGER trg_film_insertion
AFTER INSERT ON film
FOR EACH ROW
EXECUTE FUNCTION register_film_insertion();
```

## Creamos un trigger para hacer seguimiento de las películas eliminadas

1. Creamos la tabla para registrar las eliminaciones

```sql
CREATE TABLE film_delete_audit (
	audit_id SERIAL PRIMARY KEY,
	old_film_id INTEGER NOT NULL,
	film_title TEXT NOT NULL,
	delete_date TIMESTAMP NOT NULL
);
```

2. Definimos una función para registrar la eliminación e incluir la fecha de la misma

```sql
CREATE OR REPLACE FUNCTION register_film_delete()
RETURNS TRIGGER AS $$
BEGIN
	INSERT INTO film_delete_audit (old_film_id, film_title, delete_date)
	VALUES (OLD.film_id, OLD.title, NOW());
	RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

3. Asociamos la función al evento producido antes de un DELETE en la tabla `film`

```sql
CREATE TRIGGER trg_film_delete
BEFORE DELETE ON film
FOR EACH ROW
EXECUTE FUNCTION register_film_delete();
```

## ¿Para qué son las secuencias en SQL?

Las secuencias en bases de datos (como PostgreSQL) son objetos especiales que generan valores numéricos únicos y consecutivos, comúnmente utilizados para claves primarias o identificadores automáticos.

Son relevantes porque automatizan la generación de identificadores y son esenciales para mantener el orden, la unicidad y la coherencia en las tablas de una base de datos relacional.
