# CRM-Migration
## Part 1
Hey everyone! 👋

So, I was building my portfolio and I hit a snag... how can I showcase my automation projects without leaking the company data I work with? Confidentiality is key, right? 😬

Then it hit me: why not just use a free system? That way, there are no limits! 🚀

Plus, I get to learn and level up my skills on this CRM journey.

Of course! Here is that new section, formatted in the same informal and intuitive English style for your README.

### The System of Choice 🎯
After a bit of research, I made my call: I'm going with SuiteCRM. For the local database, I'm using the classic MySQL and XAMPP duo.

With that decided, it was time to get the environment ready. Here's what I did:

- **Database Setup:** I jumped into phpMyAdmin, created a new database specifically for SuiteCRM, and set up a login user for it.

- **CRM Configuration:** Got the SuiteCRM instance installed and configured to connect with the new database.

- **Toolkit Prep:** Fired up MySQL Workbench to have a solid GUI for the database. I also set up a Python profile and the Database Client (JDBC) extension in VS Code to interact with everything right from my editor.

Environments are prepped and ready to go! 🚀

### Getting Hands-On: The Data! 💻
System's all set up, so what's next? DATA! Time to grab some information to get started.

Our good old partner Kaggle is always here to help out.

The mission is to migrate an [Amazon Products Dataset (2023 version)](https://www.kaggle.com/datasets/asaniczka/amazon-products-dataset-2023-1-4m-products/data) into my system! Let's do this! 🔥

agora precisamos enteder as tabelas e o sistema em si, então como vamos trabalhar com a tablea produtos e categorias, tenho que verificar como o sistema enxerga estas tabelas para prepara-las, então o primeiro passo é fazer um cadastro teste e verificar como o sistema compoe esta tabela. vou fazer primeiramente para a categoria

![CRM-CADASTRO](./img/register-categories-print.png)

### Diving Into the Data Structure 🕵️‍♀️

Alright, systems are prepped, data's lined up... but before we just start shoveling info in, we need to understand how SuiteCRM likes its data! Think of it like this: if you want to teach a new dog old tricks, you first need to learn how they learn, right? 😉

Our main focus right now is on products and categories. To make sure our data plays nice with SuiteCRM, the first step is to see how the system handles these tables internally.

The easiest way? A little test run! I'll create a new category directly in SuiteCRM and then peek under the hood to see how it structures the data. This will give us the perfect blueprint for our imports.

Here’s a quick peek at the test category creation:

Now, to see how that looks in the database, a quick query in MySQL Workbench does the trick:

```sql
SELECT * FROM aos_product_categories;
```

### Python Powering Up: Data Transformation! 🐍
Now that we know exactly how SuiteCRM wants its category data, it's Python's time to shine! We'll grab our raw data and whip it into shape to match SuiteCRM's table structure.

First, let's load our Amazon categories dataset:

```python
import pandas as pd
df_csv =  pd.read_csv("ARQUIVES/amazon_categories.csv")
df_csv.columns
```
Right off the bat, I can see a few things we need to tweak. For starters,  needs to become , and we'll need to generate new IDs that fit SuiteCRM's format.category_namename
To get the exact structure, I'm also pulling the existing category table directly from MySQL for a side-by-side comparison. This ensures we don't miss any crucial columns or data types.


```python
import mysql.connector
cns = mysql.connector.connect(
    host="localhost",
    user="PYTHON",
    password="python",
    database="suitecrm_db",port=3307
)
```

Here's the lowdown on the columns we need to match, straight from my Workbench peek:

**Columns:**
id char(36) PK 
name varchar(255) 
date_entered datetime 
date_modified datetime 
modified_user_id char(36) 
created_by char(36) 
description text 
deleted tinyint(1) 
assigned_user_id char(36) 
is_parent tinyint(1) 
parent_category_id char(36)


Time to transform our DataFrame! We're dropping the old , renaming , and populating all those essential SuiteCRM fields with the correct values and user IDs (the one you see is my default admin user ID). We're also generating fresh UUIDs for each entry!

```python
import uuid
from datetime import datetime
import numpy as np
import pandas as pd

df_csv.drop(columns=['id'],inplace=True)
df_csv.rename(columns={'category_name':'name'},inplace=True)

df_csv['date_entered'] = datetime.now()
df_csv['date_modified'] = datetime.now()
df_csv['modified_user_id'] = '72eca1f5-2903-0fc1-8b34-68739d02d654'
df_csv['created_by'] ='72eca1f5-2903-0fc1-8b34-68739d02d654'
df_csv['description'] = ''
df_csv['deleted'] = 0
df_csv['assigned_user_id'] = '72eca1f5-2903-0fc1-8b34-68739d02d654'
df_csv['is_parent'] = 0
df_csv['parent_category_id'] = np.nan
df_csv['id'] = df_csv.apply(lambda x: str(uuid.uuid4()), axis=1)

df_final = df_csv[['id', 'name', 'date_entered', 'date_modified', 'modified_user_id',
       'created_by', 'description', 'deleted', 'assigned_user_id', 'is_parent',
       'parent_category_id']]
```

With our DataFrame all spruced up and perfectly aligned with SuiteCRM's table structure, it's go time! We successfully imported our first table, bringing in a **whopping 248 category entries!**


```python
from sqlalchemy import create_engine
engine = create_engine(
    "mysql+pymysql://PYTHON:python@localhost:3307/suitecrm_db",
    echo=False,      
    pool_pre_ping=True
)
df_final.to_sql('aos_product_categories', engine, if_exists='append', index=False)
```

## Part 2: Wrangling the Amazon Products ⚙️
Alright, time for the second file: `amazon_products.csv.` The plan is similar to the first import—load, transform, and save.

![CRM-CADASTRO-P](./img/register-products-print.png)

But... we have a problem. Remember how SuiteCRM created its own unique IDs for the product categories? Well, this products file still references the old category IDs from the original CSV. We need to bridge that gap. Let's get to it.

### Loading the Data
First, let's get our data loaded: the new products CSV and the existing products table from the SuiteCRM database (so we know what schema to match).

```python
import pandas as pd
import mysql.connector

df_csv = pd.read_csv("ARQUIVES/amazon_products.csv")

cns = mysql.connector.connect(
    host="localhost",
    user="PYTHON",
    password="python",
    database="suitecrm_db",
    port=3307
)
Query = "SELECT * FROM aos_products"
df_mysql = pd.read_sql(Query, cns)
```

### Initial Cleanup 🧹
A little housekeeping first. We'll rename the columns we want to keep to match the CRM's names (`asin` -> `part_number`, etc.) and drop the ones we don't need.

```python

df_csv.rename(columns={'asin':'part_number', 'title':'name', 'productURL':'url'}, inplace=True)

df_csv.drop(columns=['imgUrl', 'stars', 'reviews', 'listPrice', 'isBestSeller', 'boughtInLastMonth'], inplace=True)
```
### The ID Crosswalk: The Big Fix 🗺️
Here’s the main challenge. We can't link products to categories because the IDs don't match.

To solve this, we'll create a temporary "crosswalk" table that maps the old `category_id` from the CSVs to the new `id` generated by SuiteCRM.

```python
df_categories_csv = pd.read_csv("ARQUIVES/amazon_categories.csv")

df_categories_mysql = pd.read_sql_table('aos_product_categories', engine)

dfr = df_categories_mysql[['id', 'name']]

df_merge = df_categories_csv.merge(dfr, left_on='category_name', right_on='name', how='left')

df_merge.drop(columns=['name', 'category_name'], inplace=True)
```
Applying the New IDs ✨
Now we can use our `df_merge` crosswalk to bring the correct SuiteCRM category ID into our main products dataframe.


```python
df_csv = df_csv.merge(df_merge, left_on='category_id', right_on='id_x', how='left')

df_csv.drop(columns=['id_x', 'category_id'], inplace=True)

df_csv.rename(columns={'id_y':'aos_product_category_id'}, inplace=True)
```
### Final Touches & Schema Matching ✅
With the main relationship fixed, we just need to fill in the rest of the required metadata fields to make SuiteCRM happy. This includes timestamps, user IDs, setting default values, and generating a unique `id` (UUID) for each new product.

```python
df_csv['date_entered'] = datetime.now()
df_csv['date_modified'] = datetime.now()
df_csv['modified_user_id'] = '72eca1f5-2903-0fc1-8b34-68739d02d654'
df_csv['created_by'] = '72eca1f5-2903-0fc1-8b34-68739d02d654'
df_csv['description'] = ''
df_csv['deleted'] = 0
df_csv['type'] = 'Good'
df_csv['currency_id'] = 99 # Corresponds to a value in the currencies table

df_csv['cost'] = df_csv['price']
df_csv['cost_usdollar'] = df_csv['price']
df_csv['price_usdollar'] = df_csv['price']

df_csv['id'] = df_csv.apply(lambda x: str(uuid.uuid4()), axis=1)

df_csv['assigned_user_id'] = np.nan
df_csv['contact_id'] = ''
df_csv['product_image'] = np.nan
df_csv['category'] = np.nan
```
##The Grand Finale: Import! 🎉
Last step! We reorder our dataframe columns to the exact sequence of the database table and run the import.
```python

final_columns = ['id', 'name', 'date_entered', 'date_modified', 'modified_user_id',
       'created_by', 'description', 'deleted', 'assigned_user_id', 'maincode',
       'part_number', 'category', 'type', 'cost', 'cost_usdollar',
       'currency_id', 'price', 'price_usdollar', 'url', 'contact_id',
       'product_image', 'aos_product_category_id']

df_final = df_csv[final_columns]

df_final.to_sql('aos_products', engine, if_exists='replace', index=False)
```

After a few minutes, we have over a million product lines loaded into our CRM!
