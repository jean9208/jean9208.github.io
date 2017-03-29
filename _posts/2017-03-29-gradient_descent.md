---
layout: post
title: "Gradient Descent"
output: html_document
date: "29 marzo, 2017"
image: '/assets/img/posts/'
description: 'Simple Gradient Descent on R for linear regression'
tags:
- R 
- Highcharts
categories:
- Algorithms
comments: true
---

## Trying gradient descent for linear regression

The best way to learn an algorith is to code it. So here it is, my take on Gradient Descent Algorithm for simple linear regression.

First, we fit a simple linear model with lm for comparison with gradient descent values.


```r
#Load libraries

library(dplyr)
library(highcharter)

#Scaling length variables from iris dataset.

iris_demo <- iris[,c("Sepal.Length","Petal.Length")] %>%
  mutate(sepal_length = as.numeric(scale(Sepal.Length)),
         petal_length = as.numeric(scale(Petal.Length))) %>%
  select(sepal_length,petal_length)

#Fit a simple linear model to compare coefficients.

regression <- lm(iris_demo$petal_length~iris_demo$sepal_length)

coef(regression)
```

```
##            (Intercept) iris_demo$sepal_length 
##           4.643867e-16           8.717538e-01
```

```r
iris_demo_reg <- iris_demo

iris_demo_reg$reg <- predict(regression,iris_demo)

#Plot the model with highcharter

highchart() %>%
  hc_add_series(data = iris_demo_reg, type = "scatter", hcaes(x = sepal_length, y = petal_length), name = "Sepal Length VS Petal Length") %>%
  hc_add_series(data = iris_demo_reg, type = "line", hcaes(x = sepal_length, y = reg), name = "Linear Regression") %>%
  hc_title(text = "Linear Regression")
```

<iframe src="/html_wg/gradient_descent/linear_regression.html" height="500"></iframe> <a href="/html_wg/gradient_descent/linear_regression.html" target="_blank">open</a>

We will try to acomplish the same coefficients, this time using Gradient Descent.


```r
library(tidyr)


set.seed(135) #To reproduce results


#Auxiliary function

# y = mx + b

reg <- function(m,b,x)  return(m * x + b)


#Starting point

b <- runif(1)
m <- runif(1)


#Gradient descent function

gradient_desc <- function(b, m, data, learning_rate = 0.01){ # Small steps
  
  # Column names = Code easier to understand
  
  colnames(data) <- c("x","y")
  
  
  #Values for first iteration
  
  b_iter <- 0     
  m_iter <- 0
  n <- nrow(data)
  
  # Compute the gradient for Mean Squared Error function
  
  for(i in 1:n){
    
    # Partial derivative for b
    
    b_iter <- b_iter + (-2/n) * (data$y[i] - ((m * data$x[i]) + b))
    
    # Partial derivative for m
    
    m_iter <- m_iter + (-2/n) * data$x[i] * (data$y[i] - ((m * data$x[i]) + b))
    
  }
  
  
  # Move to the OPPOSITE direction of the derivative
  
  new_b <- b - (learning_rate * b_iter)
  new_m <- m - (learning_rate * m_iter)
  
  # Replace values and return
  
  new <- list(new_b,new_m)
  
  return(new)
  
}

# I need to store some values to make the motion plot

vect_m <- m
vect_b <- b


# Iterate to obtain better parameters

for(i in 1:1000){
  if(i %in% c(1,100,250,500)){ # I keep some values in the iteration for the plot
    vect_m <- c(vect_m,m)
    vect_b <- c(vect_b,b)
  } 
  x <- gradient_desc(b,m,iris_demo)
  b <- x[[1]]
  m <- x[[2]]
}

print(paste0("m = ", m))
```

```
## [1] "m = 0.871753774273602"
```

```r
print(paste0("b = ", b))
```

```
## [1] "b = 5.52239677041512e-10"
```

The difference in the coefficients is minimal. 

We can see how the iterations work in the next plot:


```r
#Compute new values

iris_demo$preit    <- reg(vect_m[1],vect_b[1],iris_demo$sepal_length)
iris_demo$it1      <- reg(vect_m[2],vect_b[2],iris_demo$sepal_length)
iris_demo$it100    <- reg(vect_m[3],vect_b[3],iris_demo$sepal_length)
iris_demo$it250    <- reg(vect_m[4],vect_b[4],iris_demo$sepal_length)
iris_demo$it500    <- reg(vect_m[5],vect_b[5],iris_demo$sepal_length)
iris_demo$finalit  <- reg(m,b,iris_demo$sepal_length)


iris_gathered <- iris_demo %>% gather(key = gr, value = val, preit:finalit) %>%
  select(-petal_length) %>% 
  distinct()


iris_start <- iris_gathered %>%
  filter(gr == "preit")


iris_seq <- iris_gathered %>%
  group_by(sepal_length) %>%
  do(sequence = list_parse(select(., y = val)))


iris_data <- left_join(iris_start, iris_seq)

#Motion Plot

irhc2 <- highchart() %>%
  hc_add_series(data = iris_data, type = "line", hcaes(x = sepal_length, y = val), name = "Gradient Descent") %>%
  hc_motion(enabled = TRUE, series = 0, startIndex = 0,
            labels = c("Iteration 1","Iteration 100","Iteration 250","Iteration 500","Final Iteration")) %>%
  hc_add_series(data = iris_demo_reg, type = "scatter", hcaes(x = sepal_length, y = petal_length), name = "Sepal Length VS Petal Length") %>%
  hc_title(text = "Gradient Descent Iterations")

irhc2
```

<iframe src="/html_wg/gradient_descent/gradient_descent_iterations.html" height = "500"></iframe> <a href="/html_wg/gradient_descent/gradient_descent_iterations.html" target="_blank">open</a>


Maybe in a future post we can try a multivariate regression model!
