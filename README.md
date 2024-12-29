## 2018 Customer Data <br> Linear Regression Predictive Modeling
## Kimberly Waldron  |  kimberly.h.waldron@gmail.com


## Background
The data (customer_data_2018.csv) come from the 2018 sales of a small wholesale business. The variables in the raw data set include:   
   
Variable       |   Type      |    Meaning
---------------|-------------|------------------------------------------
Customer.id    |    integer  |    Unique customer ID
State          |    string   |    State customer operates in
Last.yr.sales  |    numeric  |    2018 sales amount
Prior.yr.sales |    numeric  |    2017 sales amount
Life.orders    |    numeric  |    Total amount of sales from customer over the lifetime of the relationship
Entry.date     |    date     |    Date the customer made a first purchase
Cust.type.id   |    string   |    Descriptive word to describe customer's business type
  
  
The following models are used:        
**Simple Linear Regression**: a model that estimates the relationship between two continuous variables (one dependent and one independent) using a straight line.     
$$
Y_i \sim \beta_0 + X_i\beta_1 + \epsilon_i
$$

$$
\epsilon_i \sim N(0,\sigma^2)
$$
**Multiple Linear Regression**: a model that estimates the relationship between one dependent and two or more independent continuous variables using a straight line.      

$$
Y_i \sim \beta_0 + X_i\beta_1 + X_i\beta_2 + ... + \epsilon_i
$$

$$
\epsilon \sim N(0,\sigma^2)
$$

         
The following question is answered:     
  - **Which model best predicts company sales?**




***
## I. Load Required Libraries 
```{r setup, message=FALSE}
library(tidyverse)
library(dplyr)
library(lubridate)
library(kableExtra)
```


***
## II. Load Raw Data
```{r dd}
data_raw <- read.csv("customer_data_2018.csv", header = T, stringsAsFactors = F) %>% rename(Sales.2018 = Last.yr.sales)%>% rename(Sales.2017 = Prior.yr.sales)
data<-data_raw
data<-data[-c(1, nrow(data)),]
```
***

## III. Define Useful Functions
### (i) Significance Tests 
 $H_{o}: \rho = 0$   
 $H_{a}: \rho ≠ 0$

```{r ada}
# correlation test  
cor.mtest <- function(mat, ...) {
    mat <- as.matrix(mat)
    n <- ncol(mat)
    p.mat<- matrix(NA, n, n)
    diag(p.mat) <- 0
    for (i in 1:(n - 1)) {
        for (j in (i + 1):n) {
            tmp <- cor.test(mat[, i], mat[, j], ...)
            p.mat[i, j] <- p.mat[j, i] <- tmp$p.value
        }
    }
    colnames(p.mat) <- rownames(p.mat) <- colnames(mat)
    p.mat
}
```




***
## IV. Data Exploration

### (i) Basic Exploration {.tabset}
    
#### Structure
``` {r lsakl, warning = FALSE}
str(data)
data$Customer.id <- as.factor(data$Customer.id)
data$State <- as.factor(data$State)
data$Sales.2018 <- as.numeric(data$Sales.2018)
data$Sales.2017 <- as.numeric(data$Sales.2017)
data$Sales.2017[which(is.na(data$Sales.2017))] <- 0
data$Life.orders <- as.numeric(data$Life.orders)
data$Entry.date <- as.numeric(ymd("2019-01-01")) - as.numeric(ymd(data$Entry.date))
data$Cust.type.id <- as.factor(data$Cust.type.id)
top_10 <- data[1:10,]
```
***


#### Summary
The summary reveals the following:    
- There were 1044 unique customers in 2018.    
- Lifetime orders and sales values are skewed right, indicating that there are relatively few customers making much larger purchases and few customers are repeating business at a much higher rate than average. Note that these are two groups which may or may not be the same individuals.    
- There were $1,568,867 in sales in 2018 (matching the value on the 2018 income statement) and $1,591,916 in sales in 2017, which is $217,633.10 less than reported on 2017 income statement. This discrepancy is because our raw data is a left join of the 2018 and 2017 sales (i.e. all the 2018 sales are represented in the data but only a portion of the 2017 sales are).    
  
```{r lksas}
summary(data)
Sales.2018.total <- sum(data$Sales.2018)
Sales.2018.total
Sales.2017.total <- sum(data$Sales.2017)
Sales.2017.total
top_10 <- data[1:10,]
top_10
```
***


#### Distribution of Sales

The blue and red lines on the density plot are the median and mean of the individual sales. The right skew is due to a handful of customers who made much larger purchases than average. The top 10 customers make up `r 100*(sum(top_10$Sales.2018)/sum(data$Sales.2018))%>%round(4)`% of sales and account for approximately \$320K in revenue.   

```{r kls}
quantile(data$Sales.2018, probs = seq(0.1,1,0.1))

sum(top_10$Sales.2018)

# use log transformation of sales 
par(mfrow = c(1,2))
fit <-density(log(data$Sales.2018))
plot(fit,
     xlab = "Sales (log scale)",
     main = "Kernel Density Estimate of 2018 Sales", xaxt = "n"
     )
abline(v = c(log(median(data$Sales.2018)), log(mean(data$Sales.2018))),
       col = c("red", "blue"),
       lty = 2)
axis(side = 1, at = log(c(10,100,1000, 10000, 100000)), labels = c(10,100,1000, 10000, 100000))


# top 100 customer sales
plot(data$Sales.2018[1:100], 
     type = "h",
     xlab = "Rank",
     ylab = "Sales",
     main = "Sales of Top 100 Customers")
```

***

#### New Customers
- There were 347 new customers (customers who bought this year but not last year).    
- New customer sales are significantly lower than for customers who bought last year. They also have less variability. 
-  The median and mean are indicated by red and blue respectively, with dots denoting statistics for new customers. Interestingly 85 of the "new" customers have an entry date prior to 2017.   

```{r kljjl}
par(mfrow=c(1,2))
boxplot(log(data$Sales.2018)~data$Sales.2017==0)
new_sales <- data[data$Sales.2017==0,]
summary(new_sales)
plot(density(log(new_sales$Sales.2018)), xaxt="n", xlab = "log(Sales)", main = "")
axis(side = 1, at = log(c(10,100,1000, 10000, 100000)), labels = c(10,100,1000, 10000, 100000))
abline(v = c(log(median(new_sales$Sales.2018)), log(mean(new_sales$Sales.2018))),
       col = c("red", "blue"),
       lty = 4,
       lwd=3)
abline(v = c(log(median(data[data$Sales.2017>0,]$Sales.2018)), log(mean(data[data$Sales.2017>0,]$Sales.2018))),
       col = c("red", "blue"),
       lty = 2,
       lwd=3)
```


#### Customer Type Breakdown

The table shows the aggregate breakdown of sales by customer type in 2018.

```{r lksa}
data[,c("Cust.type.id", "Sales.2018")]%>%
  group_by(Cust.type.id)%>%
  summarise_all(c(mean, sum, sd, function(x){sum(x)/mean(x)}))%>%
  arrange(desc(fn2))%>%filter(fn4 >= 10)%>%
  kable(col.names = c("Type", "Average Sales", "Total Sales",  "Std. Dev.", "Count"))
```

***

#### Customer Location

Regarding moving the business: Some of the customers are one day closer in shipping time and some are one day farther away. A majority of the top 10 states are in an equally or more favorable location when shipping from LOCATION X. 

```{r kfdlsa}
data[,c("State", "Sales.2018")]%>%
  group_by(State)%>%
  summarise_all(c(mean, sum, sd, function(x){sum(x)/mean(x)}))%>%
  arrange(desc(fn2))%>%head(10)%>%
  kable(col.names = c("Type", "Average Sales",  "Total Sales", "Std. Dev.", "Count"))
```



***


### (ii) Relationships Between Variables {.tabset}

#### Correlations 1
- Strong correlation between 2017 and 2018 sales (r = 0.8751)  
- Statistically significant correlation between 2017 and 2018 sales (p-val = 0.029)   
- Weak correlation between lifetime orders and 2018 sales (r = 0.3138)  
```{r kkl}
# calculate correlation coefficients
cor_data<-select_if(data, is.numeric)%>%cor()
cor_data

# calculate p-values of correlations
p.mat <- cor.mtest(cor_data)

# plot correlations. X's indicate statistically insignificant correlation
corrplot::corrplot((cor_data), 
                   method="number",
                   type="upper", order="hclust", tl.col="black", tl.srt=45,
                   p.mat = p.mat, sig.level = 0.05
)
```
***


#### Correlations 2
- Minimal correlation between the number of days since the customer's first purchase and 2018 sales  
```{r lkad}
plot(log(Sales.2018)~Entry.date, data=data, xlab = "Days since first purchase", ylab = "log(Sales.2018)")
abline(lm(log(Sales.2018)~Entry.date, data=data))
```

***



#### ANOVA
- No significant difference between the mean sales among the states (p-val = 0.691)   
- There is at least one customer type with a significantly different mean than the others (p-val = 0.00113)   
- Pairwise comparison shows that the true mean of the CNSMR category may be significantly lower than several of the others   

```{r lkfsa}
par(mfrow=c(1,2))
boxplot(log(Sales.2018)~Cust.type.id, data=data[data$Sales.2017 > 0, ])
boxplot(log(Sales.2018)~State, data=data[data$Sales.2017 > 0, ])

aov1<- aov(log(Sales.2018)~State, data=data[data$Sales.2017 > 0, ])
aov2<- aov(log(Sales.2018)~Cust.type.id, data=data[data$Sales.2017 > 0, ])
summary(aov1)
summary(aov2)

SSTr1 <- 81.5
SSE1 <- 996.4
r.sq1 <- 1-SSE1/(SSE1+SSTr1)
T <- TukeyHSD(aov2)
p.vals <- T[[1]][,4]
p.vals <- p.vals[order(p.vals)]
p.vals[1:15]
```


***


## V. Modeling Recurring Sales

### (i) SLR: 2018 Sales and 2017 Sales

Fitted regression line       
log(Sales.2018) = 1.1039 + 0.82077(log(Sales.2017))  
    
- No patterns appear in the residual plot. The residuals appear to be randomly scattered around the 0 line and there is a rough horizontal band around the 0 line. The residuals appear to be approx. normally distributed. A rough line is formed in the QQ plot indicating normally distributed data. No apparent outliers.     
- p-val < 0.05, indicating there is a significant linear relationship between log(Sales.2018) and log(Sales.2017). 
- 63.9% of variation in log(Sales.2018) can be explained by log(Sales.2017) --> the prior year sales model explains 63.98% of the variance in the current year sales.   
   


```{r lsd}
line1 <- lm(log(Sales.2018)~log(Sales.2017), data=data[data$Sales.2017 > 0, ])
summary(line1)

par(mfrow=c(2,2))
plot(log(Sales.2018) ~ log(Sales.2017), data=data[data$Sales.2017 > 0, ], xaxt="n", yaxt="n")
axis(side = 1, at = log(c(10,100,1000, 10000, 100000)), labels = c(10,100,1000, 10000, 100000))
axis(side = 2, at = log(c(10,100,1000, 10000, 100000)), labels = c(10,100,1000, 10000, 100000))
abline(line1$coefficients[1], line1$coefficients[2])
resid1 <- resid(line1)
plot(fitted.values(line1),resid1)
hist(resid1)
qqnorm(resid1)
abline(0,1)
```

***
### (ii) SLR: 2018 Sales and Lifetime Orders

Fitted regression line       
log(Sales.2018) = 5.21663 + 0.51708(log(Life.orders))  
    
- Linearity assumptions hold.       
- p-val < 0.05, indicating there is a significant linear relationship between log(Sales.2018) and log(Life.orders). 
- 19.6% of variation in log(Sales.2018) can be explained by log(Life.orders) --> the lifetime orders model explains 19.6% of the variance in the current year sales.  


```{r jlksa}
line2 <- lm(log(Sales.2018)~log(Life.orders), data=data[data$Sales.2017 > 0, ])
summary(line2)
par(mfrow=c(2,2))
plot(log(Sales.2018)~log(Life.orders), data=data[data$Sales.2017 > 0, ], xaxt="n", yaxt="n")
axis(side = 2, at = log(c(10,100,1000, 10000, 100000)), labels = c(10,100,1000, 10000, 100000))
axis(side = 1, at = log(c(10,100,1000, 10000, 100000)), labels = c(10,100,1000, 10000, 100000))
abline(line2$coefficients[1], line2$coefficients[2])
resid2 <- resid(line2)
plot(fitted.values(line2),resid2)
hist(resid2)
qqnorm(resid2)
abline(0,1)
```
***


### (iii) MLR: 2018 Sales, 2017 Sales, Lifetime Orders
  
Fitted regression line       
log(Sales.2018) = 1.11458 + 0.78584(log(Sales.2017)) + 0.08128(log(Life.orders))  
    
- No interaction in the model.   
- p-val < 0.05, indicating there is a significant linear relationship between log(Sales.2018) and log(Sales.2017), log(Life.orders).  
- Both variables in model significant to predict log(Sales.2018).
- 64.3% of variation in log(Sales.2018) can be explained by log(Sales.2017) and log(Life.orders).
  

```{r kslms}
comb_model <- lm(log(Sales.2018)~log(Sales.2017)+log(Life.orders), data = data[data$Sales.2017>0,])
cs <- summary(comb_model)
print(cs)
par(mfrow=c(2,2))
resid3 <- resid(comb_model)
plot(fitted.values(comb_model),resid3)
hist(resid3)
qqnorm(resid3)
abline(0,1)

```
***

## VI. Model Limitations & Conclusion

These models are severely limited by lack of data. Namely more years of data. Since 2018 sales were lower than 2017 sales, the model expects them to decline further. Ownership has stated that 2017 is an outlier year (with more sales than usual due to a few individuals placing large orders). With the lack of information and this insight, making predictions with these models would be misleading.   
   
**The combined linear model (MLR) explained slightly more of the variance than the best simple model, adjusting for the complexity of the models.**  



