# HR Analytics with SQL
Generating data inputs to create HR dashboard at company level, department level and employee level

## 1. Problem statement

We have been tasked by HR Analytica to generate reusable data assets to power 2 of their client HR analytics tools.

We’ve also been asked specifically to generate database views that HR Analytica team can use for 2 key dashboards, reporting solutions and ad-hoc analytics requests.

Additionally - we’ve been alerted to the presence of date issues with our datasets where there were data-entry issues related to all DATE related fields - we will need to incorporate the fixes as we compile our solution.

### 1.1 Required Insights
The following insights must be generated for the dashboard
#### 1.1.1. Company Level
For all following metrics - we need a current snapshot of all the data as well as a split by gender specifically:

- Total number of employees
- Average company tenure in years
- Average latest payrise percentage
- Statistical metrics for salary values including:
- MIN, MAX, STDDEV, Inter-quartile range and median

#### 2.1.2. Department Level
All of the metrics as per the company level but at a department level, including an additional department level tenure metrics split by gender.

#### 2.1.3. Title Level
Similar to the department level metrics but instead, at a title level of granularity.

We will also need to generate similar insights for both department and title levels where we need employee count and average tenure for each level instead of company wide numbers.

### 1.2. Employee Deep Dive
The following insights must be generated for the Employee Deep Dive tool that can spotlight recent events for a single employee over time:

- See all the various employment history ordered by effective date including salary, department, manager and title changes
- Calculate previous historic payrise percentages and value changes
- Calculate the previous position and department history in months with start and end dates
- Compare an employee’s current salary, total company tenure, department, position and gender to the average benchmarks for their current position

### 1.3. Visual Outputs

#### Current Snapshot Reporting
___
<img src="/Company level.png"/>

#### Historic Employee Deep Dive
___
<img src="/Employee Deepdive.png"/>


## 2. Exploration
We've been alerted to the presence of data issues for all date related fields -> we need to inspect each table to see what adjustments we need to make

Additionally, we will start profiling all of our available tables to see how we can join themm for our complete analytical solution

From our initial inspection of the ERD, it also seems like there are slow changing dimention tables as we can see "from_date" and "to_date" columns for some tables.

Firstly, let's explore the available indexes from the employees schema before moving onto individual tables

```sql
SELECT *
FROM pg_indexes
where schemaname = 'employees';
```
| schemaname | tablename           | indexname           | indexdef                                                                                                        |
|------------|---------------------|---------------------|-----------------------------------------------------------------------------------------------------------------|
| employees  | employee            | idx_16988_primary   | CREATE UNIQUE INDEX idx_16988_primary ON employees.employee USING btree (id)                                    |
| employees  | department_employee | idx_16982_primary   | CREATE UNIQUE INDEX idx_16982_primary ON employees.department_employee USING btree (employee_id, department_id) |
| employees  | department_employee | idx_16982_dept_no   | CREATE INDEX idx_16982_dept_no ON employees.department_employee USING btree (department_id)                     |
| employees  | department          | idx_16979_primary   | CREATE UNIQUE INDEX idx_16979_primary ON employees.department USING btree (id)                                  |
| employees  | department          | idx_16979_dept_name | CREATE UNIQUE INDEX idx_16979_dept_name ON employees.department USING btree (dept_name)                         |
| employees  | department_manager  | idx_16985_primary   | CREATE UNIQUE INDEX idx_16985_primary ON employees.department_manager USING btree (employee_id, department_id)  |
| employees  | department_manager  | idx_16985_dept_no   | CREATE INDEX idx_16985_dept_no ON employees.department_manager USING btree (department_id)                      |
| employees  | salary              | idx_16991_primary   | CREATE UNIQUE INDEX idx_16991_primary ON employees.salary USING btree (employee_id, from_date)                  |
| employees  | title               | idx_16994_primary   | CREATE UNIQUE INDEX idx_16994_primary ON employees.title USING btree (employee_id, title, from_date)            |

Table employees.employee have unique indexes on a single column:
- employees.employee
- employees.department

The rest of the tables seem to have multiple records for the employee_id values based off the indexes:

- employees.department_employee
- employees.department_manager
- employees.salary
- employees.title

### 2.2. Individual Table Analysis
#### 2.2.1. Employee table
```sql
SELECT *
FROM employees.employee
LIMIT 5;
```
| id    | birth_date | first_name | last_name | gender | hire_date  |
|-------|------------|------------|-----------|--------|------------|
| 10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
| 10002 | 1964-06-02 | Bezalel    | Simmel    | F      | 1985-11-21 |
| 10003 | 1959-12-03 | Parto      | Bamford   | M      | 1986-08-28 |
| 10004 | 1954-05-01 | Chirstian  | Koblick   | M      | 1986-12-01 |
| 10005 | 1955-01-21 | Kyoichi    | Maliniak  | M      | 1989-09-12 |


Let's confirm there's only single record per employee as implied by the index
```sql
WITH id_cte AS (
  SELECT 
    id,
    COUNT(*) AS row_count
  FROM employees.employee
  GROUP BY id
)
SELECT  
  row_count,
  COUNT(id) AS employee_count
FROM id_cte
GROUP BY 1
ORDER BY 1 DESC;
```
| row_count | employee_count |
|-----------|----------------|
| 1         | 300024         |

Our initial hypothesis is that not all of our employees will exists in our other tables as there should be natural employee churn for any company

### Other tables exploration will be provided code below without the result output 
And sorry for my laziness as creating table in markdown is pretty tedious task for me xD

#### 2.2.2. department
```sql
SELECT *
FROM employees.department;
```
There are 9 records - 9 department in our company
#### 2.2.3. department_employee
```sql
SELECT *
FROM employees.department_employee
LIMIT 5;
```
There seems to be to_date = '9999-01-01' in our sample records - let's investigate the distribution of this to_date column:
```sql
SELECT
    to_date,
    COUNT(*) AS record_count
FROM employees.department_employee
GROUP BY to_date
ORDER BY record_count DESC
LIMIT 5;
```
 The majorty of values seems to 'current' with the arbitrary '9999-01-01' date value
 As we mentioned earlier, there are issues with date value, but not exactly this arbitrary value (as it already indicated the current date) - We need to leave this date value as is whe we are adding our fixes to the date input error that we mentioned before

#### 2.2.4. department_manager
```sql
SELECT *
FROM employees.department_manager
LIMIT 5;
```
Let's investigate the distribution of this to_date column to see how manay records are valid for the current period:
```sql
SELECT
    to_date,
    COUNT(*) AS record_count
FROM employees.department_manager
GROUP BY to_date
ORDER By record_count DESC;
```
Here we can see that there is vastly less movement for department managers as opposed to the department employees. There are 9 relevant current records which exactly matches up to the number of department from the department table - Good News!
```sql
SELECT
  employee_id,
  COUNT(*) AS row_count
FROM employees.department_manager
GROUP BY employee_id
ORDER BY row_count DESC
LIMIT 5;
```
There is no employee who has been managerred for more than 1 department.

#### 2.2.5. salary
Let's just inspect one employee only - employee with id = 10001
```sql
SELECT *
FROM employees.salary
WHERE employee_id = 10001
ORDER BY from_date DESC;
```
Each employee received pay raise overtime across there tenure period at the company - Multiple salary record for each employee over time, the most current on will have most recent from_date and to_date set to 'arbitrary 9999-01-01'
Let's investigate the distribution of the to_date column
```sql
SELECT
    to_date,
    COUNT(*) AS record_count,
    COUNT(DISTINCT employee_id) AS employee_count
FROM mv_employees.salary
GROUP BY 1
ORDER BY 1 DESC
LIMIT 5;
```
Inspecting how many record per employees to see how many deduplicates we should be expecting in our joins later on:
```sql
WITH employee_id_cte AS (
    SELECT
        employee_id,
        COUNT(*) AS row_count
    FROM employees.salary
    GROUP BY employee_id
)
SELECT
    row_count,
    COUNT(employee_id) AS employee_count
FROM employee_id_cte
GROUP BY row_count
ORDER BY row_count DESC;
```
The majority will have > 2 record - We should be careful when joining this table with the others

#### 2.2.6. title
```sql
SELECT *
FROM employees.title
LIMIT 5;
```
Title table contain employee current and historical title with from_date and to_date sinify effective and expire rate for each title.

```sql
SELECT
  to_date,
  COUNT(*) AS record_count,
  COUNT(DISTINCT employee_id) AS employee_count
FROM employees.title
GROUP BY 1
ORDER BY 1 DESC;
```
Let's confirm we have multiple rows for each employee_id - Many to one relationship 
```sql
WITH employee_id_cte AS (
    SELECT
        employee_id,
        COUNT(*) AS row_count
    FROM employees.title
    GROUP BY employee_id
)
SELECT
    row_count,
    COUNT(employee_id)
FROM employee_id_cte
GROUP BY row_count
ORDER BY row_count DESC;
```
Interestingly, the majority of employees never change their job title , 40% change onced, and only 3000 employees change twice (3 different titles)

## 4.1. Data cleaning

We will adjust all of our relevant data fieldsdue to the data issues identified by HR Analytica

We will be incrementing all of the data firelds except the arbitrary end date of 9999-01-01 - we will also need to cast our results back to a DATE data type as PostgreSQL interval addition forces the data type to a TIMESTAMP, which we'd like to avoid to keep the data type as similar to the original as possible

TO account for the future updates and to maximise the efficiency and productivity for the HR Analytica team - we will be implementing our adjusted datasets as materialised views with exact original indexes as per original tables in the employees schema

```sql
DROP SCHEMA IF EXISTS mv_employees CASCADE;
CREATE SCHEMA mv_employees;

-- department
DROP MATERIALIZED VIEW IF EXISTS mv_employees.department;
CREATE MATERIALIZED VIEW mv_employees.department AS
SELECT * FROM employees.department;

-- employee
DROP MATERIALIZED VIEW IF EXISTS mv_employees.employee;
CREATE MATERIALIZED VIEW mv_employees.employee AS
SELECT 
  id,
  (birth_date + INTERVAL '18 YEARS')::DATE AS birth_date,
  first_name,
  last_name,
  gender,
  (hire_date + INTERVAL '18 YEARS')::DATE AS hire_date
FROM employees.employee;

-- department_employee
DROP MATERIALIZED VIEW IF EXISTS mv_employees.department_employee;
CREATE MATERIALIZED VIEW mv_employees.department_employee AS
SELECT
  employee_id,
  department_id,
  (from_date + INTERVAL '18 YEARS')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01'
      THEN (to_date + INTERVAL '18 YEARS')::DATE
    ELSE to_date
    END AS to_date
FROM employees.department_employee;

-- department_mamanger
DROP MATERIALIZED VIEW IF EXISTS mv_employees.department_manager;
CREATE MATERIALIZED VIEW mv_employees.department_manager AS
SELECT 
  employee_id,
  department_id,
  (from_date + INTERVAL '18 YEARS')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01'
      THEN (to_date + INTERVAL '18 YEARS')::DATE
    ELSE to_date
    END AS to_date
FROM employees.department_manager;

-- salary
DROP MATERIALIZED VIEW IF EXISTS mv_employees.salary;
CREATE MATERIALIZED VIEW mv_employees.salary AS
SELECT 
  employee_id,
  amount,
  (from_date + INTERVAL '18 YEARS')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01'
      THEN (to_date + INTERVAL '18 YEARS')::DATE
    ELSE to_date
    END AS to_date
FROM employees.salary;

-- title
DROP MATERIALIZED VIEW IF EXISTS mv_employees.title;
CREATE MATERIALIZED VIEW mv_employees.title AS
SELECT 
  employee_id,
  title,
  (from_date + INTERVAL '18 YEARS')::DATE AS from_date,
  CASE
    WHEN to_date <> '9999-01-01' 
      THEN (to_date + INTERVAL '18 YEARS')::DATE
    ELSE to_date
    END AS to_date
FROM employees.title;

```

#### Index creation

```sql
CREATE UNIQUE INDEX ON mv_employees.employee USING btree(id);
CREATE UNIQUE INDEX ON mv_employees.department_employee USING btree(employee_id, department_id);
CREATE INDEX ON mv_employees.department_employee USING btree(department_id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree(id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree(dept_name);
CREATE UNIQUE INDEX ON mv_employees.department_manager USING btree(employee_id, department_id);
CREATE INDEX ON mv_employees.department_manager USING btree(department_id);
CREATE UNIQUE INDEX ON mv_employees.salary USING btree(employee_id, from_date);
CREATE UNIQUE INDEX ON mv_employees.TITLE USING btree(employee_id,title, from_date);
```

### Main analysis 

After cleaning and transformation, we will go to the main analysis part where we gather data inputs for the Current Snapshot Views at Company, Department and Title level

We need these information:
1. number of employees (`employee` view)
2. gender (`employee` view)
3. tenure (using `hire_date` from `employee` view and current date to calculate)
4. payrise (`salary` view and `LAG` function)
5. salary summary satistics (`salary` view)
6. to get the snapshot views at department and title level, we also need to access data from `department`,`department_employee` and `title` views.

### Analysis Plan
1. Use `LAG` function to get the previous salary for each employees from employee view
2. Join step 1 with employee current salary information to get **payrise** information
3. Join step 2 with employee view (get `hire_date` and `gender`), with title view (get `title`, `from_date`), with department_employee and department (get `dept_name` a.k.a department name, `from_date`)
4. Apply `WHERE` filter to keep only current records
5. Use `hire_date` to calculate **tenure**
6. Use `from_date` from `title` and `department_employee` to calculate tenure for title and department snapshot
7. Use summary function and window function to calculate various salary statistics (MEAN, MAX, INTERQUATILE, etc.) 
8. COmbine all of these elements into a single final current snapshot view

```sql

DROP VIEW IF EXISTS mv_employees.current_employee_snapshot CASCADE;
CREATE VIEW mv_employees.current_employee_snapshot AS
-- apply LAG to get previous salary amount for all employees
WITH previous_salary_cte AS (
-- Step 1: Get previous salary
  SELECT * FROM (
    SELECT
      employee_id,
      to_date,
      amount as salary,
      LAG(amount) OVER (
        PARTITION BY employee_id
        ORDER BY to_date
      ) AS prev_salary
    FROM mv_employees.salary
  ) salaries
  WHERE to_date = '9999-01-01'
),
-- Step 2: Calculate payrise information - salary_percentage_change
payrise_cte AS (
  SELECT 
    employee_id,
    salary,
    ROUND (
      100 * (prev_salary) / prev_salary::NUMERIC,
      2
    ) AS salary_percentage_change
  FROM previous_salary_cte
),
-- Step 3: Join step 2 with other table view
joined_data_cte AS (
  SELECT
    employee.id AS employee_id,
    employee.gender,
    employee.hire_date,
    payrise_cte.salary,
    payrise_cte.salary_percentage_change,
    title.title,
    department.dept_name AS department,
    title.from_date AS title_from_date,
    department_employee.from_date AS department_from_date
  FROM mv_employees.employee
  INNER JOIN payrise_cte 
    ON employee.id = payrise_cte.employee_id
  INNER JOIN mv_employees.title AS title
    ON employee.id = title.employee_id
  INNER JOIN mv_employees.department_employee
    ON employee.id = department_employee.employee_id
  INNER JOIN mv_employees.department
    ON department_employee.department_id = department.id
-- Step 4: Apply filter to keep only current records
  WHERE 
    title.to_date = '9999-01-01'
    AND department_employee.to_date = '9999-01-01'
),
-- Step 5, 6, 7, 8: Apply all of our calculation in this final output
final_output AS (
  SELECT
    employee_id,
    gender,
    title,
    salary,
    department
    salary_percentage_change,
    -- Step 5: company tenure
    DATE_PART('year', now()) - DATE_PART('year', hire_date) AS company_tenure_year,
    -- Step 6: title and department tenure
    DATE_PART('year', now()) - DATE_PART('year', title_from_date) AS title_tenure_year,
    DATE_PART('year', now()) - DATE_PART('year', department_from_date) AS department_tenure_year
    -- We skip step 7 because we don't need to aggregate salary at this point yet
  FROM joined_data_cte
)
SELECT * FROM final_output;
```

Sample rows from `mv_employees.current_employee_snapshot`

| employee_id | gender | title           | salary | department      | salary_percentage_change | company_tenure_years | title_tenure_years | department_tenure_years |
|-------------|--------|-----------------|--------|-----------------|--------------------------|----------------------|--------------------|-------------------------|
| 10001       | M      | Senior Engineer | 88958  | Development     | 4.54                     | 17                   | 17                 | 17                      |
| 10002       | F      | Staff           | 72527  | Sales           | 0.78                     | 18                   | 7                  | 7                       |
| 10003       | M      | Senior Engineer | 43311  | Production      | -0.89                    | 17                   | 8                  | 8                       |
| 10004       | M      | Senior Engineer | 74057  | Production      | 4.75                     | 17                   | 8                  | 17                      |
| 10005       | M      | Senior Staff    | 94692  | Human Resources | 3.54                     | 14                   | 7                  | 14                      |

