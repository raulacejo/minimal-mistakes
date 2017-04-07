En este post trabajaremos el tratamiento de texto en R. Lo primero será conseguir los datos, y lo que haremos será tratar con scripts que recogen los diálogos completos, que están embebidos en código html de los que extraeremos lo que necesitamos. Usaremos para ello el paquete [rvest](https://www.rdocumentation.org/packages/rvest/versions/0.3.2).

```
# Cargamos el paquete rvest
library(rvest)
```

Los scripts están escritos de tal forma que podríamos reproducir este ejercicio para cualquier otra serie. 
Establecemos ahora nuestras variables: serie de tv que queremos, directorios, urls de base, etc...

```
# En caso de usar otra serie debemos chequear la url que se usa para esa serie.
tvshow <- "vikings"
 
# Creamos el directorio de descarga y cambiamos a él
directory = paste("~/Data Analysis/files/", tvshow, sep="")
dir.create(directory, recursive = TRUE, showWarnings = FALSE)
setwd(directory)
 
# Url base y url completa
baseurl <- "http://www.springfieldspringfield.co.uk/"
url <- paste(baseurl,"episode_scripts.php?tv-show=", tvshow, sep="")
```

De la serie Vikings hay disponibles cuatro temporadas con 10 episodios cada una. Lo primero que hacemos es scrapear todos las urls de todos los episodios, para ello las buscamos en el código html. Después de la etiqueta h3, el primer href de s01e01 con class="season-episode-title". Y esta sería la clase que estableceremos como nuestro nodo.

```
# leer el HTML
scrape_url <- read_html(url)
# selección del nodo
s_selector <- ".season-episode-title"
 
# scraping de href nodes in .season-episode-title
all_urls_season <- html_nodes(scrape_url, s_selector) %>%
  html_attr("href")
```
Veamos la estructura de all_url_Season:

```
str(all_urls_season)
```
```
##  chr [1:40] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e01&quot; ...
```
```
head(all_urls_season)
```
```
## [1] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e01&quot;
## [2] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e02&quot;
## [3] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e03&quot;
## [4] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e04&quot;
## [5] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e05&quot;
## [6] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s01e06&quot;
```
```
tail(all_urls_season)
```
```
## [1] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s04e05&quot;
## [2] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s04e06&quot;
## [3] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s04e07&quot;
## [4] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s04e08&quot;
## [5] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s04e09&quot;
## [6] &quot;view_episode_scripts.php?tv-show=vikings&amp;episode=s04e10&quot;
```
Podemos ver como efectivamente tenemos 40 urls de episodios (4 temporadas, 10 episodios cada una). Ahora que ya tenemos todas las variables y las urls, podemos empezar tratar los scripts y guardarlos en distintos archivos para hacer el tratamiento de texto.

```
# Loop a través de las urls de cada temporada
for (i in all_urls_season) {
  uri <- read_html(paste(baseurl, i, sep="/"))
  # para saber que nodo tenemos que seleccionar, inspeccionar el site (html)
  script_selector <- ".scrolling-script-container"
  # scraping a todo el script de texto, lo guardamos en una variable 
  text <- html_nodes(uri, script_selector) %>% 
    html_text()
 
# Tomamos los últimos cinco caracteres de all_urls_season como temporada para grabarlos en archivos separados
substrRight <- function(x, n) {
  substr(x, nchar(x)-n+1, nchar(x))
  }
seasons <- substrRight(i, 5)
# Escribir cada script a un archivo de texto separado
write.csv(text, file = paste(directory, "/", tvshow, "_", seasons, ".txt", sep=""), row.names = FALSE)
}
```
### Empezando el tratamiento de texto

Ahora que ya tenemos todos los archivos de texto podemos empezar con el tratamiento, para ello vamos a usar el paquete [tm](https://www.rdocumentation.org/packages/tm/versions/0.6-2).
```
# cargamos el paquete tm
library(tm)

# establecemos los filepath donde están los scripts
cname <- file.path(directory)
# comprobamos si los filepath contienen nuestros scripts
(docname <- dir(cname))

# Creamos un Corpus de los archivos de texto para poder hacer analisis
docs <- Corpus(DirSource(cname), readerControl = list(id=docname))
# Mostramos un sumario del Corpus, tenemos que tener 40 documentos
summary(docs)

# Inspeccionamos el primer documento
inspect(docs[1])
```

Los scripts contienen mucha información que no necesitamos y no es útil para el tratamiento de texto, por tanto, necesitamos limpiarlo un poco. Borramos números, convertimos el texto a minúscula, borramos signos de puntuación y palabras vacías, en este caso todo está en inglés.

```
docs <- tm_map(docs, tolower)
docs <- tm_map(docs, removePunctuation)
docs <- tm_map(docs, removeNumbers)
docs <- tm_map(docs, removeWords, stopwords("english"))
```

Ahora hacemos stemming, que es un método para reducir las palabras a su raíz. Se puede ver como ejemplo de esto las palabras waits, waited, waiting, todas ellas procedentes de la raíz wait.

```
library(SnowballC)
docs <- tm_map(docs, stemDocument)
```

En todo este proceso hemos borrado muchos caracteres, y se han generado muchos espacios en blanco, por lo que ahora los borramos también.

```
docs <- tm_map(docs, stripWhitespace)
```

Volvemos ahora a echar un vistazo a nuestro archivo.

```
inspect(docs[1])
```

Estamos preparados ya para convertir de nuevo el archivo a texto plano.

```
docs <- tm_map(docs, PlainTextDocument)
```

Creamos ahora una Matriz de Términos de Documento, la cual recoge el número de veces que cada término del Corpus aparece en cada uno de los documentos. Y le añadimos igualmente los nombres a algunas columnas:

```
# Creamos la matriz (tdm = Term Document Matrix)
tdm <- TermDocumentMatrix(docs)
# Añadimos nombres de columna 
docname <- gsub("vikings_", "",docname)
docname <- gsub(".txt", "",docname)
docname <- paste("s",docname, sep="")
colnames(tdm) <- docname
# Show and inspect the tdm
tdm
inspect(tdm[1:10,1:6])
```

Ahora construimos una Matriz de Documentos y Términos, que no es más que la traspuesta de tdm.

```
dtm <- DocumentTermMatrix(docs)
rownames(dtm) <- docname
dtm
inspect(dtm[1:10,1:6])
```

En este punto ya somos capaces de contestar preguntas como cual es el término que más se repite o encontrar como se asocian entre ellos.

### Frecuencia de términos 

Vamos a ver los términos más utilizados en la serie, por ejemplo el top 20.

```
freq <- sort(colSums(as.matrix(dtm)), decreasing=TRUE)
head(freq,20)
```

Añadimos esto a un dataframe para poder visualizarlo. 

```
tf <- data.frame(term=names(freq), freq=freq)   
head(tf,20)  
```

Y lo pintamos en un gráfico.

```
tf$term <- factor(tf$term, levels = tf$term[order(-tf$freq)])
library(ggplot2)
p <- ggplot(subset(tf, freq>200), aes(term, freq))    
p <- p + geom_bar(stat="identity")   
p <- p + theme(axis.text.x=element_text(angle=45, hjust=1))   
p
```
![Gráfico 1](http://www.networkx.nl/wp-content/uploads/2016/07/unnamed-chunk-16-1.png)
 Vemos como "will" es el término que más aparece en la serie, seguido del nombre del protagonista, "ragnar".

### Análisis complementario

Como pudimos ver anteriormente cuando observamos la Matriz de Términos del Documento (tdm), tenemos muchos términos dispersos (90%). Esto es mucho, por lo que procedemos a eliminarlo.

```
tdm.common = removeSparseTerms(tdm, sparse = 0.1)
tdm
tdm.common
```

Conseguimos bajar la dispersión a un 3%. Podemos ver cuántos términos teníamos antes y cuántos tenemos ahora.

```
dim(tdm)
dim(tdm.common)
```

De 5.608 términos hemos pasado a 36. Inspeccionamos los primeros 10 términos de los primeros 6 documentos.

```
inspect(tdm.common[1:10,1:6])
```

Vamos a ver ahora los términos más utilizados mediante un mapa de calor, utilizando ggplot2. Como este paquete trabaja con matrices, necesitamos convertir tdm.common.

```
tdm.dense <- as.matrix(tdm.common)
dim(tdm.dense)
```

Para poder visualizar, necesitamos los datos en una matriz normal.

```
library(reshape2)
tdm.dense.m <- melt(tdm.dense, value.name = "count")
head(tdm.dense.m)
```

Ahora si podemos montar el mapa de calor.
 
```
library(ggplot2)
ggplot(tdm.dense.m, aes(x = Docs, y = Terms, fill = log10(count))) +
     geom_tile(colour = "white") +
     scale_fill_gradient(high="steelblue" , low="white")+
     ylab("") +
     theme(panel.background = element_blank()) +
     theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))

```
![Gráfico 2](http://www.networkx.nl/wp-content/uploads/2016/07/unnamed-chunk-22-1.png)
Con este gráfico podemos ver cuales de los términos más comunes se usan en cada episodio.

Vamos a pintar ahora un correlograma de los episodios. Un correlograma es un gráfico de la correlación de una matriz, es muy útil para destacar las variables más correlacionadas en una dataset. En este gráfico, los coeficientes de correlación se colorean en base a sus valores.

```
corr <- cor(tdm.dense)
library(corrplot)
corrplot(corr, method = "circle", type = "upper", tl.col="black", tl.cex=0.7)
```
![Correlograma](http://www.networkx.nl/wp-content/uploads/2016/07/unnamed-chunk-23-1.png)

Si transponemos el dataset tdm.dense podemos hacer un correlograma para los términos.

```
tdm.dense.t <- t(tdm.dense)
corr.t <- cor(tdm.dense.t)
corrplot(corr.t,method = "circle", type = "upper", tl.col="black", tl.cex=0.7)
```

Como véis, se pueden hacer muchos análisis basados en texto, pero por ahora, lo vamos a dejar aquí.

[Post original](http://www.networkx.nl/data-science/text-mining-r-vikings/)
