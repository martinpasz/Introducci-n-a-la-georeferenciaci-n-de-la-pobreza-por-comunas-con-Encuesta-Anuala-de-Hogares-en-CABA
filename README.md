---
title: "Introducción a georeferenciación en R"
author: "Martín Pasztetnik"
email: "martinpasz@gmail.com"
date: "2023-11-01"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

# 1. Introducción

La **información georreferenciada** desempeña un papel importante en la toma de decisiones, la comprensión de fenómenos y la resolución de problemas en diversas disciplinas. Permite **visualizar datos en un contexto espacial**, lo que facilita la **identificación de patrones, tendencias y relaciones** que pueden pasar desapercibidas en datos tabulares. El uso de R para analizar y visualizar datos geoespaciales proporciona una herramienta poderosa para explorar, comunicar y tomar decisiones basadas en la ubicación, lo que hace que la enseñanza de este tema sea mucho muy importante en la formación de cualquier persona interesada en aprovechar la información espacial.

# 2. Librerías a utilizar

### Tidyverse

<https://www.tidyverse.org/>

-   El paquete "tidyverse" es una colección de paquetes de R diseñados para simplificar y mejorar el flujo de trabajo en análisis de datos. Incluye herramientas como "dplyr" para la manipulación de datos, "ggplot2" para la creación de gráficos, "tidyr" para la limpieza y organización de datos, y otros paquetes relacionados. En el caso de no tenerlo instalado, se realiza de la siguiente forma:

```{r warning=FALSE, eval=FALSE}
  install.packages("tidyverse")
```

### Simple Features (sf)

<https://r-spatial.github.io/sf/>

-   El paquete "sf" en R es una herramienta para trabajar con datos geoespaciales. Ofrece funcionalidades para la manipulación, análisis y visualización de información geográfica, permitiendo a los usuarios manejar datos espaciales como puntos, líneas y polígonos. "sf" se integra con otras bibliotecas de R, como "ggplot2" y "dplyr", lo que facilita la creación de gráficos geoespaciales y el análisis de datos georreferenciados. En el caso de no tenerlo instalado, se realiza de la siguiente forma:

```{r eval=FALSE, warning=FALSE, eval=FALSE}
  install.packages("sf")
```

### Cairo (opcional)

<https://github.com/s-u/Cairo>

El paquete "Cairo" en R proporciona una interfaz para la biblioteca de gráficos Cairo, que ofrece capacidades avanzadas de renderización de gráficos en distintos formatos, incluyendo PNG, PDF, SVG y más. Permite a los usuarios de R crear gráficos de alta calidad con opciones de anti-aliasing y soporte para una amplia gama de fuentes y estilos. "Cairo" es especialmente útil para la generación de gráficos vectoriales y de mapa de bits con calidad de publicación.

```{r eval=FALSE, warning=FALSE, eval=FALSE}
  install.packages("Cairo")
```

# 3. Mapa de las comunas (o zonas o cualquier mapa...)

El **formato SHP** (Shapefile) es uno de los estándares más comunes para **almacenar datos geoespaciales**. Un **archivo SHP consta de varios componentes esenciales**, incluyendo el archivo principal con extensión .shp que almacena la geometría de las entidades geoespaciales, como puntos, líneas o polígonos. Junto con el archivo .shp, se encuentran archivos adicionales como .shx (índice espacial), .dbf (tabla de atributos) y posiblemente otros archivos opcionales que almacenan información adicional, como proyecciones o índices de búsqueda. La combinación de estos archivos permite almacenar y gestionar de manera eficiente datos espaciales complejos que incluyen tanto la ubicación geográfica como atributos relacionados, lo que resulta valioso en una amplia variedad de aplicaciones de GIS (Sistemas de Información Geográfica).

Acceso a shape de por **COMUNAS** de la Ciudad de Buenos Aires en formato SHP (Shapefile):

[![](images/descarga_mapas.JPG)](https://data.buenosaires.gob.ar/dataset/comunas)

# 4. Dataset (Encuesta Anual de Hogares - DGEyC CABA)

<https://www.estadisticaciudad.gob.ar/eyc/?page_id=702>

Es una encuesta anual que realiza la DGESYC desde 2002. Se trata de un operativo por muestreo que involucra un número importante de viviendas particulares distribuidas en el territorio de la Ciudad. La muestra está diseñada de manera tal que los resultados de la encuesta permiten una representatividad del total de la Ciudad y de cada una de sus comunas.

Se propone recabar datos para conocer y analizar la situación socioeconómica y demográfica de la población y de los hogares de la Ciudad. Conocer la situación del mercado laboral y el estado y la dinámica demográfica de la población es necesario para el diseño informado de la gestión pública.

# 5. Proceso

#### 5.1 Creación de un proyecto

El ejemplo que voy a presentar contempla el **mapeo de la distribución de la pobreza total** (pobreza no indigente + indigencia) **en las comunas de la Ciudad de Buenos Aires.** Para tal fin, y como buena práctica en R, lo primero es siempre **crear una carpeta general del proyecto**, con cierto orden para los diferentes tipos de archivos. Obviamente, esta parte no es necesaria y se puede tener todo en la misma carpeta. En mi caso generalmente los ordeno por:

![](images/04_proyecto.JPG)

/datasets -\> donde coloco mis txt, excel o csv\
/mapa -\> donde descomprimo el zip con todos los archivos de las comunas (todos eh, no solamente el shp)\
/outputs -\> donde exporto mis gráficos, o base de datos modeladas\
/scripts -\> donde guarto básicamente el proceso (en este caso, este mismo archivo que estoy armando ahora)

Una vez creada la carpeta principal y las subcarpetas -con los nombres que quieran-, lo ideal es siempre empezar con un **Proyecto en R**. Crear un proyecto en R en lugar de simplemente cambiar el directorio de trabajo con \`setwd()\` ofrece varias ventajas. Un proyecto en R, que generalmente se crea utilizando [RStudio](https://www.rstudio.com/tags/rstudio-ide/), proporciona un e**ntorno de trabajo organizado y coherente**. En un proyecto, **todos los archivos y carpetas relacionados con tu análisis de datos se almacenan en un directorio específico**, lo que facilita la colaboración y la reproducibilidad. Además, RStudio guarda **configuraciones específicas del proyecto, como el entorno, el historial y las rutas**, lo que evita conflictos entre diferentes proyectos y te permite trabajar en varios proyectos sin confusiones:

**a.** Crear proyecto

![](images/01_proyecto.JPG)

**b.** Elegir el directorio de trabajo existente (opción 2)

![](images/02_proyecto.JPG)

c\. Después cada vez que abra Rstudio, solo tengo que ir a **File \> Open Project** y listo.

#### 5.2 Cargar librerías necesarias

```{r warning=FALSE}
library(tidyverse)
library(readr)
library(sf)
library(Cairo)
```

#### 5.3 Cargamos los datasets de la EAH del año que hayamos descargado

```{r}
# Cargamos primero el dataset de individuos

eah2022_ind <- read.table("datasets/eah2022_bu_ampliada_ind.txt", # Ruta de archivo con extensión
                          header = TRUE, # Encabezado
                          sep = ";", # Separador
                          dec = ",") # Decimales

# Cargamos el dataset de hogares

eah2022_hog <- read.table("datasets/eah2022_bu_ampliada_hog.txt", # Ruta de archivo con extensión
                          header = TRUE, # Encabezado
                          sep = ";", # Separador
                          dec = ",") # Decimales

              
```

Y ahora lo que tenemos que hacer es pegarle las propiedades del hogar ([porque la pobreza se mide por hogar](https://www.estadisticaciudad.gob.ar/eyc/?page_id=120212#lineadeindigencia))

#### 5.4 Unimos las bases de hogar con individuos

```{r}
eah2022_combinada <- left_join(eah2022_ind, eah2022_hog) # Usamos left join
```

#### 5.5 Cargamos el mapa de Comunas de la Ciudad

```{r}
mapa_comunas <- read_sf("mapa/comunas_wgs84.shp") # Cargamos solamente el archivo con extensión shp
```

#### 5.6 Análisis exploratorio de datos (EDA)

Ahora lo que habría que hacer es pegar la base combinada con hogares e individuos de la EAH al mapa de comunas de la Ciudad. Este paso se realiza siempre de esa manera: base eah sobre mapa y no al revés. Cuando se realizó el matcheo de la EAH individuos con hogares, automáticamente dplyr detectó por que nombre de variables pegaba las coincidencias (by = join_by(id, nhogar, comuna, dominio, fexp). Entonces la primera condición para la fusión de objetos es que tengan los mismos nombres de variables (y tipos).

Vamos a ver como es la variable de comunas de la EAH:

```{r}
names(eah2022_combinada) # Función para ver solamente los nombres de las variables/columnas de un dataset
```

Una vez que identificamos el nombre de la variable que necesitamos, vemos de que clase de variable es y que composición tiene

```{r}
glimpse(eah2022_combinada$comuna)

```

La variable comuna tiene 12.501 observaciones/filas y es integer o numérica. Vamos a ver cuantas categorías tiene:

```{r}
eah2022_combinada %>%
  count(comuna)
```

Efectivamente tiene 15 categorías, cada una con una serie de observaciones que suman las 12.501 que contamos antes. Si bien las bases de la DGEyC del GCBA están consolidadas y no presentan errores de consistencia (nulos, vacíos, etc), nunca está de más hacer un análisis exploratorio de los datos para ver si está todo OK.

Ahora deberíamos repetir exactamente el mismo proceso con el mapa, ya que para que la combinación de objetos funcionen, estos tienen que tener el mismo nombre y ser del mismo tipo.

```{r}
mapa_comunas

```

Como podemos ver, el objeto mapa es pequeño y lo podemos visualizar sin problemas entero. En primer lugar observamos que la variable/columa de comuna está escrita "COMUNAS" en mayúsculas y plural y que el tipo de dato es double (otro tipo de dato numérico). Ya que los nombres de las categorías en ambas bases (EAH y MAPA) son iguales (1, 2, 3, 4, 5...) y que además son valores numéricos, lo unico que restaría hacer es cambiarle el nombre a la variable/columna de la EAH y para eso hacemos:

```{r}
eah2022_combinada <- eah2022_combinada %>% 
  rename(COMUNAS = comuna) # Utilizamos la función rename de dplyr con nombre_nuevo = nombre_viejo
```

#### 5.7 Cálculo de pobreza (o cualquier otro cálculo)

Calculamos -en este caso particular- el porcentaje de pobreza total por comuna en un nuevo objeto para posteriormente pegarlo al mapa. Para ello primero generamos una variable de pobreza dicotómica (pobre - no pobre). Hay muchísimas maneras de hacer esto, una sencilla a motivos prácticos es la siguiente con la función mutate de dplyr:

```{r}
eah2022_combinada <- eah2022_combinada %>% 
  mutate(pobreza_total = case_when(i_estratos == 1 | i_estratos == 2 ~ "1. Pobre",
                                   i_estratos >= 3 ~ "2. No pobre"))

# Vemos como nos quedó en el dataset (sin ponderar):

eah2022_combinada %>%
  count(pobreza_total)
```

Ahora generamos el summarize para calcular el porcentaje de pobreza por comunas:

```{r}
tabla_pobreza_comunas <- eah2022_combinada %>% # Generamos un nuevo objeto
  group_by(COMUNAS, pobreza_total) %>% # Agrupamos por comuna y pobreza dicotomica
  summarise(n = sum(fexp)) %>%  # Sumamos los factores de expansión de la base ponderada
  mutate(porcentaje = round(n / sum(n)*100,1)) %>%  # Generamos una variable de porcentaje con un redondeo de 1 dígito
  filter(pobreza_total != "2. No pobre") # Filtramos a los 2. No pobres una vez realizado el cálculo

tabla_pobreza_comunas
  
```

#### 5.8 Pegado de resultados al mapa

Solo faltaría pegar los resultados de pobreza por comuna al mapa, para ello generamos un nuevo objeto (también se puede sobreescribir el mapa, pero no recomiendo perder el original)

```{r}
mapa_comunas_pobreza <- left_join(mapa_comunas, tabla_pobreza_comunas)
```

Automáticamente detecta que la unión se realiza por la variable "COMUNAS". Si no hubiesemos cambiado el nombre, no sabría como matchear ambas bases y mostraría un error.

#### 5.9 Gráficar

"ggplot" es una popular librería en R para la creación de gráficos de alta calidad y versatilidad. Con su enfoque en la gramática de gráficos, "ggplot" permite a los usuarios construir visualizaciones de datos personalizadas y estilizadas con facilidad, lo que lo convierte en una herramienta esencial para la representación visual efectiva de datos. La gramática de ggplot es bastante sencilla y opera en capas, como una cebolla (de abajo para arriba):

![](screenshots/ggplot_add.png)

Para graficar nuestro mapa de pobreza por comunas:

```{r warning=FALSE}
grafico_pobreza_comunas <-  ggplot()+ # Definimos un ggplot
  geom_sf(data = mapa_comunas_pobreza, # definimos el tipo de objeto geom, en este caso el mapa
  aes(fill = porcentaje))+ # Definimos el relleno para cada comuna
  geom_sf_text(data = mapa_comunas_pobreza, aes(label = paste0("COMUNA ", COMUNAS)), color = "gray25", size = 2, nudge_y = 0.003)+ # Ponemos el nombre a cada comuna
  geom_sf_text(data = mapa_comunas_pobreza, aes(label = paste0(porcentaje, "%")), color = "black", size = 4, fontface = "bold", nudge_y = -0.001) + # Ponemos el porcentaje a cada comuna
  scale_fill_gradientn(colors = c("lightyellow", "gold", "darkorange", "#CD2626"), values = scales::rescale(c(0, 13, 26, 50)), name = "") + # Definimos una paleta de colores peresonalizada (opcional) y redefinimos la escala para esa paleta
  theme_void()+ # Cambiamos el tema 
  labs(subtitle = "Personas en situación de pobreza por comunas. En %.") # Escribimos un título para el gráfico

grafico_pobreza_comunas



```

Para salvar el mapa en vector y por ende, en muchisima mayor calidad, utilizamos la librería Cairo con la siguiente función:

```{r eval=FALSE}
ggsave(grafico_pobreza_comunas, filename="outputs/grafico_pobreza_comunas.pdf", 
       width=6, height=6, units="in", device=cairo_pdf)
```

En este caso estamos exportando un PDF vectorizado. Podemos ajustar el parámetro del tamaño también. El resultado sería este:

![](screenshots/comunas.JPG)

Fin.
