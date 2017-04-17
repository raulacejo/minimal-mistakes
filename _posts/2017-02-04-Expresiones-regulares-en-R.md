Siempre me han parecido muy complejas, y al mismo tiempo, nunca he dudado de su gran utilidad. Saber manejar las expresiones regulares, ya sea en R o en cualquier otro lenguaje, nos puede ahorrar muchos quebraderos de cabeza, y es que su uso permite la realización de tareas que en un principio podrían parecer muy complejas.

Me inicio con vosotros en el aprendizaje de expresiones regulares (también conocidas como regex) a través del [tutorial][1] de Gloria Li and Jenny Bryan.

#####Preparación de los datos y paquetes

En esta guía se usa el paquete [stringr][2] de Hadley Wickham, que facilita mucho el uso de regex en R. Así que si no lo has hecho ya, deberías instalarlo:

```{r echo = FALSE} 
library(stringr)
```

Los archivos que se usa en el tutorial se descargan de [aquí][3], y mediante la función list.files() obtenemos una lista de todos los archivos descargados.

```
files <- list.files(path = "/Users/gart/Google Drive/R/STAT545-UBC.github.io-master")
head(files)
```

Y cargamos en R el archivo "gapminderDataFiveYear.txt":

```
gDat <- read.delim("/Users/gart/Google Drive/R/STAT545-UBC.github.io-master/gapminderDataFiveYear.txt")
str(gDat)
```

Ahora podemos usar alguna función para extraer determinados nombres de archivos, por ejemplo, podemos usar grep() para encontrar todos aquellos archivos que contienen la cadena "dplyr". Con el parámetro value = TRUE obtenemos los valores, mientras que si optamos por el parámetro value = FALSE obtenemos como resultados los índices de esos archivos:

```
grep("dplyr", files, value = TRUE)
```

```
grep("dplyr", files, value = FALSE)
```

Pero, ¿como haríamos para extraer todos los archivos que contengan "hw" y luego algo más y luego "dplyr"?. Aquí es donde entran en juego las expersiones regulares.

Como decíamos, una expresión regular es un patrón que describe una relación de cadenas con una estructura común. En R, muchas funciones de {base} así como del paquete stringr utilizan expresiones regulares, incluse el "buscar y reemplazar" en RStudio permite el uso de regex.

#####Sintáxis de las expresiones regulares

Las expresiones regulares normalmente especifican caracteres (o clases de caracteres) a buscar, además de información sobre las repeticiones y la ubicación dentro de la cadena. Esto se logra con la ayuda de meta-caracteres que tienen un significado específico: $ * +. ? [] ^ {} | () \. Vamos a utilizar algunos ejemplos para introducir pequeñas sintaxis de expresiones regulares y entender lo que significan estos metacaracteres.

**Secuencias de escape**

Hay algunos caracteres especiales en R que no pueden ser codificados directamente en una cadena. Por ejemplo, supongamos que especificamos un patrón con comillas simples y que desea encontrar países con la comilla simple '. Tendríamos que "escapar" la comilla simple en el patrón, precediéndola con el metacarácter \, porque es claro que no forma parte de la especificación del patrón. Veamos un código de ejemplo para esto:

```
grep('\'', levels(gDat$country), value = TRUE)
```

Hay [otros caracteres][4] en R que suelen requerir escape, algunos ejemplos:

* \': comillas simples
* \": comillas dobles
* \n: linea nueva
* \r: retorno de carro
* \t: tabulado

**Cuantificadores**

Estos meta-caracteres sirven para especificar el número de repeticiones en el patrón:

* *: aparece al menos 0 veces
* +: aparece al menos una vez
* ?: aparece como máximo una vez
* {n}: aparece exactamente n veces
* {n,}: aparece al menos n veces
* {n,m}: aparece entre n y m veces

```
(strings <- c("a", "ab", "acb", "accb", "acccb", "accccb"))
grep("ac*b", strings, value = TRUE)
grep("ac+b", strings, value = TRUE)
grep("ac?b", strings, value = TRUE)
grep("ac{2}b", strings, value = TRUE)
grep("ac{2,}b", strings, value = TRUE)
grep("ac{2,3}b", strings, value = TRUE)
```

**Posición del patrón dentro de la cadena**

* ^: el match se encuentra al principio de la cadena.
* $: el match se encuentra al final de la cadena.
* \b: el match está al principio de una palabra o tras espacio en blanco. No confundir con ^ $, los cuales buscan match al inicio de cadenas.
* \B: el match está tras espacio en blanco, y no al principio de una palabra.

```
(strings <- c("abcd", "cdab", "cabd", "c abd"))
grep("ab", strings, value = TRUE)
grep("^ab", strings, value = TRUE)
grep("ab$", strings, value = TRUE)
grep("\\bab", strings, value = TRUE)
```

**Operadores**

* .: match con cualquier carácter simple.
* [...]: una lista de caractéres, match con cualquiera de los caracteres dentro de los corchetes. Igualmente se puede usar un rango de caracteres dentro de los corchetes.
* [^...]: lista invertida de caracteres, igual que [...] pero el match es con cualquier carácter excepto los incluidos entre corchetes.
* |: operador "O", se produce el match con cualquiera de los patrones expresados a izquierda y derecha del |.
* (...): agrupación en expresiones regulares. 

Todo esto se entiende mejor con un ejemplo:

```
(strings <- c("^ab", "ab", "abc", "abd", "abe", "ab 12"))
grep("ab.", strings, value = TRUE)
grep("ab[c-e]", strings, value = TRUE)
grep("ab[^c]", strings, value = TRUE)
grep("^ab", strings, value = TRUE)
grep("\\^ab", strings, value = TRUE)
grep("abc|abd", strings, value = TRUE)
gsub("(ab) 12", "\\1 34", strings)
```

**Caracteres de clase**

Nos permiten especificar clases enteras de caracteres, tales como números, letras... Hay dos formas de construirlos, usando [: :] o usando \.

* [:digit:] o \d: dígitos, 0 1 2 3 4 5 6 7 8 9, similar a [0:9]
* \D: no dígitos, equivalente a [^0-9]
* [:lower:]: minúsculas, equivalente a [a-z]
* [:upper:]: mayúsuclas, equivalente a [A-Z]
* [:alpha:]: caracteres alfabéticos, equivalente a [[:lower:][:upper:]] o [A-z]
* [:alnum:]: caracteres alfanumérico, equivalente a [[:alpha][:digit:]] o [A-z0-9]
* \w: palabras, equivalente a [[:alnum]_] o [A-z0-9_]
* \W: no palabras, equivalente a [^A-z0-9_]
* [:blank:]: caractéres en blanco, como espacios o tabuladores
* [:space:]: espacios en blanco: tabulado, nueva linea, tabulado vertical, retorno de carro, espacio.
* \s: espacios.
* \S: no espacios.
* [:punct:]: caracteres de puntuación, ! " # $ % & ’ ( ) * + , - . / : ; < = > ? @ [  ] ^ _ ` { | } ~.

**Estándares de sintaxis**

Hay varios estándares a la hora de construir expresiones literales, y R ofrece dos:

* POSIX extended regular expresions (por defecto)
* Perl-like regular expresions.

Se puede cambiar facilmente de una a otra con perl = FALSE/TRUE en las funciones de R base, tales como grep() o sub(). La sintaxis puede variar entre estos dos estándares, así, si estás acostumbrado a trabajar con Python o Java, probablemente te resulte más comodo trabajar con el modo Perl-like. En este post solo se usan el modo por defecto de R (Posix standar).

Hay otro tipo más de expresiones regulares, las llamadas "fixed". En este modo, el patrón se ha de valorar literalmente. Se especifica con fixed = TRUE en las funciones base de R o usando la función fixed() por ejemplo en las funciones del paquete stringr. Une ejemplo de esto, en el modo estándar, la expresión "A.b" hará match con cualquier cadena que con una A seguida de cualquier carácter seguido de b, pero si usamos el modo fixed, solo habrá match con el literal "A.b".

Veamos esto en código:

```
(strings <- c("Axbc", "A.bc"))
pattern <- "A.b"
grep(pattern, strings, value = TRUE)
grep(pattern, strings, value = TRUE, fixed = TRUE)
```

Por defecto, los patrones en R distinguen entre minúsculas y mayúsculas, pero esto se puede cambiar facilmente con ignore.case = TRUE (para funciones de R base) o con ignore.case() en funciones de otros paquetes. 

Usando el ejemplo anterior:

```
pattern <- "a.b"
grep(pattern, strings, value = TRUE)
grep(pattern, strings, value = TRUE, ignore.case = TRUE)
```
**Actualización**: interesante la [Cheat Sheet de RStudio para expresiones regulares](https://www.rstudio.com/wp-content/uploads/2016/09/RegExCheatsheet.pdf) 

El [artículo oríginal][1] incluye algunos ejemplos para practicar, así como recursos online([este][5] y [este otro][6] que nos pueden ayudar a la hora de construir nuestras expresiones regulares en R.

[1]: http://stat545.com/block022_regular-expression.html
[2]: http://www.rdocumentation.org/packages/stringr/versions/1.1.0
[3]: https://github.com/STAT545-UBC/STAT545-UBC.github.io
[4]: https://stat.ethz.ch/R-manual/R-devel/library/base/html/Quotes.html
[5]: http://www.regexpal.com/
[6]: http://www.regexr.com/
