IBM dashDB快速构建电子商务大数据分析
========================================================
author: Eric (lipenglp@cn.ibm.com)
date: 2015/03/13
transition-speed: slow


议程: 使用IBM dashDB快速构建用户流失模型，预测未来用户流失概率
========================================================
- 1. 使用dashDB R library: ibmdbR
- 2. 加载历史订单数据到dashDB
- 3. 数据变换
- 4. 构建分析矩阵
- 5. 变形分析因子
- 6. 构建朴素贝叶斯模型，训练模型
- 7. 测试预测模型，保存模型
- 8. 加载新订单数据
- 9. 使用预测模型， 分析用户流失概率


使用dashDB R library: ibmdbR
========================================================

- 加载dashDB R library
- 建立连接到dashDB
<small>
```{r}
# Load the ibmdbR package 
library(ibmdbR)
#make a connection
con <- idaConnect('BLUDB')
idaInit(con)
```
</small>
加载历史订单数据
========================================================

 - 1. Import the data into the dashDB with dashDB console with your dashDB account.
      * [bluemix](https://console.ng.bluemix.net/)
      * [dashDB console](https://yp-dashdb-small-01-lon02.services.eu-gb.bluemix.net:8443/console/ibmblu/index_Customer.jsp)
      * [RStudio](http://yp-dashdb-small-01-lon02.services.eu-gb.bluemix.net:8787/) 

 - 2. Download the sample data to your desktop 
       * [C_ALL.csv](https://raw.githubusercontent.com/cuericlee/dashDB/master/C_ALL.csv)
       * [C_NEW.csv](https://raw.githubusercontent.com/cuericlee/dashDB/master/C_NEW.csv)
       
 - 3. Manage -> Load Data -> Loda from desktop -> Load files -> Create table called C_ALL

数据变换
========================================================

- 1. Go to dashDB console, Manage -> Run SQL Scripts
    *[dashDB console](https://yp-dashdb-small-01-lon02.services.eu-gb.bluemix.net:8443/console/ibmblu/index_Customer.jsp)

- 2. Run folllowing DDL to create views for further anayalitic.

``` 
DROP VIEW USER_BUY；
create view USER_BUY AS
SELECT USER_ID, COUNT(USER_ID) AS TOTAL_BUY  FROM C_ALL WHERE TYPE=0 GROUP BY USER_ID；
DROP VIEW USER_VISITS；
create view USER_VISITS AS
SELECT USER_ID, COUNT(USER_ID) AS TOTAL_BUY  FROM C_ALL  GROUP BY USER_ID；
DROP VIEW USER_BRAND;
CREATE VIEW USER_BRAND AS
SELECT USER_ID, COUNT(UNIQUE(BRAND_ID)) AS TOTAL_BRAND  FROM C_ALL  GROUP BY USER_ID;
```

构建分析矩阵
========================================================

 - 3. Construct the metric with factor and result columns. (USER_BUY, USER_VISITS, USER_BRAND)
 
```{r}
user_buy<- ida.data.frame('USER_BUY')
user_visits<- ida.data.frame('USER_VISITS')
user_brand<- ida.data.frame('USER_BRAND')
```

变形分析因子
========================================================

 - 4. Construct the correlation metric for model and rbind CHURN, IN_B2B

``` {r}

  # Create a link to the CUSTOMER_CHURN table, which is used for churn prediction.
  # In this sample, we are interested in the correlation of two variables to churn:
  # - Whether the customer is in a business-to-business (b22) industry
  # - Whether the number of products a customer bought was less than three
  # The following transformations make the output easier to read.
  # These transformations are local to R and are not (and do not need to be) written to the database.
 
```

变形分析因子(2)
========================================================
<small>
```{r}
transform<- function (user_buy, user_visits, user_brand) {
  tempDf <- idaMerge(user_buy, user_visits, by="USER_ID")
  churnDf <- idaMerge(tempDf, user_brand, by="USER_ID")
  churnDf$CHURN<- ifelse(churnDf$TOTAL_VISITS > 10 & churnDf$TOTAL_BRAND>3,'nochurn','churn')
  # The IN_B2C_INDUSTRY field is also encoded as or 1. Transform this to a string ('no' or 'yes').
  churnDf$IN_B2C<- ifelse(churnDf$TOTAL_BUY > 3 & churnDf$TOTAL_BRAND>10,'yes','no')
  # Transform the value of the TOTAL_BUY field to a discrete value ('threeormore' or 'lessthanthree').
  churnDf$TOTALBUY<- ifelse(churnDf$TOTAL_BUY>2,'threeormore','lessthanthree')
  churnDf
}
```
</small>

构建模型，训练模型
========================================================

- 5. Train the model CHURN~IN_B2B+TOTALBUY+TOTAL_BRAND

``` {r, echo=TRUE}
churnDf<- transform(user_buy, user_visits, user_brand)
# Display the first few rows of the resulting ida.data.frame.
writeLines("Input IDA data frame (USER_ID):")
head(churnDf)

```


构建朴素贝叶斯模型，训练模型
========================================================
``` {r}
# Build a naive Bayes model based on the transformed data.
nb <- idaNaiveBayes(CHURN~IN_B2C+TOTALBUY+TOTAL_BRAND, churnDf,"USER_ID")

# Print the model
print(nb)
```


测试预测模型，保存模型
========================================================
```{r}

# Use the predict method to make predictions
preds <- predict(nb, churnDf,"USER_ID")

saveRDS(nb, "nb.rds")
```



加载新订单数据
========================================================

 - 6. Load new data into dashDB with console. C_NEW.csv -> C_NEW
 
      * [console](https://yp-dashdb-small-01-lon02.services.eu-gb.bluemix.net:8443/console/ibmblu/index_Customer.jsp)
      * [C_NEW.csv](https://raw.githubusercontent.com/cuericlee/dashDB/master/C_NEW.csv)
 
```
drop view USER_BUY_NEW;
create view USER_BUY_NEW AS
SELECT USER_ID, COUNT(USER_ID) AS TOTAL_BUY  FROM C_NEW WHERE TYPE=0 GROUP BY USER_ID;

drop view USER_VISITS_NEW;
create view USER_VISITS_NEW AS
SELECT USER_ID, COUNT(USER_ID) AS TOTAL_VISITS  FROM C_NEW  GROUP BY USER_ID;
```

变形新订单数据
========================================================

```{r}
# Load the ibmdbR package and make a connection
library(ibmdbR)
con <- idaConnect('BLUDB')
idaInit(con)

# 1. Load new data into dashDB. C_ORDERS.new.csv
# 
# create view USER_BUY_NEW AS
# SELECT USER_ID, COUNT(USER_ID) AS TOTAL_BUY  FROM C_NEW WHERE TYPE=0 GROUP BY USER_ID;
# 
# create view USER_VISITS_NEW AS
# SELECT USER_ID, COUNT(USER_ID) AS TOTAL_VISITS  FROM C_NEW  GROUP BY USER_ID;

user_buy<- ida.data.frame('USER_BUY_NEW')
user_visits<- ida.data.frame('USER_VISITS_NEW')
user_brand<- ida.data.frame('USER_BRAND_NEW')

transform<- function (user_buy, user_visits, user_brand) {
    # Create a link to the CUSTOMER_CHURN table, which is used for churn prediction.
  # In this sample, we are interested in the correlation of two variables to churn:
  # - Whether the customer is in a business-to-business (b2b) industry
  # - Whether the number of products a customer bought was less than three
  tempDf <- idaMerge(user_buy, user_visits, by="USER_ID")
  churnDf <- idaMerge(tempDf, user_brand, by="USER_ID")
  
  # The following transformations make the output easier to read.
  # These transformations are local to R and are not (and do not need to be) written to the database.
  
  churnDf$CHURN<- ifelse(churnDf$TOTAL_VISITS > 10 & churnDf$TOTAL_BRAND>3,'nochurn','churn')
  
  # The IN_B2C_INDUSTRY field is also encoded as or 1. Transform this to a string ('no' or 'yes').
  churnDf$IN_B2C<- ifelse(churnDf$TOTAL_BUY > 3 & churnDf$TOTAL_BRAND>10,'yes','no')
  
  # Transform the value of the TOTAL_BUY field to a discrete value ('threeormore' or 'lessthanthree').
  churnDf$TOTALBUY<- ifelse(churnDf$TOTAL_BUY>2,'threeormore','lessthanthree')
  
  churnDf
}
```

使用预测模型， 分析用户流失概率
========================================================

```{r, echo=TRUE}
churnDf<- transform(user_buy, user_visits, user_brand)
# Re-use modle
#ibmdbR:::
nbCopy<- readRDS("nb.rds")
#print(nbCopy)
preds <- predict(nbCopy, churnDf,"USER_ID")

output<- head(preds, n = 100L)
output
write.csv(output, "preds.csv")
# Close the connection to the database
idaClose(con)
```

