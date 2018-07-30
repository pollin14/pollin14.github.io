---
layout: post
title:  "Triggers: Las restricciones faltantes de SQL"
date:   2018-07-30 15:52:14 -0500
categories: sql
---

## Introducción

Un **trigger**  (disparador en español) es código SQL que se ejecuta *antes* o *después* de una determinada acción, por ejemplo, antes de insertar o después de actualizar una fila. Su uso y definición son bastante generales pero yo he encontrado un uso poco explotado y muy útil para sistemas con muchas entidades u objetos: restricciones personalizadas.

## Las restricciones de SQL

SQL viene con unas cuantas restricciones para mantener la integridad de los datos. Entre las que se encuentran la **primary key** (clave primaria), que al igual que **unique**, nos restringen a que la columna o columnas defiendas como clave primaria no se repitan, es decir, sean únicas. También están las **foreign keys** (claves foráneas) las cuales nos restringen a que un valor de una tabla exista como clave primaria en otra. Dicho en otras palabras, no es posible agregar un registro en una tabla sin que exista otro registro en otra tabla. Incluso los **data types** (tipos de datos) son una restricción ya que nos impiden ingresar o actualizar datos del tipo incorrecto.

Estas son todas las principales restricciones que nos dada Sql. Todas ellas dependen de como se modele la información de negocio. Sin embargo, no son suficientes para todos los casos donde queremos agregar restricciones a la información ingresada lo cual, al igual que si no existieran los tipos de datos, tendríamos serias inconsistencias en los datos recuperados. Por ejemplo, un programador podría guardar en una columna tipo `VARCHAR` el valor monetario `1,001.59` y otro recuperándolo para hacer cálculos con el. Esto complicará la implementación del sistema, dificultará su mantenimiento y sus cambios. 

Definamos una base de datos con muchas restricciones. Queremos guardar perfiles de jefes (`boss_profiles`)  y otro para perfiles de empleados (`employee_profiles`), además, queremos que cada jefe y empleado tenga asociado un usuario (`users`). También queremos que un usuario tenga asociadas cero o más cuentas bancarias (`user_bank_accounts`). Por último, queremos que un jefe tenga cero o más pagos (`payments`) y que no se puedan agregar pagos si el jefe no tiene ninguna cuenta bancaria.

Una definición de la base de datos bastante común sería la siguiente.

```sql
CREATE TABLE users (  
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,  
  email VARCHAR(100) UNIQUE NOT NULL  
);  

CREATE TABLE employee_profiles (  
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,  
  user_id INT UNSIGNED UNIQUE,  
  name VARCHAR(100) NOT NULL,  
  FOREIGN KEY (user_id) REFERENCES users (id)  
);
  
CREATE TABLE boss_profiles (  
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,  
  user_id INT UNSIGNED UNIQUE,  
  name VARCHAR(100) NOT NULL,  
  FOREIGN KEY (user_id) REFERENCES users (id)  
);  
  
CREATE TABLE user_bank_accounts (  
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,  
  number VARCHAR(18) UNIQUE NOT NULL,  
  user_id INT UNSIGNED       NOT NULL,  
  FOREIGN KEY (user_id) REFERENCES users (id)  
);  
  
CREATE TABLE payments (  
  id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,  
  boss_profile_id INT UNSIGNED,
  amount FLOAT,
  FOREIGN KEY (boss_profile_id) REFERENCES boss_profiles (id)  
);
```

La definición de estas tablas casi cumple todos los requerimientos excepto uno: No pueden existir pagos sin que exista una cuenta de banco asociada a un usuario.

Para cubrir estas dos restricciones y garantizar que nuestros datos son consistentes con nuestras reglas de negocio necesitamos hacer uso de los triggers.

## Extendiendo las restricciones de Sql

Un flujo correcto para ingresar un pago es el siguiente:

1. Inserción de un usuario
2. Inserción de un empleador
3. Inserción de una cuenta bancaria
4. Inserción de un pago

Notemos que los pasos 1 y 2 no se pueden intercambiar ya que existe la restricción de clave foránea en la tabla `boss_profiles` la cual nos obliga a que exista un usuario antes de insertar un jefe. En contraste, los paso 3 y 4 si son intercambiables ya que no existe ninguna restricción para insertar primero un pago y luego la cuenta bancaria o viceversa. Pero en nuestro modelo de negocio no tiene sentido que exista un pago cuando un jefe no tienen jefe no tiene ninguna cuenta bancaria asociada.

Este tipo de errores en los datos son causados, por ejemplo, cuando un servicio externo falla y el programa no maneje bien el error; cuando se inserten datos manualmente;  cuando exista un error de programación en uno de los varios sistemas que tienen accesos a estas tablas; cuando se inicializa un nuevo ambiente de pruebas (testing) o desarrollo.

Inicializar un ambiente de desarrollo es el caso más común donde se insertan inconsistencias en la base de datos. Esto ocurre, ya sea una por una prueba automatizada o importar una base de datos antigua para iniciar el desarrollo.

Podemos resolver todos estos problemas agregando una restricción personalizada con un *trigger*.

```sql
DELIMITER |  
  
#  
# Trigger creation  
#  
# It avoid insert a new row in the `payments` table if the user has not a row in the `user_bank_accounts`  
#  
  
CREATE TRIGGER trigger_insert_before_payments_without_user_bank_account  
BEFORE INSERT ON payments  
FOR EACH ROW  
 BEGIN 
	SELECT count(*)  
    INTO @user_accounts  
    FROM `boss_profiles`  
      LEFT JOIN `users` ON `users`.`id` = `boss_profiles`.`user_id`  
	  RIGHT JOIN `user_bank_accounts` ON `user_bank_accounts`.`user_id` = `users`.`id`  
	  WHERE `boss_profiles`.`id` = NEW.boss_profile_id;  
  
  IF @user_accounts = 0  
  THEN  
	SIGNAL SQLSTATE '45000'  
	SET MESSAGE_TEXT = 'The `users` row has not an `user_bank_accounts` row. Add first an `user_bank_accounts` row before insert.';  
  END IF;  
END;  
  
DELIMITER ;
```

Primeramente, definimos el nombre del trigger `trigger_insert_before_payments_without_user_bank_account` usando la siguiente nomenclatura `trigger_[insert|update|delete]_[after|before]_[table]_A_DESCRIPTION`. Lo que logramos con esta nomenclatura es auto-documentar el trigger y evitar colisión de nombres.

Luego, [en el cuerpo de trigger](https://dev.mysql.com/doc/refman/5.5/en/create-trigger.html), definimos una sentencia `SELECT` para contar el numero de registros en la tabla `user_bank_accounts` que tiene el nuevo jefe (`NEW.boss_profile_id`) insertado en la tabla `payments`, para después almacenarlo en la variable `@user_accounts`. Por último, si el valor de la variable es 0, es decir, no el jefe que se quiere insertar no tiene ninguna cuenta bancaria asociada, lanzamos una excepción con el código 45000 ( [excepción sin manejar definida por el usuario](https://dev.mysql.com/doc/refman/5.5/en/signal.html)) indicando por que no se pudo hacer la inserción en `boss_profiles`.

## Rendimiento

Esta es un tema largo y complejo pero explicare de manera breve las consecuencias de agregar restricciones. 

Siempre que agregamos cualquier restricción a una tabla el rendimiento de de esta disminuye al realizar la inserción ya que hay que verificar que todas se cumplan. Así que, agregar una restricción `unique` a una tabla hará que las inserciones sean más lentas y, por supuesto, agregar una restricción con un trigger que usa `JOINS` serán todavía más lentas. Por tal motivo sí nos encontramos ante un caso de uso que necesite realizar muchas inserciones a una buena velocidad, no deberíamos usar restricciones. Un ejemplo de este tipo de tablas, son tablas FLAT (Seudo-vistas Materializadas) de Magento. 

Sin embargo, en Programación Orientada a Objetos, más en los *tipados* como PHP7 y Java, prefiero agregar tantas restricciones a los datos como sea posible, para maximizar la estabilidad del sistema y auto-documentarlo. Ya que lo que buscamos aquí es una alta mantenibilidad y en segunda instancia, performance y este ultimo pude ser mejorado aumentando el poder del servidor de base de datos.

## Conclusión

Es factible y recomendable usar *triggers*, siempre que sea posible, para agregando restricciones personalizadas a los datos con solo con un poco de código (en nuestro ejemplo fue solamente un `SELECT` y una condición, todo lo demás es la estructura mínima de un trigger), evitándonos encontrarnos con errores inesperados en la aplicación por culpa de datos inconsistentes.

> Written with [StackEdit](https://stackedit.io/).
