# Aplicación de Machine Learning para Detección de Amenazas Web

## Tráfico Web y Contexto del Dataset

Cada vez que navegamos por Internet, generamos lo que se conoce como **tráfico web**: un conjunto de **peticiones (requests)** y **respuestas (responses)** entre nuestro navegador y los servidores que alojan los sitios que visitamos.  
Este tráfico ocurre constantemente, incluso cuando no somos plenamente conscientes de ello. Por ejemplo:

- Al ingresar a **una tienda online** para ver productos o añadir algo al carrito.  
- Al **iniciar sesión** en nuestras redes sociales o en un servicio de correo electrónico.  
- Al **rellenar un formulario** en una página institucional.  
- O incluso al **visualizar contenido multimedia** en plataformas de streaming.  

Cada una de estas acciones se traduce en **solicitudes HTTP** (como `GET`, `POST` o `PUT`), que pueden ser analizadas para detectar comportamientos anómalos.  
En condiciones normales, estas peticiones contienen parámetros legítimos; sin embargo, cuando un atacante introduce código malicioso —por ejemplo, inyecciones SQL o scripts XSS—, el tráfico se convierte en una señal potencial de amenaza.

Para estudiar y detectar estos patrones maliciosos, se emplea el siguiente conjunto de datos.

---

## Dataset CSIC-2010

### Características
- **Contenido:** 36,000 peticiones normales + 25,000 anómalas.  
- **Aplicación:** Comercio electrónico en español.  
- **Formato:** Peticiones HTTP (GET, POST, PUT) etiquetadas como “normal” o “anómala”.

### Tipos de Ataques Incluidos
- **Ataques estáticos:** Peticiones a recursos ocultos o inexistentes.  
- **Ataques dinámicos:** Modificación de argumentos (*SQL injection, XSS, buffer overflow*).  
- **Peticiones ilegales no intencionadas:** Comportamiento anómalo sin malicia.

---

## Ejemplos de Peticiones Anómalas

### 1. Inyección SQL

```tex
GET http://localhost:8080/tienda1/publico/anadir.jsp?id=2&nombre=Jam%F3n+Ib%E9rico&precio=85&cantidad=%27%3B+DROP+TABLE+usuarios%3B+SELECT+*+FROM+datos+WHERE+nombre+LIKE+%27%25&B1=A%F1adir+al+carrito
 HTTP/1.1
 ```

 **Característica maliciosa:** Contiene `DROP TABLE usuarios` en el parámetro `cantidad`.

### 2. Cross-Site Scripting (XSS)

```tex
GET http://localhost:8080/tienda1/publico/autenticar.jsp?modo=entrar&login=bob%40%3CSCRipt%3Ealert%28Paros%29%3C%2FscrIPT%3E.parosproxy.org&pwd=84m3ri156&remember=on&B1=Entrar
 HTTP/1.1
```

**Característica maliciosa:** Inyección de script en el parámetro `login`.

### 3. Path Traversal / Recursos No Existentes

```tex
GET http://localhost:8080/asf-logo-wide.gif
~ HTTP/1.1
```

**Característica maliciosa:** Solicitud de recurso con extensión inusual (`gif~`).

### 4. Inyección de Cabeceras HTTP

```tex
GET http://localhost:8080/tienda1/publico/registro.jsp?modo=registro&login=armand&password=prusiato&nombre=any%250D%250ASet-cookie%253A%2BTamper%253D5765205567234876235
```

**Característica maliciosa:** Inyección de cabecera `Set-Cookie` en el parámetro `nombre`.

---

## Transformación y Extracción de Datos

El conjunto de datos CSIC-2010 contiene peticiones HTTP con múltiples atributos (método, URL, protocolo, longitud del contenido, encabezados, entre otros).  
Antes de aplicar cualquier algoritmo de Machine Learning, es necesario transformar y extraer información relevante a partir de estos campos textuales y categóricos.

El proceso de **preprocesamiento y extracción de características** siguió las siguientes etapas:

1. **Codificación de Variables Categóricas**  
   Se utilizaron técnicas de **Label Encoding** para convertir variables textuales en valores numéricos interpretables por los modelos.  
   Esto se aplicó a las características:
   - `feature-Method` (GET, POST, PUT, etc.)
   - `feature-Protocol` (HTTP/1.0, HTTP/1.1, etc.)
   - `feature-Connection` (keep-alive, close, etc.)

2. **Análisis y Transformación de la URL**  
   Dado que las direcciones URL son un indicador clave de comportamiento malicioso, se extrajeron las siguientes variables:
   - `url_length`: longitud total de la URL.
   - `has_extension`: detección de extensiones web comunes (.html, .jsp, .php, .asp).
   - `is_localhost`: indicador de si la URL pertenece a un servidor local.

3. **Procesamiento del User-Agent**  
   A partir del encabezado `feature-User-Agent`, se extrajo una nueva característica `browser_type` que clasifica el tipo de navegador detectado (Chrome, Firefox, Safari, Edge u otros).  
   Esta información permite identificar patrones asociados a scripts automatizados o herramientas de ataque.

4. **Tratamiento de Longitud de Contenido**  
   El campo `feature-Content-Length` —que en algunos casos no estaba presente— fue normalizado reemplazando el valor `'notpresent'` por `0` y convirtiendo la columna a formato numérico (`int`).

5. **Análisis del Parámetro Query**  
   La cadena de consulta (`feature-Query`) se descompuso en dos variables:
   - `has_query`: indica si la petición contiene parámetros.
   - `query_length`: mide la longitud de dichos parámetros.  
   Estas métricas permiten detectar peticiones manipuladas, típicas de ataques de **inyección SQL** o **XSS**.

6. **Simplificación del Encabezado Accept**  
   El campo `feature-Accept` se clasificó en categorías simples de tipo MIME:
   - `application/json`, `text/html`, `image/*`, `application/xml`, entre otros.  
   Esto permitió capturar la naturaleza de las peticiones sin introducir ruido textual.

7. **Codificación de la Variable Objetivo**  
   Finalmente, la columna `Label` se codificó en formato numérico (`label_encoded`) para distinguir entre peticiones **normales** y **anómalas**.

Tras esta etapa, se prepararon las variables predictoras `X` y la variable objetivo `y` mediante la función `prepare_features()`, que seleccionó las 11 características más informativas para el entrenamiento de los modelos.

---

## Modelos de Machine Learning Empleados

Para la detección de tráfico web anómalo, se implementaron y compararon diversos modelos de aprendizaje supervisado:

1. **Naïve Bayes (GaussianNB, MultinomialNB y BernoulliNB)**  
   - Utilizado por su eficiencia y bajo costo computacional en la clasificación de texto y datos categóricos.  
   - El modelo **GaussianNB** mostró un excelente desempeño al trabajar con variables numéricas continuas como `url_length` y `query_length`.  
   - **MultinomialNB** se adaptó mejor a variables de conteo, mientras que **BernoulliNB** se empleó para variables binarias como `has_query` o `has_extension`.


