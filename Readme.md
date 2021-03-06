
SQLite (Query from database file)
=================================

In this tutorial you will be constructing a `coffee.db` containing tables for `coffees`, `salespeople`, `customers`, `orders`, and `order_items`. Once constructed, you will query this database using R. This code is part of my project for my "Data Technology" course with Prof. Harner.

#### 1 Create the `coffee.db` from the `coffee.sql` file. List the tables.

    #Put the SQLite command line code and output here.

    bash-4.1$ sqlite3
    sqlite> .read coffee.sql

    #Here is the list of tables                                                                
    sqlite> .tables
    coffees      customers    order_items  orders       salespeople

    sqlite> .backup coffee.db

#### 2 Display and explain the schema.

    sqlite> .schema
    CREATE TABLE coffees (
      id INTEGER PRIMARY KEY,
      coffee_name TEXT NOT NULL,
      price REAL NOT NULL
    );
    CREATE TABLE customers (
      id INTEGER PRIMARY KEY,
      company_name TEXT NOT NULL,
      street_address TEXT NOT NULL,
      city TEXT NOT NULL,
      state TEXT NOT NULL,
      zip TEXT NOT NULL
    );
    CREATE TABLE order_items (
      id INTEGER PRIMARY KEY,
      order_id INTEGER,
      coffee_id INTEGER,
      coffee_quantity INTEGER,
      FOREIGN KEY(order_id) REFERENCES orders(id),
      FOREIGN KEY(coffee_id) REFERENCES coffees(id)
    );
    CREATE TABLE orders (
      id INTEGER PRIMARY KEY,
      customer_id INTEGER,
      salesperson_id INTEGER,
      FOREIGN KEY(customer_id) REFERENCES customers(id),
      FOREIGN KEY(salesperson_id) REFERENCES salespeople(id)
    );
    CREATE TABLE salespeople (
      id INTEGER PRIMARY KEY,
      first_name TEXT NOT NULL,
      last_name TEXT NOT NULL,
      commission_rate REAL NOT NULL
    );

*In this schema we have 4 tables:*

*+coffees: which must have a coffe\_name and price.*

*+customers: which must have company\_name, street address, city name, state and zip code all in tex format.*

*+order\_item: which has the order.id and coffee.id and coffee.quantity in Integers and two foreign in the following are defined for the orders table and coffee tables.*

*+orders: which have the customer.id and salesperson.id in integer and two more foreing keys for the customers table and salespeople table*

*+salespeople: which must have the first.name and last.name in text format and the commission.rate in real format.*

*It seems that everything that is needed for a coffee house to track the orders of its customers are included in these 4 tables. More details are provided in the next session.*

### 3 Discuss the extent to which the normal forms are satisfied.

*"First normal form requires that the columns in a table must be atomic, there should be no duplicative columns, and every table must have a primary key." By looking at the codes we can see that all the conditions are hold.*

*"Second normal form requires that a table must be in first normal form and all columns in the table must relate to the entire primary key. This rule formalizes the idea that there should be a table for each entity in the data set." The first normal does hold as we saw in the previous part. Also the second condition for this normal is also hold because the other columns of each table relates to the primary key. For example the columns of the key in the coffee table which are coffe name and its price are related to its primary key or the the detailed addresses in the customers table are all related to their primary key too. So I would say the second norm is also hold*

*"Third normal form requires that a table must be in second normal form and all columns in the table must relate **only** to the primary key (not to each other). This rule further emphasizes the idea that there should be a separate table for each entity in the data set." Again as I said in the previous part all the columns looks to me to be related to the primary key directly so I would say it hols the 3rd norm too.*

#### 4 What are the total sales and total commissions for each salesperson? The rows of the resulting table should start with the `last_name` of the salespeople.

``` r
library("RSQLite")

con <- dbConnect(dbDriver("SQLite"), dbname="coffee.db")

measure.df <- dbGetQuery(con, "SELECT sal.last_name, SUM((sal.commission_rate/100)*o1.coffee_quantity*cof.price) Commission ,SUM(o1.coffee_quantity*cof.price) PriceWithoutCommission,
SUM((1+sal.commission_rate/100)*o1.coffee_quantity*cof.price) Total
                         FROM order_items o1
                         INNER JOIN orders o2
                         on o1.order_id = o2.id
                         INNER JOIN customers cus
                         on cus.id=o2.customer_id
                         INNER JOIN coffees cof
                         on cof.id=o1.coffee_id
                         INNER JOIN salespeople sal
                         on sal.id=o2.salesperson_id
                         GROUP BY sal.id
                         ")

measure.df
```

    ##   last_name Commission PriceWithoutCommission   Total
    ## 1 Flinstone     11.187                 111.87 123.057
    ## 2    Rubble     13.984                 139.84 153.824

#### 5 What are the total amounts bought by each customer? The rows of the table should start with the `company_name`.

``` r
# Put the R code here.
measure.df <- dbGetQuery(con, "SELECT cus.company_name, SUM(o1.coffee_quantity*cof.price) TotalPaidByCompany 
                         FROM order_items o1
                         INNER JOIN orders o2
                         on o1.order_id = o2.id
                         INNER JOIN customers cus
                         on cus.id=o2.customer_id
                         INNER JOIN coffees cof
                         on cof.id=o1.coffee_id
                         INNER JOIN salespeople sal
                         on sal.id=o2.salesperson_id
                         GROUP BY cus.id
                         ")

measure.df
```

    ##   company_name TotalPaidByCompany
    ## 1   ACME, INC.             111.87
    ## 2       FOOBAR             139.84

#### 6 What are the sales numbers and the dollar amounts of each type of coffee sold? The rows of the table should start with the `coffee_name`.

``` r
measure.df <- dbGetQuery(con, "SELECT cof.coffee_name, SUM(o1.coffee_quantity) Quantity, cof.price,  SUM(o1.coffee_quantity*cof.price) TotalBoughtofThisCoffee
                         FROM order_items o1
                         INNER JOIN orders o2
                         on o1.order_id = o2.id
                         INNER JOIN customers cus
                         on cus.id=o2.customer_id
                         INNER JOIN coffees cof
                         on cof.id=o1.coffee_id
                         INNER JOIN salespeople sal
                         on sal.id=o2.salesperson_id
                         GROUP BY cof.id
                         ")

measure.df
```

    ##    coffee_name Quantity price TotalBoughtofThisCoffee
    ## 1    Colombian       15  7.99                  119.85
    ## 2 French_Roast        8  8.99                   71.92
    ## 3     Espresso        6  9.99                   59.94

#### 7 create an R data frame for each order item (in this case 4 rows). The columns of the table should be: `company_name`, `last_name`, `commission_rate`, `coffee_name`, `coffee_quantity`, and `price`. More descriptive names of the columns can be used.

``` r
measure.df <- dbGetQuery(con, "SELECT cus.company_name , sal.last_name , sal.commission_rate, cof.coffee_name,o1.coffee_quantity, cof.price
                         FROM order_items o1
                         INNER JOIN orders o2
                         on o1.order_id = o2.id
                         INNER JOIN customers cus
                         on cus.id=o2.customer_id
                         INNER JOIN coffees cof
                         on cof.id=o1.coffee_id
                         INNER JOIN salespeople sal
                         on sal.id=o2.salesperson_id
                         ")
measure.df
```

    ##   company_name last_name commission_rate  coffee_name coffee_quantity
    ## 1   ACME, INC. Flinstone              10    Colombian               5
    ## 2   ACME, INC. Flinstone              10 French_Roast               8
    ## 3       FOOBAR    Rubble              10     Espresso               6
    ## 4       FOOBAR    Rubble              10    Colombian              10
    ##   price
    ## 1  7.99
    ## 2  8.99
    ## 3  9.99
    ## 4  7.99

#### 8 Redo questions 4---6 in R based on the data in the data frame created in question 7.

``` r
attach(measure.df)
measure.df.copy <- measure.df

#Part4
    measure.df.copy$Sales <- coffee_quantity*price 
    measure.df.copy$Commission <- commission_rate*.01*coffee_quantity*price
    measure.df.copy$total <- measure.df.copy$Sales + measure.df.copy$Commission
    
    # aggregate(c(measure.df.copy$Sales), by= list(measure.df.copy$last_name),sum )
    # aggregate(c(measure.df.copy$Commission), by= list(measure.df.copy$last_name),sum )
    # aggregate(c(measure.df.copy$total), by= list(measure.df.copy$last_name),sum )
    # aggregate(c(measure.df.copy$coffee_quantity), by= list(measure.df.copy$last_name),sum )
    
    #Number of Coffees
    numberOfCoffees <- tapply(measure.df.copy$coffee_quantity, last_name, sum)
    #Price of coffees
    priceOfcoffees <- tapply(measure.df.copy$Sales, last_name, sum)
    #Commissions
    Comissions <- tapply(measure.df.copy$Commission, last_name, sum)
    #Total amount
    TotalPrice <- tapply(measure.df.copy$total, last_name, sum)
    cbind(numberOfCoffees, priceOfcoffees,Comissions,TotalPrice)
```

    ##           numberOfCoffees priceOfcoffees Comissions TotalPrice
    ## Flinstone              13         111.87     11.187    123.057
    ## Rubble                 16         139.84     13.984    153.824

``` r
#Part5
  #This part is similar to the previous one, we only need to sum the prices for each customer instead of each salesperson. I also assume that the commission should be paid by the owner of the coffee shops not by the customers.
  PaymentByCompany <- tapply(measure.df.copy$Sales, company_name, sum) #without commision rate
  as.data.frame(PaymentByCompany)
```

    ##            PaymentByCompany
    ## ACME, INC.           111.87
    ## FOOBAR               139.84

``` r
#part6
  cof.quntity <- tapply(measure.df.copy$coffee_quantity, coffee_name, sum)
  cof.Price <- tapply(measure.df.copy$Sales, coffee_name, sum) #Excluding commission rate
  cbind(cof.quntity,cof.Price)
```

    ##              cof.quntity cof.Price
    ## Colombian             15    119.85
    ## Espresso               6     59.94
    ## French_Roast           8     71.92
