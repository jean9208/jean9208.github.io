---
layout: post
title: "Postgresql + R Sandbox"
output: html_document
date: "24 septiembre, 2017"
description: 'Using ElephantSQL host databases for R'
tags:
- R 
- SQL 
categories:
- Databases
comments: true
---



## ElephantSQL

[ElephantSQL](https://www.elephantsql.com/) offers a free instance of Postgresql, with a limit of 20 MB and 5 concurrent connections. For example, you can upload a shiny application that depends on data from ElephantSQL.

You only need to register to the site and automatically you can acces your free instance.

In this post we will see how to take advantage of this cloud database.


## Getting the data

For this example I will use the open data of air quality available in the page of SEDEMA (Environment Secretary) of Mexico City.

The data is structured by one csv file per year, and is avalilable from 1992.


```r
#Auxiliary function to download the files

load_sedema <- function(year){

  #URL to the file
  #from 1992
  link <- paste0("http://148.243.232.112:8080/opendata/IndiceCalidadAire/indice_",year,".csv") 
  
  #Columns classes
  types <- c("character", rep("numeric",26))
  
  #Download the file
  air_data <- read.csv(link,skip = 9, stringsAsFactors = F, encoding = "latin1", header = F,
                       colClasses = types, na.string = "NA")
  
  #Remove missing data
  air_data <- air_data[!air_data[,1]=="",1:27]
  
  #Fix time variable
  air_data$V1 <- paste0(substring(air_data$V1, 1, 6), year) #We need to asure that all dates are from the specified year
  
  return(air_data)

}
```

Next step is to create the table on Postgresql, now that we know thw structure of the csv.


```r
library(RPostgreSQL)


# SQL query to create main table if it not exists

"
CREATE TABLE IF NOT EXISTS air_quality (
  FECHA date,  
  HORA integer,
  NO_OZONO integer,
  NO_AZUFRE integer,
  NO_NITROGENO integer,
  NO_CARBONO integer,
  NO_PM10 integer,
  NE_OZONO integer,
  NE_AZUFRE integer,
  NE_NITROGENO integer,
  NE_CARBONO integer,
  NE_PM10 integer,
  CE_OZONO integer,
  CE_AZUFRE integer,
  CE_NITROGENO integer,
  CE_CARBONO integer,
  CE_PM10 integer,
  SO_OZONO integer,
  SO_AZUFRE integer,
  SO_NITROGENO integer,
  SO_CARBONO integer,
  SO_PM10 integer,
  SU_OZONO integer,
  SU_AZUFRE integer,
  SU_NITROGENO integer,
  SU_CARBONO integer,
  SU_PM10 integer,
  ID serial,
  PRIMARY KEY (ID)
)
" -> query

#Be sure to change your credentials! You can check them on the Details window on your ElephantSQL instance!

#dbname is the user & default database
#host is the serve
#you can get the port from URL

# Connect to database

drv <- dbDriver("PostgreSQL")

con <- dbConnect(drv, dbname = user, 
                 host = db_url, port = 5432,
                 user = user, password = pwd)


# Create table

dbGetQuery(con, query)
```

Next we upload the table from one year


```r
data = upload_sedema(2017)

#Correct format for date
data$V1 <- strptime(data$V1, "%d/%m/%Y")
data$V1 <- gsub("/","-",data$V1)

#Set ID
data$id <- seq(ind,nrow(data) + ind - 1)

#Upload data
  
dbWriteTable(conn = con, name = "air_quality",value = data, append = T, row.names = F)
```

Now you can upload all of the years! Be sure to check the [full script](https://github.com/jean9208/Mexico-City-Air-Quality/blob/master/bulk_import.R)

<img src="{{ site.url }}/assets/img/posts/elephantsql.png" title="elephantsql" alt="elephantsql" style="display: block; margin: auto;">

We can query the data now.

```r
query <- 
'
SELECT 
  * 
FROM 
  "public"."air_quality" 
LIMIT 100
  
'

last100 <- dbGetQuery(con, query)

head(last100)

# Close the connection
  
on.exit(dbDisconnect(con)
```


```
## Loading required package: methods
```

```
## Loading required package: DBI
```

```
##        fecha hora no_ozono no_azufre no_nitrogeno no_carbono no_pm10
## 1 1992-04-01    7       55        34           10         43      NA
## 2 1992-04-01    8       72        39           15         46      NA
## 3 1992-04-01    9       80        44           25         52      NA
## 4 1992-04-01   10       84        48           31         62      NA
## 5 1992-04-01   11      161        43           45         73      NA
## 6 1992-04-01   12      250        41           42         82      NA
##   ne_ozono ne_azufre ne_nitrogeno ne_carbono ne_pm10 ce_ozono ce_azufre
## 1       70        24           19         43      NA       56        39
## 2       68        25           21         43      NA       56        37
## 3       62        35           30         46      NA       68        41
## 4       47        40           33         47      NA       85        43
## 5       81        37           28         47      NA      123        45
## 6       89        32           19         47      NA      185        38
##   ce_nitrogeno ce_carbono ce_pm10 so_ozono so_azufre so_nitrogeno
## 1           20         46      NA       34        26            9
## 2           23         45      NA       46        29           10
## 3           36         48      NA       54        32           15
## 4           64         55      NA       62        34           26
## 5           50         59      NA       81        35           19
## 6           38         62      NA      124        35           16
##   so_carbono so_pm10 su_ozono su_azufre su_nitrogeno su_carbono su_pm10 id
## 1         27      NA       25        18           16         64      NA  1
## 2         31      NA       31        20           18         65      NA  2
## 3         38      NA       32        24           21         65      NA  3
## 4         45      NA       42        26           36         65      NA  4
## 5         47      NA       69        24           40         66      NA  5
## 6         49      NA       55        22           27         67      NA  6
```

I hope this little example can help you to try PostgreSQL even if you don't have it installed on your computer or if you don't have a server.
