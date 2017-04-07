Suele ser bastante frecuente encontrarse con dataset en los que muchas variables no tengan todos los datos completos, es decir, que tengan datos perdidos (conocidos como missing values). En muchas ocasiones se hace necesario imputar esos valores faltantes, para lo que existen diferentes métodos.

En este contexto, el paquete de R [mice](https://www.rdocumentation.org/packages/mice/versions/2.25/topics/mice), de Stef van Buuren, nos ofrece un algoritmo capaz de realizar imputaciones  multivariantes, basadas en los valores del resto de variables del dataset.

Para verlo en acción vamos a replicar el ejercicio de Klodian Dhana en [este post](http://datascienceplus.com/handling-missing-data-with-mice-package-a-simple-approach/), y para ello usamos el dataset que el autor ha creado para el ejemplo:

```
dat <- read.csv(url("http://goo.gl/19NKXV"), header=TRUE, sep=",")
```

Comprobamos la existencia y número de valores perdidos en las variables:

```
sapply(dat, function(x) sum(is.na(x)))
```

En este momento, introducimos algunos valores perdidos en el dataset, no sin antes hacer una copia para mantener el original.

```
original <- dat
set.seed(10)
dat[sample(1:nrow(dat), 20), "Cholesterol"] <- NA
dat[sample(1:nrow(dat), 20), "Smoking"] <- NA
dat[sample(1:nrow(dat), 20), "Education"] <- NA
dat[sample(1:nrow(dat), 5), "Age"] <- NA
dat[sample(1:nrow(dat), 5), "BMI"] <- NA
```

Confirmamos que ahora sí hay valores perdidos.

```
sapply(dat, function(x) sum(is.na(x)))
```

En este punto, se realizan las correspondientes transformaciones de las variables en factores y/o continuas. 

```
library(dplyr)
dat <- dat %>%
mutate(Smoking = as.factor(Smoking)) %>%
mutate(Education = as.factor(Education)) %>%
mutate(Cholesterol = as.numeric(Cholesterol))
```

Y echamos un vistazo a la estructura del dataset:

```
str(dat)
```

Ahora que todo está correcto, es hora de comenzar con las imputaciones, y para ello llamamos al paquete mice, usando el siguiente código estándar:

```
library(mice)
init = mice(dat, maxit=0) 
meth = init$method
predM = init$predictorMatrix
```
 
Dado que el algoritmo que usa mice para predecir e imputar los valores se basa en el valor del resto de variables, es posible que no queramos que algunas de ellas participen en esta predicción. En el ejemplo,  es claro que la variable ID no tiene valor predictivo.

Para que sirva como ejemplo, usamos la variable BMI para no ser incluida en las variables predictoras, aunque se imputará.

```
predM[, c("BMI")]=0
```

Si lo que queremos es no imputar algunas de las variables, usaremos este código, aunque seguirá siendo una de las variables predictoras:

```
meth[c("Age")]=""
```

El siguiente paso es especificar los métodos de imputación para las distintas variables, ya que hay distintos métodos en función de si la variables es continua, binaria, ordinal, etc. Por ejemplo:

```
meth[c("Cholesterol")]="norm"
meth[c("Smoking")]="logreg"
meth[c("Education")]="polyreg"
```

Y por fin ejecutamos el la imputación múltiple (m = 5):

```
set.seed(103)
imputed = mice(dat, method=meth, predictorMatrix=predM, m=5)
```

Finalmente, creamos un dataset tras la imputación y comprobamos los valores perdidos:

```
imputed <- complete(imputed)
sapply(imputed, function(x) sum(is.na(x)))
```

En el [ejemplo original](http://datascienceplus.com/handling-missing-data-with-mice-package-a-simple-approach/) podéis ver como se realizan las comprobaciones de la exactitud del proceso, y como varía de unas variables a otras.