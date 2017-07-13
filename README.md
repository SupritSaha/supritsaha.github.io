### Speeding up with data.table
#### *Suprit Saha*
#### <supritster@gmail.com>

A common criticism about R especially to users having prior programming experience
in other languages is that it is **slow**. Although their concerns are valid, one must
understand that speed was never R's forte. Ease of data analysis, availability of
plethora of statistical packages and great visualization makes R a preferred choice
for statisticians. The growing popularity of machine learning has led to Python
being considered as an alternative to R.Python packages such as **Numpy** and **Pandas**
are undeniably faster due to better memory management and are simple to learn.
One can clearly understand the differences in computing speed when working with
large datasets.

In 2006, Matt Dowie released a R package called **data.table**, which today is one
of the fastest open-source in-memory data manipulation packages. Providing a description
of the package, he said:

***This package does very little. Just like a data frame but without row names. Upto
to 10 times faster. Up to 10 times less memory.***


Alas, this package has turned out to be the goto package of many R users for performing
efficient data manipulation tasks.
This tutorial covers most of the functions used in the data.table package and compares
it with regular R data frames.

### **Overview**

As Matt Dowie mentioned in his package description, we can think of data.table as
extended data frames with two notable features :-

* Speed
* Cleaner and precise syntax

The aim of data.table is to reduce programming time by using fewer function calls,
less variable name repetition and reduce computing time through faster aggregation
and updating by reference.

<br>

#### **Creating a data table**

```{r}
library(data.table)
DT <- data.table(A = c(2,3,6,8), B = c("even","odd","even","even"), C = c(TRUE,FALSE))
DT
```

This is the similar to creating data frames.

```{r}
DF <-  data.frame(A = c(2,3,6,8), B = c("even","odd","even","even"), C = c(TRUE,FALSE))
DF
```

**Note** : Recycling of vectors also happens for data table. In fact:
```{r}
class(DT)
```

<br>

#### **Converting data frame to data table**

```{r}
library(data.table)
DF <- as.data.table(DF)
```

This method is time and memory consuming when working with large lists or data frames because it makes a complete copy of the input object before converting into a data table.

A better way is to modify the input by reference as follows:
```{r}
setDT(DF) # No need to assign 
```

<br>

#### **Reading data **

*fread()* function to efficiently read large files. It can automatically detect separator and column type.
```{r}
titanic_dt <- fread("train _titanic.csv",na.strings = "")
titanic_df <- read.csv("train _titanic.csv",header = TRUE,stringsAsFactors = FALSE,na.strings = "")
```

**Why is fread command faster than other import commands like read.table ?**

Both fread and read.table functions are written in C but **fread** memory maps the file into memory and then iterates through the file using pointers  whereas **read.csv** reads the file into a buffer via a connection.

One can get an in-depth understanding about fread [here](https://github.com/Rdatatable/data.table/wiki/Convenience-features-of-fread) 

<br>

#### **Writing to CSV**

One of the fastest ways to write to a CSV file is using **fwrite**. It parallelizes the data writing process.
```{r}
fwrite(titanic_dt,file = "titanic.csv") # row.names are set to FALSE by default
```

<br>

#### **The data.table syntax**

The general syntax for data.table resembles to standard SQL query : 

                              
                              DT[i, j, by]
                              
where 

1. DT is the data.table
2. i is equivalent to WHERE in SQL which allow row subsetting
3. j is equivalent to SELECT in SQL which allow manipulating columns 
4. by is equivalent to GROUP BY in SQL 

Ex : 
```{r}
titanic_dt[Survived == 1,                 # subsetting rows of survived passengers
          .(avg_age = mean(Age,na.rm = TRUE),     # calculating average age and median fare
            median_fare = median(Fare,na.rm = TRUE)),
          by = Sex]               # grouping by gender
```

<br>

#### **Selecting rows**

```{r,results="hide"}
titanic_dt[6,] # same as data.frame
titanic_dt[6] 
```

Both the above commands select the 6th row of the data.table. However, the second command does not give the desired output for data frames.

```{r}
titanic_df[6][1:5,] # here 6th column of data frame is selected
```

<br>

#### **Selecting columns**

There are different ways to select one or many columns from a data.table. User should preferably stick to one of the approaches for consistency.
```{r,results='hide'}
# Single column selection
titanic_dt[,2] # returns a 1 column data table.This feature was not there earlier and would return 2. Please ensure you have the latest version of data.table
titanic_dt[[2]] # returns a vector
titanic_dt[,"Survived"] # returns a 1 column data table
titanic_dt[,Survived] # returns a vector
titanic_dt$Survived # returns a vector

# Multiple column selection
titanic_dt[,2:3] # same as data.frame. 
titanic_dt[,2:3,with = FALSE] 
titanic_dt[,list(Survived,Pclass)]
titanic_dt[,.(Survived,Pclass)] # .() is an alias to list()
titanic_dt[,c("Survived","Pclass")]
titanic_dt[,c("Survived","Pclass"),with = FALSE]  
```

<br>

#### **Adding/Removing columns**

One can add or remove columns by reference. Explicit assignment statement not required.
```{r,results='hide'}
titanic_dt[,male_less_than_25 := ifelse(Age < 25 & Sex == 'male',1,0)] # Adding one column
titanic_dt[,c("gender_short","surname") := list(substr(x = Sex,start = 1,stop = 1),substr(Name,start = 1,as.vector(regexpr(text = Name,pattern = ","))-1))]  # Adding multiple columns


titanic_dt[,male_less_than_25 := NULL] # Deleting one column
drop_cols <- c("gender_short","surname")
titanic_dt[,(drop_cols) := NULL] # Deleting multiple columns
#titanic_dt[,4:5 := NULL] # Drop columns by specifying column indices
```

Removing rows by reference is not currently present in data.table. One can use **subset** or **negative indices** as in data.frame to remove rows.

<br>

#### **Conditional subsetting**

Subsetting rows based on multiple filters on multiple columns can be done by :
```{r,results='hide'}
titanic_dt[Sex =='male' & Pclass %in% c(1,3)] 
subset(titanic_dt,Sex =='male' & Pclass %in% c(1,3)) # Both are equivalent
```

The above subsetting is a **vector scan approach** which means that we scan through all the rows in a table trying to identify if the filter condition is satisfied or not. However, this is quite inefficient especially for larger datasets. Complexity : *O(n)*

Data.table has **set** function which enables super fast filtering. We set keys which reorder the dataset by key columns. Since the data is sorted, we don’t have to scan through the entire length of the column. Here data.table uses **binary search** which perform the filtering operations in both speed and memory efficient manner. Complexity : *O(log n)*

```{r,results='hide'}
setkey(titanic_dt,Sex,Pclass)
titanic_dt[.("male",c(1,3))] # Much faster method
```

Let us look for a value say 5 for Pclass which is not valid for this dataset.
```{r}
setkey(titanic_dt,Sex,Pclass) # Reorders the table by the key columns by reference
titanic_dt[.("male",5)] # Returns NA for all other columns
titanic_dt[.("male",5),nomatch = 0] # If we want nothing to be returned
```
<br>

#### **Sorting data**

Sorting can be performed using the following approaches:
```{r,results='hide',echo=FALSE}
titanic_dt <- fread("train _titanic.csv",na.strings = "")
```

```{r,results='hide'}
titanic_dt[order(Survived,-Pclass,Age,na.last = FALSE)] # Sorting Survived,Age in increasing order and Pclass in decreasing order. This does not change the original data.table unless assigned
```

```{r}
setorder(titanic_dt,Survived,-Pclass,Age) # Updating by reference which changes the ordering of original table
head(titanic_dt) 
```

If the ordered columns contain missing values, the default settings of the two methods described would give different results since **order** keeps NA values at the top whereas **setorder** arranges them at the bottom. Both the methods are orders of magnitude faster than **base order**.

<br>

#### **Renaming columns**
```{r}
setnames(titanic_dt,old = c("Pclass","Sex"),new = c("Class","Gender"))
names(titanic_dt)
```

<br>

#### **Replace missing values**

```{r,results='hide'}
titanic_dt[is.na(Age),Age := median(titanic_dt$Age,na.rm = TRUE)] # replacing NA values with median. Note the usage of titanic_dt$Age instead of Age. Here the dataset gets updated
```


<br>

### **Aggregations**

We have discussed about **i** and **j** from data.table's general syntax. Here, we’ll see how they can be combined together with **by** argument to provide group level summaries.

```{r,results='hide',echo=FALSE}
titanic_dt <- fread("train _titanic.csv",na.strings = "")
```

```{r}
titanic_dt[,.(count = .N, 
              avg_age = mean(Age,na.rm = TRUE)), 
              by = .(Sex,Survived)] # .N is a special variable that holds the number of rows for each group
```
One can also group by expressions :

```{r}
titanic_dt[,.(count = .N, 
              percentage_survived = mean(Survived,na.rm = TRUE)* 100),
              by = ifelse(Age < 25 ,"Young",ifelse(Age < 60, "Middle Aged","Old"))] # Expression used in by. Note NA values are also included in group as well as the group variable is not aptly named


titanic_dt[!is.na(Age),
           .(count = .N, 
             percentage_survived = mean(Survived,na.rm = TRUE)* 100),
             by = list(age_group = ifelse(Age < 25 ,"Young",ifelse(Age < 60, "Middle Aged","Old")))] # To remove the NA group, we initially filter the rows in the i argument

```

The **by** argument retains the original ordering of the groups.

<br>

#### **keyby**

There may be occasions when we require the group variables to be sorted. **keyby** is used for such purposes.
```{r}
titanic_dt[,.(count = .N, 
              percentage_survived = mean(Survived,na.rm = TRUE)* 100),
              keyby = .(Pclass,Sex)] 
```

<br>


#### **Summarizing multiple columns using .SD**

Let us extract the last 2 rows of each column grouped by Survived :
```{r}
titanic_dt[,.(PassengerId = tail(PassengerId,2),
              Pclass = tail(Pclass,2),
              Name = tail(Name,2),
              Sex = tail(Sex,2),
              Age = tail(Age,2),
              SibSp = tail(SibSp,2),
              Parch = tail(Parch,2),
              Ticket = tail(Ticket,2),
              Fare = tail(Fare,2),
              Cabin = tail(Cabin,2),
              Embarked = tail(Embarked,2)),
              by = Survived]     
```
The above command works but obviously not feasible to write if we have large number of columns.

data.table provides a special symbol, called **.SD**. It stands for **S**ubset of **D**ata. It by itself is a data.table that holds the data for the current group defined using **by**.
```{r}
titanic_dt[,lapply(.SD,function(x) tail(x,2)),
            by = Survived] 
titanic_dt[,tail(.SD,2),
            by = Survived] # since tail(.SD,2) returns the last 2 rows of a data.table which is also a list,therefore not necessary to wrap it around lapply
```

If we want to summarize over specific columns, use **.SDcols**
```{r}
titanic_dt[,lapply(.SD,mean,na.rm = TRUE),by = .(by1 = Survived,by2 = Pclass %in% c(1,3)),.SDcols = c("Age","Fare")]
```

<br>

#### **Chaining**

A series of operations can be evaluated using data.table's chaining syntax. It is analogous to subqueries in SQL and pipeline in **dplyr**.
```{r}
titanic_dt[,lapply(.SD,mean,na.rm = TRUE),by = .(by1 = Survived,by2 = Pclass %in% c(1,3)),.SDcols = c("Age","Fare")][order(by1,by2)][which(Age == max(Age))] # summarising,sorting,selection
```

<br>

### **Joins**

<br>

#### **Inner Join**

```{r,results='hide'}
passenger_resident <- data.table(PassengerId = c(4,54,43,875,453,984),Country = c('UK','India','US','France','Germany','France'))

# Method 1
setkey(titanic_dt,PassengerId)
setkey(passenger_resident,PassengerId)
titanic_dt[passenger_resident,nomatch = 0]

# Method 2
merge(titanic_dt,passenger_resident,by = 'PassengerId')
```

<br>

#### **Left Outer Join**

```{r,results='hide'}
merge(titanic_dt,passenger_resident,by = 'PassengerId',all.x = TRUE)
```

<br>

#### **Right Outer Join**
```{r,results='hide'}
# Method 1
setkey(titanic_dt,PassengerId)
setkey(passenger_resident,PassengerId)
titanic_dt[passenger_resident]

# Method 2
merge(titanic_dt,passenger_resident,by = 'PassengerId',all.y = TRUE)
```

<br>

#### **Full Outer Join**
```{r,results='hide'}
merge(titanic_dt,passenger_resident,by = 'PassengerId',all = TRUE)
```

<br>

**What is the difference between X[Y] and merge(X,Y)?**

* X[Y] is a join, looking up X’s rows using Y (or Y’s key if it has one) as an index.
* Y[X] is a join, looking up Y’s rows using X (or X’s key if it has one) as an index.
* merge(X,Y)  does both ways at the same time. The number of rows of X[Y] and Y[X] usually
  differ; whereas the number of rows returned by merge(X,Y) and merge(Y,X) is the same.
 
The  usefulness of X[Y] lies in the following example:
```{r}
passenger_resident[titanic_dt,sum(Survived == 1 & Sex == 'male')] # Here data.table automatically inspects the j expression to see which columns it uses. It will only subset those columns only; others are ignored.
```

<br>

### **Other useful functions**

* Reshaping  data : melt(wide to long),dcast(long to wide)
* Time series data : rolling joins
* Binding rows : rbindlist 
* Reordering columns : setcolorder
* Create lag/lead variables : shift

### **End Notes**

The data.table package is one of the most widely used R packages and should be an essential tool for any R user. Although the syntax of data.tables is slightly different from data.frames, it is worthwhile to invest time learning it for efficient R programming. 

### **References**

1. [Official R data.table documentation](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-intro.html)
2. [Github data.table wiki](https://github.com/Rdatatable/data.table/wiki)
3. [Data Camp Course on data.table](https://www.datacamp.com/courses/data-table-data-manipulation-r-tutorial)
4. [Stack Overflow](https://stackoverflow.com/questions/tagged/data.table)

