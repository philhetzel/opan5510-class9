# Data Joins in Polars: Connecting Related Datasets

## Setup Instructions

### Prerequisites
- Python 3.12 or higher
- Visual Studio Code
- uv (Python package manager) - Install from [astral.sh/uv](https://astral.sh/uv)

### Step 1: Clone the Repository
**Option A: Using GitHub Desktop (Recommended)**
1. Open GitHub Desktop
2. Click "Clone a repository from the Internet"
3. Enter the repository URL
4. Choose your local folder and click "Clone"

**Option B: Using Command Line**
```bash
git clone <repository-url>
cd opan5510-class9
```

### Step 2: Open in VS Code
Navigate to the project folder and open it in VS Code:
- In GitHub Desktop: Click "Open in Visual Studio Code"
- Or open VS Code and use File � Open Folder

### Step 3: Create Virtual Environment and Install Dependencies
Open a terminal in VS Code (Terminal � New Terminal) and run:
```bash
# Create virtual environment and install dependencies in one step
uv sync

# This creates a .venv folder and installs all packages from pyproject.toml
```

### Step 4: Install VS Code Extensions
Open VS Code and install these extensions:
1. **Jupyter** - For running notebook files
   - Extension ID: `ms-toolsai.jupyter`
2. **Data Wrangler** - For viewing dataframes interactively
   - Extension ID: `ms-toolsai.datawrangler`

### Step 5: Verify Installation
1. Open `class_example.ipynb` in VS Code
2. When prompted to select a kernel, choose the Python interpreter from `.venv` (should show `.venv` in the path)
3. Run the first cell to verify everything works

## Introduction

This class focuses on one of the most critical data operations: **joining datasets**. Just like in real business scenarios where information is spread across multiple systems and tables, you'll learn how to properly connect related data.

## 1. Understanding Database Relationships

### What are Joins?

Joins are operations that combine rows from two or more tables based on a related column between them. Think of it like connecting puzzle pieces; each piece has a specific way it fits with others.

### Key Concepts: Primary and Foreign Keys

**Primary Key**: A column (or combination of columns) that uniquely identifies each row in a table.
- Must be unique - no duplicates allowed
- Cannot be NULL
- Examples: customer_id, order_id, product_sku

**Foreign Key**: A column that creates a link between two tables by referencing the primary key of another table.
- Can have duplicates
- Can be NULL (if the relationship is optional)
- Examples: customer_id in an orders table (references customers table)

### Visual Example

```
Customers Table (Primary Key: customer_id)
| customer_id | name    | email              |
|-------------|---------|-------------------|
| 1           | Alice   | alice@email.com   |
| 2           | Bob     | bob@email.com     |
| 3           | Charlie | charlie@email.com |

Orders Table (Foreign Key: customer_id)
| order_id | customer_id | product    | amount |
|----------|-------------|------------|--------|
| 101      | 1           | Laptop     | 1200   |
| 102      | 2           | Mouse      | 25     |
| 103      | 1           | Keyboard   | 80     |
| 104      | 3           | Monitor    | 300    |
```

In this example:
- `customer_id` in the Customers table is the **primary key**
- `customer_id` in the Orders table is the **foreign key**
- The relationship allows us to connect order information with customer information

## 2. Types of Joins in Polars

### Inner Join
Returns only rows that have matching values in both tables.

```python
# Syntax
result = df1.join(df2, on="key_column", how="inner")
```

**Use when**: You only want records that exist in both datasets.

### Left Join
Returns all rows from the left table, and matched rows from the right table. NULL values for non-matching rows.

```python
# Syntax
result = df1.join(df2, on="key_column", how="left")
```

**Use when**: You want to keep all records from your main dataset and add information when available.


## 3. Common Join Pitfalls and How to Avoid Them

### Pitfall 1: Duplicate Keys
When your join key has duplicates in one or both tables, you get a cartesian product (explosion of rows).

**Example Problem**:
```
Table A: 3 rows with customer_id = 1
Table B: 2 rows with customer_id = 1
Result: 3 * 2 = 6 rows for customer_id = 1
```

**How to detect**:
```python
# Check for duplicates before joining
duplicates_df1 = df1.group_by("join_key").agg(pl.len().alias("count")).filter(pl.col("count") > 1)
duplicates_df2 = df2.group_by("join_key").agg(pl.len().alias("count")).filter(pl.col("count") > 1)
```

### Pitfall 2: Wrong Join Keys
Joining on the wrong column(s) leads to incorrect matches or no matches at all.

**Common mistakes**:
- Joining on columns with different data types (string vs integer)
- Joining on columns with different formats (leading zeros, case sensitivity)
- Joining on partial keys when you need composite keys

**How to prevent**:
```python
# Check data types
print(df1.schema)
print(df2.schema)

# Check for formatting issues
df1.select(pl.col("key").unique()).sort("key")
df2.select(pl.col("key").unique()).sort("key")
```

### Pitfall 3: Missing Values in Join Keys
NULL values in join keys never match, even with other NULLs.

**How to handle**:
```python
# Check for nulls in join keys
nulls_df1 = df1.filter(pl.col("join_key").is_null()).select(pl.len())
nulls_df2 = df2.filter(pl.col("join_key").is_null()).select(pl.len())

# Consider filling nulls or filtering them out before joining
df1_clean = df1.filter(pl.col("join_key").is_not_null())
```

### Pitfall 4: Not Understanding Data Grain After Joins
Joins can change the grain (level of detail) of your data. After joining, the new grain of the resulting dataset will be the same as the grain of the table with the foreign key.

**Example**:
- Before join: Each row = one customer
- After join with orders: Each row = one order (customers repeated)

**Always verify**:
```python
print(f"Rows before join: {len(df1)}")
print(f"Rows after join: {len(result)}")
print(f"Expected rows based on join type and data: ...")
```

## 4. Best Practices for Joins

### 1. Always Validate Your Join Keys
```python
# Before joining, check:
# 1. Uniqueness
# 2. Data types
# 3. Null values
# 4. Format consistency
```

### 2. Use Meaningful Variable Names
```python
# Good
customers_with_orders = customers.join(orders, on="customer_id", how="left")

# Less clear
df_result = df1.join(df2, on="id", how="left")
```

### 3. Document Your Joins
```python
# This join adds order information to each customer
# One-to-many relationship: each customer can have multiple orders
# Result grain: one row per order
customer_orders = customers.join(orders, on="customer_id", how="inner")
```

## 4. Business Context for Joins

### Common Business Scenarios

1. **Customer Analysis**: Join customer demographics with transaction data
2. **Product Analysis**: Join product catalog with sales data
3. **Financial Reporting**: Join budget data with actual expense data
4. **HR Analytics**: Join employee data with performance reviews

### Questions to Ask Before Joining

1. What is the business question I'm trying to answer?
2. What is the relationship between these datasets? (one-to-one, one-to-many, many-to-many)
3. What should happen to non-matching records?
4. What is the expected output grain?
5. How will I validate the results?

## Summary

1. **Joins connect related data** using common columns (keys)
2. **Primary keys uniquely identify rows**, foreign keys reference other tables
3. **Different join types** serve different purposes - choose based on your needs
4. **Common pitfalls** include duplicate keys, wrong keys, nulls, and grain changes
5. **Always validate** your joins by checking row counts and sampling results

Remember: Good joins start with understanding your data relationships and end with validating your results!