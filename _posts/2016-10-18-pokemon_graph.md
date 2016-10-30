---
layout: post
title: "Building a pokemon graph database"
output: github_document
date: "18 octubre, 2016"
image: '/assets/img/'
description: 'Building a pokemon graph with R and Neo4j'
tags:
- R 
- Neo4j 
categories:
- Pokemon
- rstats
---



## GitHub Documents

This is an R Markdown format used for publishing markdown documents to GitHub. When you click the **Knit** button all R code chunks are run and a markdown file (.md) suitable for publishing to GitHub is generated.

## Including Code

You can include R code in the document as follows:


```r
library(RNeo4j)
library(rvest)
library(visNetwork)

source("jbkunst_pokemon.R")

link <- "http://pokemondb.net/type"

link_html <- read_html(link)

types <- link_html %>%
  html_nodes("table") %>%
  .[[1]] %>%
  html_table()

names(types)[1] <- "Type"
types$Type <- tolower(types$Type)
names(types)[2:ncol(types)] <- types$Type
types[is.na(types)] <- 1
types[types == ""] <- 1
types[types == "½"] <- 0.5



df %>% select(id, type =  type_1) -> t1
df %>% select(id, type =  type_2) -> t2

rbind(t1,t2) -> tf

poke_df <- df %>% select(-type_1, -type_2) %>% 
  left_join(tf, by = "id") %>% 
  filter(!is.na(type))


username <- "Pokemon"
url <- "http://pokemon.sb10.stations.graphenedb.com:24789/db/data/"
password <- "4FUa969pxohWdmBk62qH"

graph = startGraph(     url = url,
                   username = username,
                   password = password)
#Delete all nodes
#clear(graph)


addConstraint(graph, "Pokemon", "id")
addConstraint(graph, "Type", "type")


pokenodes <- function(x) {
  pokemon <- getOrCreateNode(graph, "Pokemon", id = x["id"], name = x["pokemon"],
                             height = x["height"], weight = x["weight"],
                             attack = x["attack"], defense = x["defense"],
                             hp = x["hp"], special_attack = x["special_attack"],
                             special_defense = x["special_defense"], speed = x["speed"],
                             url_image = x["url_image"], url_icon = x["url_icon"])
  
  type <- getOrCreateNode(graph, "Type", type = x["type"])
  
  createRel(pokemon,"TYPE",type)
}

apply(poke_df[1:nrow(poke_df),],1,pokenodes)

types <- types %>% gather(Type)

names(types)[2] <- "Type_Rel"

effectiveness <- types %>% filter(value != 1)

query = "
MERGE (n:Type {type:{type_1}})
MERGE (m:Type {type:{type_2}})
CREATE (n)-[r:EFECTIVENESS]->(m)
SET r.value = {value}
"

t = newTransaction(graph)

for (i in 1:nrow(effectiveness)) {
  type_1 = effectiveness[i, ]$Type
  type_2 = effectiveness[i, ]$Type_Rel
  value = effectiveness[i, ]$value
  
  appendCypher(t, 
               query, 
               type_1 = type_1, 
               type_2 = type_2, 
               value  = value)
}

commit(t)
```

## Including Plots

You can also embed plots, for example:



Note that the `echo = FALSE` parameter was added to the code chunk to prevent printing of the R code that generated the plot.
