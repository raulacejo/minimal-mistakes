En este post veremos como recopilar datos de una web para después analizarlos y visualizarlos en R. Para ello usaremos el paquete [rvest](https://www.rdocumentation.org/packages/rvest/versions/0.3.2) y conseguiremos los datos de la [Wikipedia](https://en.wikipedia.org/wiki/Obesity_in_the_United_States).

Primero cargamos todos los paquetes que vamos a necesitar:

```
library(rvest)
library(ggplot2)
library(dplyr)
library(scales)
```

Descargamos los datos de la wikipedia. La primera línea del código llama a los datos de la Wikipedia y la segunda transforma los datos de la tabla que nos interesa en un dataframe:

```
obesity = read_html("https://en.wikipedia.org/wiki/Obesity_in_the_United_States")

obesity = obesity %>%
     html_nodes("table") %>%
     .[[1]]%>%
     html_table(fill=T)
```

Observamos nuestros datos:

```
head(obesity)
```

```
 State and District of Columbia Obese adults Overweight (incl. obese) adults
1                        Alabama        30.1%                           65.4%
2                         Alaska        27.3%                           64.5%
3                        Arizona        23.3%                           59.5%
4                       Arkansas        28.1%                           64.7%
5                     California        23.1%                           59.4%
6                       Colorado        21.0%                           55.0%
  Obese children and adolescents Obesity rank
1                          16.7%            3
2                          11.1%           14
3                          12.2%           40
4                          16.4%            9
5                          13.2%           41
6                           9.9%           51
```

El dataframe tiene buena pinta, pero necesitamos realizar algunos ajustes para facilitar su representación gráfica:

**Limpieza de datos**

```
str(obesity)

'data.frame':	51 obs. of  5 variables:
 $ State and District of Columbia : chr  "Alabama" "Alaska" "Arizona" "Arkansas" ...
 $ Obese adults                   : chr  "30.1%" "27.3%" "23.3%" "28.1%" ...
 $ Overweight (incl. obese) adults: chr  "65.4%" "64.5%" "59.5%" "64.7%" ...
 $ Obese children and adolescents : chr  "16.7%" "11.1%" "12.2%" "16.4%" ...
 $ Obesity rank                   : int  3 14 40 9 41 51 49 43 22 39 ...

# borrar los % y convertir los datos en numéricos
for(i in 2:4){
     obesity[,i] = gsub("%", "", obesity[,i])
     obesity[,i] = as.numeric(obesity[,i])
}

# veamos de nuevo el dataset con los cambios
str(obesity)
'data.frame':	51 obs. of  5 variables:
 $ State and District of Columbia : chr  "Alabama" "Alaska" "Arizona" "Arkansas" ...
 $ Obese adults                   : num  30.1 27.3 23.3 28.1 23.1 21 20.8 22.1 25.9 23.3 ...
 $ Overweight (incl. obese) adults: num  65.4 64.5 59.5 64.7 59.4 55 58.7 55 63.9 60.8 ...
 $ Obese children and adolescents : num  16.7 11.1 12.2 16.4 13.2 9.9 12.3 14.8 22.8 14.4 ...
 $ Obesity rank                   : int  3 14 40 9 41 51 49 43 22 39 ...
```

Vamos a "arreglar" los nombres de las variables, quitando espacios en blanco:

```
names(obesity)
[1] "State and District of Columbia"  "Obese adults"                   
[3] "Overweight (incl. obese) adults" "Obese children and adolescents" 
[5] "Obesity rank"

names(obesity) = make.names(names(obesity))
names(obesity)
[1] "State.and.District.of.Columbia"  "Obese.adults"                   
[3] "Overweight..incl..obese..adults" "Obese.children.and.adolescents" 
[5] "Obesity.rank"
```

El siguiente paso es cargar los datos del mapa:

```
# carga de datos del mapa
states = map_data("state")
str(states)
'data.frame':	15537 obs. of  6 variables:
 $ long     : num  -87.5 -87.5 -87.5 -87.5 -87.6 ...
 $ lat      : num  30.4 30.4 30.4 30.3 30.3 ...
 $ group    : num  1 1 1 1 1 1 1 1 1 1 ...
 $ order    : int  1 2 3 4 5 6 7 8 9 10 ...
 $ region   : chr  "alabama" "alabama" "alabama" "alabama" ...
 $ subregion: chr  NA NA NA NA ...
```

Para unir los dos datasets (obesity y states) por la variable region, necesitamos crear una nueva variable (region) en el dataset obesity.

```
# crear nueva variable region
obesity$region = tolower(obesity$State.and.District.of.Columbia)
```

Ahora si podemos unir los datasets:

```
states = merge(states, obesity, by="region", all.x=T)
str(states)
'data.frame':	15537 obs. of  11 variables:
 $ region                         : chr  "alabama" "alabama" "alabama" "alabama" ...
 $ long                           : num  -87.5 -87.5 -87.5 -87.5 -87.6 ...
 $ lat                            : num  30.4 30.4 30.4 30.3 30.3 ...
 $ group                          : num  1 1 1 1 1 1 1 1 1 1 ...
 $ order                          : int  1 2 3 4 5 6 7 8 9 10 ...
 $ subregion                      : chr  NA NA NA NA ...
 $ State.and.District.of.Columbia : chr  "Alabama" "Alabama" "Alabama" "Alabama" ...
 $ Obese.adults                   : num  30.1 30.1 30.1 30.1 30.1 30.1 30.1 30.1 30.1 30.1 ...
 $ Overweight..incl..obese..adults: num  65.4 65.4 65.4 65.4 65.4 65.4 65.4 65.4 65.4 65.4 ...
 $ Obese.children.and.adolescents : num  16.7 16.7 16.7 16.7 16.7 16.7 16.7 16.7 16.7 16.7 ...
 $ Obesity.rank                   : int  3 3 3 3 3 3 3 3 3 3 ...
```

**Visualizar los datos**

Por fin ya podemos representar gráficamente los datos de frecuencia de obesidad en los adultos:

```
## Realizar el gráfico ####

# adultos
ggplot(states, aes(x = long, y = lat, group = group, fill = Obese.adults)) + 
     geom_polygon(color = "white") +
     scale_fill_gradient(name = "Percent", low = "#feceda", high = "#c81f49", guide = "colorbar", na.value="black", breaks = pretty_breaks(n = 5)) +
     labs(title="Prevalence of Obesity in Adults") +
     coord_map()
``` 

![Mapa 1](http://datascienceplus.com/wp-content/uploads/2016/06/adults.png)

De igual modo podemos proceder para representar la obesidad en los niños:

```
# niños
ggplot(states, aes(x = long, y = lat, group = group, fill = Obese.children.and.adolescents)) + 
     geom_polygon(color = "white") +
     scale_fill_gradient(name = "Percent", low = "#feceda", high = "#c81f49", guide = "colorbar", na.value="black", breaks = pretty_breaks(n = 5)) +
     labs(title="Prevalence of Obesity in Children") +
     coord_map()
```

![Mapa 2](http://datascienceplus.com/wp-content/uploads/2016/06/children.png)


[source](http://datascienceplus.com/visualizing-obesity-across-united-states-by-using-data-from-wikipedia/)
