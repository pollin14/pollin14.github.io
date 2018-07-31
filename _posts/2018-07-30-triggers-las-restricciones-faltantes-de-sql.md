---
layout: post
title:  "Triggers: Las restricciones faltantes de SQL"
date:   2018-07-30 15:52:14 -0500
categories: sql

---

<h2 id="introducción">Introducción</h2>
<p>Un <strong>trigger</strong>  (disparador en español) es código SQL que se ejecuta <em>antes</em> o <em>después</em> de una determinada acción, por ejemplo, antes de insertar o después de actualizar una fila. Su uso y definición son bastante generales pero yo he encontrado un uso poco explotado y muy útil para sistemas con muchas entidades u objetos: restricciones personalizadas.</p>
<h2 id="las-restricciones-de-sql">Las restricciones de SQL</h2>
<p>SQL viene con unas cuantas restricciones para mantener la integridad de los datos. Entre las que se encuentran la <strong>primary key</strong> (clave primaria), que al igual que <strong>unique</strong>, nos restringen a que la columna o columnas defiendas como clave primaria no se repitan, es decir, sean únicas. También están las <strong>foreign keys</strong> (claves foráneas) las cuales nos restringen a que un valor de una tabla exista como clave primaria en otra. Dicho en otras palabras, no es posible agregar un registro en una tabla sin que exista otro registro en otra tabla. Incluso los <strong>data types</strong> (tipos de datos) son una restricción ya que nos impiden ingresar o actualizar datos del tipo incorrecto.</p>
<p>Estas son todas las principales restricciones que nos dada Sql. Todas ellas dependen de como se modele la información de negocio. Sin embargo, no son suficientes para todos los casos donde queremos agregar restricciones a la información ingresada lo cual, al igual que si no existieran los tipos de datos, tendríamos serias inconsistencias en los datos recuperados. Por ejemplo, un programador podría guardar en una columna tipo <code>VARCHAR</code> el valor monetario <code>1,001.59</code> y otro recuperándolo para hacer cálculos con el. Esto complicará la implementación del sistema, dificultará su mantenimiento y sus cambios.</p>
<p>Definamos una base de datos con muchas restricciones. Queremos guardar perfiles de jefes (<code>boss_profiles</code>)  y otro para perfiles de empleados (<code>employee_profiles</code>), además, queremos que cada jefe y empleado tenga asociado un usuario (<code>users</code>). También queremos que un usuario tenga asociadas cero o más cuentas bancarias (<code>user_bank_accounts</code>). Por último, queremos que un jefe tenga cero o más pagos (<code>payments</code>) y que no se puedan agregar pagos si el jefe no tiene ninguna cuenta bancaria.</p>
<p>Una definición de la base de datos bastante común sería la siguiente.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> users <span class="token punctuation">(</span>  
  id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">PRIMARY</span> <span class="token keyword">KEY</span> <span class="token keyword">AUTO_INCREMENT</span><span class="token punctuation">,</span>  
  email <span class="token keyword">VARCHAR</span><span class="token punctuation">(</span><span class="token number">100</span><span class="token punctuation">)</span> <span class="token keyword">UNIQUE</span> <span class="token operator">NOT</span> <span class="token boolean">NULL</span>  
<span class="token punctuation">)</span><span class="token punctuation">;</span>  

<span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> employee_profiles <span class="token punctuation">(</span>  
  id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">PRIMARY</span> <span class="token keyword">KEY</span> <span class="token keyword">AUTO_INCREMENT</span><span class="token punctuation">,</span>  
  user_id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">UNIQUE</span><span class="token punctuation">,</span>  
  name <span class="token keyword">VARCHAR</span><span class="token punctuation">(</span><span class="token number">100</span><span class="token punctuation">)</span> <span class="token operator">NOT</span> <span class="token boolean">NULL</span><span class="token punctuation">,</span>  
  <span class="token keyword">FOREIGN</span> <span class="token keyword">KEY</span> <span class="token punctuation">(</span>user_id<span class="token punctuation">)</span> <span class="token keyword">REFERENCES</span> users <span class="token punctuation">(</span>id<span class="token punctuation">)</span>  
<span class="token punctuation">)</span><span class="token punctuation">;</span>
  
<span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> boss_profiles <span class="token punctuation">(</span>  
  id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">PRIMARY</span> <span class="token keyword">KEY</span> <span class="token keyword">AUTO_INCREMENT</span><span class="token punctuation">,</span>  
  user_id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">UNIQUE</span><span class="token punctuation">,</span>  
  name <span class="token keyword">VARCHAR</span><span class="token punctuation">(</span><span class="token number">100</span><span class="token punctuation">)</span> <span class="token operator">NOT</span> <span class="token boolean">NULL</span><span class="token punctuation">,</span>  
  <span class="token keyword">FOREIGN</span> <span class="token keyword">KEY</span> <span class="token punctuation">(</span>user_id<span class="token punctuation">)</span> <span class="token keyword">REFERENCES</span> users <span class="token punctuation">(</span>id<span class="token punctuation">)</span>  
<span class="token punctuation">)</span><span class="token punctuation">;</span>  
  
<span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> user_bank_accounts <span class="token punctuation">(</span>  
  id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">PRIMARY</span> <span class="token keyword">KEY</span> <span class="token keyword">AUTO_INCREMENT</span><span class="token punctuation">,</span>  
  number <span class="token keyword">VARCHAR</span><span class="token punctuation">(</span><span class="token number">18</span><span class="token punctuation">)</span> <span class="token keyword">UNIQUE</span> <span class="token operator">NOT</span> <span class="token boolean">NULL</span><span class="token punctuation">,</span>  
  user_id <span class="token keyword">INT</span> UNSIGNED       <span class="token operator">NOT</span> <span class="token boolean">NULL</span><span class="token punctuation">,</span>  
  <span class="token keyword">FOREIGN</span> <span class="token keyword">KEY</span> <span class="token punctuation">(</span>user_id<span class="token punctuation">)</span> <span class="token keyword">REFERENCES</span> users <span class="token punctuation">(</span>id<span class="token punctuation">)</span>  
<span class="token punctuation">)</span><span class="token punctuation">;</span>  
  
<span class="token keyword">CREATE</span> <span class="token keyword">TABLE</span> payments <span class="token punctuation">(</span>  
  id <span class="token keyword">INT</span> UNSIGNED <span class="token keyword">PRIMARY</span> <span class="token keyword">KEY</span> <span class="token keyword">AUTO_INCREMENT</span><span class="token punctuation">,</span>  
  boss_profile_id <span class="token keyword">INT</span> UNSIGNED<span class="token punctuation">,</span>
  amount <span class="token keyword">FLOAT</span><span class="token punctuation">,</span>
  <span class="token keyword">FOREIGN</span> <span class="token keyword">KEY</span> <span class="token punctuation">(</span>boss_profile_id<span class="token punctuation">)</span> <span class="token keyword">REFERENCES</span> boss_profiles <span class="token punctuation">(</span>id<span class="token punctuation">)</span>  
<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<p>La definición de estas tablas casi cumple todos los requerimientos excepto uno: No pueden existir pagos sin que exista una cuenta de banco asociada a un usuario.</p>
<p>Para cubrir estas dos restricciones y garantizar que nuestros datos son consistentes con nuestras reglas de negocio necesitamos hacer uso de los triggers.</p>
<h2 id="extendiendo-las-restricciones-de-sql">Extendiendo las restricciones de Sql</h2>
<p>Un flujo correcto para ingresar un pago es el siguiente:</p>
<ol>
<li>Inserción de un usuario</li>
<li>Inserción de un empleador</li>
<li>Inserción de una cuenta bancaria</li>
<li>Inserción de un pago</li>
</ol>
<p>Notemos que los pasos 1 y 2 no se pueden intercambiar ya que existe la restricción de clave foránea en la tabla <code>boss_profiles</code> la cual nos obliga a que exista un usuario antes de insertar un jefe. En contraste, los paso 3 y 4 si son intercambiables ya que no existe ninguna restricción para insertar primero un pago y luego la cuenta bancaria o viceversa. Pero en nuestro modelo de negocio no tiene sentido que exista un pago cuando un jefe no tienen jefe no tiene ninguna cuenta bancaria asociada.</p>
<p>Este tipo de errores en los datos son causados, por ejemplo, cuando un servicio externo falla y el programa no maneje bien el error; cuando se inserten datos manualmente;  cuando exista un error de programación en uno de los varios sistemas que tienen accesos a estas tablas; cuando se inicializa un nuevo ambiente de pruebas (testing) o desarrollo.</p>
<p>Inicializar un ambiente de desarrollo es el caso más común donde se insertan inconsistencias en la base de datos. Esto ocurre, ya sea una por una prueba automatizada o importar una base de datos antigua para iniciar el desarrollo.</p>
<p>Podemos resolver todos estos problemas agregando una restricción personalizada con un <em>trigger</em>.</p>
<pre class=" language-sql"><code class="prism  language-sql"><span class="token keyword">DELIMITER</span> <span class="token operator">|</span>  
  
<span class="token comment">#  </span>
<span class="token comment"># Trigger creation  </span>
<span class="token comment">#  </span>
<span class="token comment"># It avoid insert a new row in the `payments` table if the user has not a row in the `user_bank_accounts`  </span>
<span class="token comment">#  </span>
  
<span class="token keyword">CREATE</span> <span class="token keyword">TRIGGER</span> trigger_insert_before_payments_without_user_bank_account  
BEFORE <span class="token keyword">INSERT</span> <span class="token keyword">ON</span> payments  
<span class="token keyword">FOR EACH ROW</span>  
 <span class="token keyword">BEGIN</span> 
	<span class="token keyword">SELECT</span> <span class="token function">count</span><span class="token punctuation">(</span><span class="token operator">*</span><span class="token punctuation">)</span>  
    <span class="token keyword">INTO</span> <span class="token variable">@user_accounts</span>  
    <span class="token keyword">FROM</span> <span class="token punctuation">`</span>boss_profiles<span class="token punctuation">`</span>  
      <span class="token keyword">LEFT</span> <span class="token keyword">JOIN</span> <span class="token punctuation">`</span>users<span class="token punctuation">`</span> <span class="token keyword">ON</span> <span class="token punctuation">`</span>users<span class="token punctuation">`</span><span class="token punctuation">.</span><span class="token punctuation">`</span>id<span class="token punctuation">`</span> <span class="token operator">=</span> <span class="token punctuation">`</span>boss_profiles<span class="token punctuation">`</span><span class="token punctuation">.</span><span class="token punctuation">`</span>user_id<span class="token punctuation">`</span>  
	  <span class="token keyword">RIGHT</span> <span class="token keyword">JOIN</span> <span class="token punctuation">`</span>user_bank_accounts<span class="token punctuation">`</span> <span class="token keyword">ON</span> <span class="token punctuation">`</span>user_bank_accounts<span class="token punctuation">`</span><span class="token punctuation">.</span><span class="token punctuation">`</span>user_id<span class="token punctuation">`</span> <span class="token operator">=</span> <span class="token punctuation">`</span>users<span class="token punctuation">`</span><span class="token punctuation">.</span><span class="token punctuation">`</span>id<span class="token punctuation">`</span>  
	  <span class="token keyword">WHERE</span> <span class="token punctuation">`</span>boss_profiles<span class="token punctuation">`</span><span class="token punctuation">.</span><span class="token punctuation">`</span>id<span class="token punctuation">`</span> <span class="token operator">=</span> NEW<span class="token punctuation">.</span>boss_profile_id<span class="token punctuation">;</span>  
  
  <span class="token keyword">IF</span> <span class="token variable">@user_accounts</span> <span class="token operator">=</span> <span class="token number">0</span>  
  <span class="token keyword">THEN</span>  
	SIGNAL SQLSTATE <span class="token string">'45000'</span>  
	<span class="token keyword">SET</span> MESSAGE_TEXT <span class="token operator">=</span> <span class="token string">'The `users` row has not an `user_bank_accounts` row. Add first an `user_bank_accounts` row before insert.'</span><span class="token punctuation">;</span>  
  <span class="token keyword">END</span> <span class="token keyword">IF</span><span class="token punctuation">;</span>  
<span class="token keyword">END</span><span class="token punctuation">;</span>  
  
<span class="token keyword">DELIMITER</span> <span class="token punctuation">;</span>
</code></pre>
<p>Primeramente, definimos el nombre del trigger <code>trigger_insert_before_payments_without_user_bank_account</code> usando la siguiente nomenclatura <code>trigger_[insert|update|delete]_[after|before]_[table]_A_DESCRIPTION</code>. Lo que logramos con esta nomenclatura es auto-documentar el trigger y evitar colisión de nombres.</p>
<p>Luego, <a href="https://dev.mysql.com/doc/refman/5.5/en/create-trigger.html">en el cuerpo de trigger</a>, definimos una sentencia <code>SELECT</code> para contar el numero de registros en la tabla <code>user_bank_accounts</code> que tiene el nuevo jefe (<code>NEW.boss_profile_id</code>) insertado en la tabla <code>payments</code>, para después almacenarlo en la variable <code>@user_accounts</code>. Por último, si el valor de la variable es 0, es decir, no el jefe que se quiere insertar no tiene ninguna cuenta bancaria asociada, lanzamos una excepción con el código 45000 ( <a href="https://dev.mysql.com/doc/refman/5.5/en/signal.html">excepción sin manejar definida por el usuario</a>) indicando por que no se pudo hacer la inserción en <code>boss_profiles</code>.</p>
<h2 id="consecuencias">Consecuencias</h2>
<p>Siempre que agregamos cualquier restricción a una tabla el rendimiento de de esta disminuye al realizar la inserción ya que hay que verificar que todas se cumplan. Así que, agregar una restricción <code>unique</code> a una tabla hará que las inserciones sean más lentas y, por supuesto, agregar una restricción con un trigger que usa <code>JOINS</code> serán todavía más lentas. Por tal motivo sí nos encontramos ante un caso de uso que necesite realizar muchas inserciones a una buena velocidad, no deberíamos usar restricciones. Un ejemplo de este tipo de tablas, son tablas FLAT (Seudo-vistas Materializadas) de Magento.</p>
<p>Sin embargo, en Programación Orientada a Objetos, más en los <em>tipados</em> como PHP7 y Java, prefiero agregar tantas restricciones a los datos como sea posible, para maximizar la estabilidad del sistema y auto-documentarlo. Ya que lo que buscamos aquí es una alta mantenibilidad y en segunda instancia, performance y este ultimo pude ser mejorado aumentando el poder del servidor de base de datos.</p>
<p>Cuánto se disminuye el rendimiento en inserciones debido a los trigger es todo un tema por si mismo, así que no entrare en detalles aquí.</p>
<h2 id="conclusión">Conclusión</h2>
<p>Es factible y recomendable usar <em>triggers</em>, siempre que sea posible, para agregando restricciones personalizadas a los datos con solo con un poco de código (en nuestro ejemplo fue solamente un <code>SELECT</code> y una condición, todo lo demás es la estructura mínima de un trigger), evitándonos encontrarnos con errores inesperados en la aplicación por culpa de datos inconsistentes.</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

