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

<details> 
  <summary> 1.3.1. Current Snapshot Reporting
  </summary>
___
<img src="/Company level.png"/>
</details>

<details> 
  <summary> 1.3.2. Historic Employee Deep Dive
  </summary>
___
<img src="/Employee Deepdive.png"/>
</details>

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

## 3.1. Data cleaning

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

-- department_manager
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
      100 * (salary - prev_salary) / prev_salary::NUMERIC,
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
    department,
    salary_percentage_change,
    -- Step 5: company tenure
    DATE_PART('year', now()) - DATE_PART('year', hire_date) AS company_tenure_years,
    -- Step 6: title and department tenure
    DATE_PART('year', now()) - DATE_PART('year', title_from_date) AS title_tenure_years,
    DATE_PART('year', now()) - DATE_PART('year', department_from_date) AS department_tenure_years
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

### Dashboard aggregation views
#### Company level

```sql
DROP VIEW IF EXISTS mv_employees.company_level_dashboard;
CREATE VIEW mv_employees.company_level_dashboard AS
SELECT 
  gender,
  COUNT(employee_id) as employee_count,
  ROUND(100 * COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER ()) AS employee_percentage,
  ROUND(AVG(company_tenure_years)) As company_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  -- salary statistics
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median,
  ROUND(
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) -
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)
  ) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender;
```

#### Department level insights
```sql
DROP VIEW IF EXISTS mv_employees.department_level_dashboard;
CREATE VIEW mv_employees.department_level_dashboard AS
SELECT 
  gender,
  department,
  COUNT(*) as employee_count,
  ROUND(100 * COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER (PARTITION BY department)) AS employee_percentage,
  ROUND(AVG(department_tenure_years)) As department_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  -- salary statistics
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median,
  ROUND(
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) -
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)
  ) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender, department;
```

#### Title level insights
```sql
DROP VIEW IF EXISTS mv_employees.title_level_dashboard;
CREATE VIEW mv_employees.title_level_dashboard AS
SELECT
  gender,
  title,
  COUNT(*) AS employee_count,
  ROUND(100 * COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER (PARTITION BY title)) AS employee_percentage,
  ROUND(AVG(title_tenure_years)) AS title_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  -- salary statistics
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median,
  ROUND(
      PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary)
      - PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary)
  ) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 
  gender,
  title;
```

## Employee Deep Dive
For the historic employee deep dive analysis - we will need to split up our interim outputs into 3 parts:

### 1. Current Employee Information
- Full name
- Gender
- Birthday
- Department
- Title/Position tenure
- Company tenure
- Current salary
- Latest salary change percentage
- Manager name

### 2. Salary comparison to various benchmarks including:
- Company tenure
- Title/Position
- Department
- Gender

### 3. The last 5 historical employee events categorised into:
- Salary increase/decrease
- Department transfer
- Manager reporting line change
- Title changes

For **1.Current Employee Information**, we can utilise the previous current_employee_snapshot view from previous analysis, the only difference is the addition of manager information that we can get from de_partment manager table

```sql
-- 1. Replace the view with an updated version with manager info
CREATE OR REPLACE VIEW mv_employees.current_employee_snapshot AS
-- apply LAG to get previous salary amount for all employees
WITH cte_previous_salary AS (
  SELECT * FROM (
    SELECT
      employee_id,
      to_date,
      LAG(amount) OVER (
        PARTITION BY employee_id
        ORDER BY from_date
      ) AS amount
    FROM mv_employees.salary
  ) all_salaries
  -- keep only latest valid previous_salary records only
  -- must have this in subquery to account for execution order
  WHERE to_date = '9999-01-01'
),
-- combine all elements into a joined CTE
cte_joined_data AS (
  SELECT
    employee.id AS employee_id,
    -- include employee full name
    CONCAT_WS(' ', employee.first_name, employee.last_name) AS employee_name,
    employee.gender,
    employee.hire_date,
    title.title,
    salary.amount AS salary,
    cte_previous_salary.amount AS previous_salary,
    department.dept_name AS department,
    -- include manager full name
    CONCAT_WS(' ', manager.first_name, manager.last_name) AS manager,
    -- need to keep the title and department from_date columns for tenure calcs
    title.from_date AS title_from_date,
    department_employee.from_date AS department_from_date
  FROM mv_employees.employee
  INNER JOIN mv_employees.title
    ON employee.id = title.employee_id
  INNER JOIN mv_employees.salary
    ON employee.id = salary.employee_id
  -- join onto the CTE we created in th first step
  INNER JOIN cte_previous_salary
    ON employee.id = cte_previous_salary.employee_id
  INNER JOIN mv_employees.department_employee
    ON employee.id = department_employee.employee_id
  -- department is joined only to the department_employee table!
  INNER JOIN mv_employees.department
    ON department_employee.department_id = department.id
  -- add in the department_manager information onto the department table
  INNER JOIN mv_employees.department_manager
    ON department.id = department_manager.department_id
  -- join again on the employee_id field to another employee for manager's info
  INNER JOIN mv_employees.employee AS manager
    ON department_manager.employee_id = manager.id
  -- appply where filter to keep only relevant records
  WHERE salary.to_date = '9999-01-01'
    AND title.to_date = '9999-01-01'
    AND department_employee.to_date = '9999-01-01'
    -- add in department_manager to_date column filter
    AND department_manager.to_date = '9999-01-01'
)
-- finally we can aply all our calculations in this final output
SELECT
  employee_id,
  gender,
  title,
  salary,
  department,
  -- salary change percentage
  ROUND(
    100 * (salary - previous_salary) / previous_salary :: NUMERIC,
    2
  ) AS salary_percentage_change,
  -- tenure calculations
  DATE_PART('year', now()) - DATE_PART('year', hire_date) AS company_tenure_years,
  DATE_PART('year', now()) - DATE_PART('year', title_from_date) AS title_tenure_years,
  DATE_PART('year', now()) - DATE_PART('year', department_from_date) AS department_tenure_years,
  -- need to add the two newest fields after original view columns!
  employee_name,
  manager
FROM cte_joined_data;
```

The generate salary benchmark view from company tenure, gender, department and title

```sql
-- Note the slightly vebose column names - this helps us avoid renaming later

DROP VIEW IF EXISTS mv_employees.tenure_benchmark;
CREATE VIEW mv_employees.tenure_benchmark AS
SELECT
  company_tenure_years,
  AVG(salary) AS tenure_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY company_tenure_years;

DROP VIEW IF EXISTS mv_employees.gender_benchmark;
CREATE VIEW mv_employees.gender_benchmark AS
SELECT
  gender,
  AVG(salary) AS gender_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender;

DROP VIEW IF EXISTS mv_employees.department_benchmark;
CREATE VIEW mv_employees.department_benchmark AS
SELECT
  department,
  AVG(salary) AS department_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY department;

DROP VIEW IF EXISTS mv_employees.title_benchmark;
CREATE VIEW mv_employees.title_benchmark AS
SELECT
  title,
  AVG(salary) AS title_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY title;
```

To drive the final output for employee deepdive, apart from their current statistics (position tenure, company tenure, salary and manager), we also want to look at 5 previous change that happen with the manager (salary change, title change, manager change, department change, etc.). Thus we will conduct a further join to join historical data using GREATEST and LEAST query to filter out only valid join

Here are the steps:

1. For each employee, calculate the latest salary (the one that got salary_to_date = `'9999-01-01'`) and the previous latest salary (using LAG) to calculate salary percentage change

2. Join data from step 1 with employees data (employees, salary, title, department, manager), calculate the valid date range with GREATEST ( `salary_from_date`, `title_from_date`, `department_from_date`, `manager_from_date`) as `effective_date` and LEAST(`to_date`s data) as `expire_date` and using WHERE 
to filter only data that have `effective_date` < `expire_date`
3. In the next query, calculate the previous employee information (previous salary, previous title, previous dept, previous manager)

4. In the next query, join preivous joined data with the salary benchmark data (tenure, position, department, gender benchmark) to calculate salary difference. At the same time, using CASE condition function to determine what type of event happening for each date range by comparing all of the previous lag records.

5. In the final views, as we only want to see the 5 latest events, we use `ROW_NUMBER()` windows function to filter only event having row_number <= 5;

<details>
  <summary> Click here to see the entire SQL code </summary>

```sql
DROP VIEW IF EXISTS mv_employees.historic_employee_records CASCADE;
CREATE VIEW mv_employees.historic_employee_records AS
-- we need the previous salary only for the latest record
WITH cte_previous_salary AS (
  SELECT * FROM (
    SELECT
      employee_id,
      to_date,
      LAG(amount) OVER (
        PARTITION BY employee_id
        ORDER BY to_date
      ) AS amount,
      ROW_NUMBER() OVER (
          PARTITION BY employee_id
          ORDER BY to_date DESC
      ) AS record_rank
    FROM mv_employees.salary
  ) all_salries
  -- keep only latest previous_salary records only
  WHERE record_rank = 1
),
cte_join_data AS (
  SELECT
    employee.id AS employee_id,
    employee.birth_date,
    -- calculate employee_age field
    DATE_PART('YEAR', now()) - 
      DATE_PART('YEAR', employee.birth_date) AS employee_age,
    -- employee fullname
    CONCAT_WS(' ', employee.first_name, employee.last_name) AS employee_name,
    employee.gender,
    employee.hire_date,
    title.title,
    salary.amount AS salary,
    -- need to separtely define the previous_latest_salary
    -- to differentiate between the following lag record!
    cte_previous_salary.amount AS previous_latest_salary,
    department.dept_name AS department,
    -- use the 'mamanger aliased version of employee table for manager
    CONCAT_WS(' ', manager.first_name,manager.last_name) AS manager,
    -- calculated tenure fields
    DATE_PART('YEAR', now()) -
      DATE_PART('YEAR', employee.hire_date) AS company_tenure_years,
    DATE_PART('YEAR', now()) - 
      DATE_PART('YEAR', title.from_date) AS title_tenure_years,
    DATE_PART('YEAR', now()) - 
      DATE_PART('year', department_employee.from_date) AS department_tenure_years,
    -- we also need to use age & date_part functions here to generate month difference
    DATE_PART('MONTHS', AGE(now(), title.from_date)) AS title_tenure_months,
    GREATEST(
      title.from_date,
      salary.from_date,
      department_employee.from_date,
      department_manager.from_date
    ) AS effective_date,
    LEAST(
      title.to_date,
      salary.to_date,
      department_employee.to_date,
      department_manager.to_date
    ) AS expiry_date
  FROM mv_employees.employee
  INNER JOIN mv_employees.title
    ON employee.id = title.employee_id
  INNER JOIN mv_employees.salary
    ON employee.id = salary.employee_id
  -- join with the cte_previous_salary only for the previous_latest_salary
  INNER JOIN cte_previous_salary
    ON employee.id = cte_previous_salary.employee_id
  INNER JOIN mv_employees.department_employee
    ON employee.id = department_employee.employee_id
  -- department table is joined only with the department_employee
  INNER JOIN mv_employees.department
    ON department_employee.department_id = department.id
  -- add in the department_manager information
  INNER JOIN mv_employees.department_manager
    ON department.id = department_manager.department_id
  -- join again on the employee_id field to another employee for manager's information
  INNER JOIN mv_employees.employee AS manager
    ON department_manager.employee_id = manager.id
  -- filter only valid records resulting from the join
),
-- 
cte_ordered_events AS (
  SELECT 
    employee_id,
    birth_date,
    employee_age,
    employee_name,
    gender,
    hire_date,
    title,
    LAG(title) OVER w AS previous_title,
    salary,
    -- pprevious latest salary is based off the CTE
    previous_latest_salary,
    LAG(salary) OVER w AS previous_salary,
    department,
    LAG(department) OVER w AS previous_department,
    manager,
    LAG(manager) OVER w AS previous_manager,
    company_tenure_years,
    title_tenure_years,
    title_tenure_months,
    department_tenure_years,
    effective_date,
    expiry_date,
    -- we use a reversed ordered effective date window to capture the latest 5 events
    ROW_NUMBER() OVER (
      PARTITION BY employee_id
      ORDER BY effective_date DESC
    ) AS event_order
  FROM cte_join_data
  WHERE effective_date <= expiry_date

  WINDOW 
    w AS (PARTITION BY employee_id ORDER BY effective_date)
),
-- finnally we apply our case when statements to generate the employee events
-- and generate our benchmark comparisons for the final output
-- we aliased our FROM table as "base" for compact code!
final_output AS (
  SELECT
    base.employee_id,
    base.gender,
    base.birth_date,
    base.employee_age,
    base.hire_date,
    base.title,
    base.employee_name,
    base.previous_title,
    base.salary,
    -- previous latest salary is based off the CTE
    base.previous_latest_salary,
    -- previous salary is based off the LAG records
    base.previous_salary,
    base.department,
    base.previous_department,
    base.manager,
    base.previous_manager,
    -- tenure metrics
    base.company_tenure_years,
    base.title_tenure_years,
    base.title_tenure_months,
    base.department_tenure_years,
    base.event_order,
    -- only include the laest salary change for the first event_order row
    CASE 
      WHEN event_order = 1
        THEN ROUND (
            100 * (base.salary - base.previous_latest_salary) /
            base.previous_latest_salary::NUMERIC,
            2
        )
      ELSE NULL
    END AS latest_salary_percentage_change,
    -- event type logic by comparing all of the previous lag records,
    CASE 
      WHEN base.previous_salary < base.salary
        THEN 'Salary Increase'
      WHEN base.previous_salary > base.salary 
        THEN 'Salary Decrease'
      WHEN base.previous_department <> base.department
        THEN 'Dept Transfer'
      WHEN base.previous_manager <> base.manager
        THEN 'Reporting Line Change'
      WHEN base.previous_title <> base.title
        THEN 'Title Change'
      ELSE NULL
    END AS event_name,
    -- salary change
    ROUND(
      100 * (base.salary - base.previous_salary) / base.previous_salary::NUMERIC,
      2
    ) AS salary_percentage_change,
    ROUND(base.salary - base.previous_salary) AS salary_amount_change,
    -- benchmark comparisons - we've omit the aliases for succintness!
    -- tenure
    ROUND(tenure_benchmark_salary) AS tenure_benchmark_salary,
    ROUND(
        100 * (base.salary - tenure_benchmark_salary) / tenure_benchmark_salary,
        2
    ) AS tenure_comparison,
    -- title
    ROUND(title_benchmark_salary) AS title_benchmark_salary,
    ROUND(
        100 * (base.salary - title_benchmark_salary) / title_benchmark_salary,
        2
    ) AS title_comparison,
    -- department
    ROUND(department_benchmark_salary) AS department_benchmark_salary,
    ROUND(
        100 * (base.salary - department_benchmark_salary) / title_benchmark_salary,
        2
    ) AS department_comparison,
    -- gender
    ROUND(gender_benchmark_salary) AS gender_benchmark_salary,
    ROUND(
        100 * (base.salary - gender_benchmark_salary) / title_benchmark_salary,
        2
    ) AS gender_comparison,
    -- leave the effective/expiry dates at the end of the query
    base.effective_date,
    base.expiry_date
  FROM cte_ordered_events AS base
  INNER JOIN mv_employees.tenure_benchmark
    ON base.company_tenure_years = tenure_benchmark.company_tenure_years
  INNER JOIN mv_employees.title_benchmark
    ON base.title = title_benchmark.title
  INNER JOIN mv_employees.department_benchmark
    ON base.department = department_benchmark.department
  INNER JOIN mv_employees.gender_benchmark
    ON base.gender = gender_benchmark.gender
  -- apply filter to only keep the latest 5 events per employee
  -- WHERE event_order <= 5
)
SELECT * FROM final_output;

DROP VIEW IF EXISTS mv_employees.employee_deep_dive;
CREATE VIEW mv_employees.employee_deep_dive AS
SELECT * FROM mv_employees.historic_employee_records
WHERE event_order <= 5;
```
</details>

## 4. Report

Finally let's summarise the results of our analysis and compile a complete end to end SQL script we can use to recreate our entire workflow

We will also regenerate the example data points provided for our visual examples for both the current and historic analysis components

### 4.1. Final SQL Script
The script is broken into 5 sections

1. Create materialized views to fix date data issues
2. Current employee snapshot view
3. Aggregated dashboard views
4. Salary benchmark views
5. Historic employee deep dive view

<details>
<summary>
This SQL script is really long - click to see the entire thing
</summary>

```sql
/*---------------------------------------------------
1. Create materialized views to fix date issues data
---------------------------------------------------*/

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

-- department_manager
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

-- Index creation

CREATE UNIQUE INDEX ON mv_employees.employee USING btree(id);
CREATE UNIQUE INDEX ON mv_employees.department_employee USING btree(employee_id, department_id);
CREATE INDEX ON mv_employees.department_employee USING btree(department_id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree(id);
CREATE UNIQUE INDEX ON mv_employees.department USING btree(dept_name);
CREATE UNIQUE INDEX ON mv_employees.department_manager USING btree(employee_id, department_id);
CREATE INDEX ON mv_employees.department_manager USING btree(department_id);
CREATE UNIQUE INDEX ON mv_employees.salary USING btree(employee_id, from_date);
CREATE UNIQUE INDEX ON mv_employees.TITLE USING btree(employee_id,title, from_date);


/*-----------------------------------
2. Create current employee snapshot view
-----------------------------------*/

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
      100 * (salary - prev_salary) / prev_salary::NUMERIC,
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
    department,
    salary_percentage_change,
    -- Step 5: company tenure
    DATE_PART('year', now()) - DATE_PART('year', hire_date) AS company_tenure_years,
    -- Step 6: title and department tenure
    DATE_PART('year', now()) - DATE_PART('year', title_from_date) AS title_tenure_years,
    DATE_PART('year', now()) - DATE_PART('year', department_from_date) AS department_tenure_years
    -- We skip step 7 because we don't need to aggregate salary at this point yet
  FROM joined_data_cte
)
SELECT * FROM final_output;

/*--------------------------
3.Aggregated dashboard views
---------------------------*/

-- Company aggregation level insights
DROP VIEW IF EXISTS mv_employees.company_level_dashboard;
CREATE VIEW mv_employees.company_level_dashboard AS
SELECT 
  gender,
  COUNT(employee_id) as employee_count,
  ROUND(100 * COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER ()) AS employee_percentage,
  ROUND(AVG(company_tenure_years)) As company_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  -- salary statistics
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median,
  ROUND(
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) -
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)
  ) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender;

-- Department level insights
DROP VIEW IF EXISTS mv_employees.department_level_dashboard;
CREATE VIEW mv_employees.department_level_dashboard AS
SELECT 
  gender,
  department,
  COUNT(*) as employee_count,
  ROUND(100 * COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER (PARTITION BY department)) AS employee_percentage,
  ROUND(AVG(department_tenure_years)) As department_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  -- salary statistics
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median,
  ROUND(
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) -
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)
  ) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender, department;

-- Title level insights
DROP VIEW IF EXISTS mv_employees.title_level_dashboard;
CREATE VIEW mv_employees.title_level_dashboard AS
SELECT
  gender,
  title,
  COUNT(*) AS employee_count,
  ROUND(100 * COUNT(*)::NUMERIC / SUM(COUNT(*)) OVER (PARTITION BY title)) AS employee_percentage,
  ROUND(AVG(title_tenure_years)) AS title_tenure,
  ROUND(AVG(salary)) AS avg_salary,
  ROUND(AVG(salary_percentage_change)) AS avg_salary_percentage_change,
  -- salary statistics
  ROUND(MIN(salary)) AS min_salary,
  ROUND(MAX(salary)) AS max_salary,
  ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)) AS median,
  ROUND(
      PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary)
      - PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary)
  ) AS inter_quartile_range,
  ROUND(STDDEV(salary)) AS stddev_salary
FROM mv_employees.current_employee_snapshot
GROUP BY 
  gender,
  title;

/*-----------------------
4. Salary Benchmark Views
-----------------------*/

-- Replace the view with an updated version with manager info
CREATE OR REPLACE VIEW mv_employees.current_employee_snapshot AS
-- apply LAG to get previous salary amount for all employees
WITH cte_previous_salary AS (
  SELECT * FROM (
    SELECT
      employee_id,
      to_date,
      LAG(amount) OVER (
        PARTITION BY employee_id
        ORDER BY from_date
      ) AS amount
    FROM mv_employees.salary
  ) all_salaries
  -- keep only latest valid previous_salary records only
  -- must have this in subquery to account for execution order
  WHERE to_date = '9999-01-01'
),
-- combine all elements into a joined CTE
cte_joined_data AS (
  SELECT
    employee.id AS employee_id,
    -- include employee full name
    CONCAT_WS(' ', employee.first_name, employee.last_name) AS employee_name,
    employee.gender,
    employee.hire_date,
    title.title,
    salary.amount AS salary,
    cte_previous_salary.amount AS previous_salary,
    department.dept_name AS department,
    -- include manager full name
    CONCAT_WS(' ', manager.first_name, manager.last_name) AS manager,
    -- need to keep the title and department from_date columns for tenure calcs
    title.from_date AS title_from_date,
    department_employee.from_date AS department_from_date
  FROM mv_employees.employee
  INNER JOIN mv_employees.title
    ON employee.id = title.employee_id
  INNER JOIN mv_employees.salary
    ON employee.id = salary.employee_id
  -- join onto the CTE we created in th first step
  INNER JOIN cte_previous_salary
    ON employee.id = cte_previous_salary.employee_id
  INNER JOIN mv_employees.department_employee
    ON employee.id = department_employee.employee_id
  -- department is joined only to the department_employee table!
  INNER JOIN mv_employees.department
    ON department_employee.department_id = department.id
  -- add in the department_manager information onto the department table
  INNER JOIN mv_employees.department_manager
    ON department.id = department_manager.department_id
  -- join again on the employee_id field to another employee for manager's info
  INNER JOIN mv_employees.employee AS manager
    ON department_manager.employee_id = manager.id
  -- appply where filter to keep only relevant records
  WHERE salary.to_date = '9999-01-01'
    AND title.to_date = '9999-01-01'
    AND department_employee.to_date = '9999-01-01'
    -- add in department_manager to_date column filter
    AND department_manager.to_date = '9999-01-01'
)
-- finally we can aply all our calculations in this final output
SELECT
  employee_id,
  gender,
  title,
  salary,
  department,
  -- salary change percentage
  ROUND(
    100 * (salary - previous_salary) / previous_salary :: NUMERIC,
    2
  ) AS salary_percentage_change,
  -- tenure calculations
  DATE_PART('year', now()) - DATE_PART('year', hire_date) AS company_tenure_years,
  DATE_PART('year', now()) - DATE_PART('year', title_from_date) AS title_tenure_years,
  DATE_PART('year', now()) - DATE_PART('year', department_from_date) AS department_tenure_years,
  -- need to add the two newest fields after original view columns!
  employee_name,
  manager
FROM cte_joined_data;

-- Generate a benchmark views for company tenure, gender, department and title
-- Note the slightly vebose column names - this helps us avoid renaming later

DROP VIEW IF EXISTS mv_employees.tenure_benchmark;
CREATE VIEW mv_employees.tenure_benchmark AS
SELECT
  company_tenure_years,
  AVG(salary) AS tenure_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY company_tenure_years;

DROP VIEW IF EXISTS mv_employees.gender_benchmark;
CREATE VIEW mv_employees.gender_benchmark AS
SELECT
  gender,
  AVG(salary) AS gender_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY gender;

DROP VIEW IF EXISTS mv_employees.department_benchmark;
CREATE VIEW mv_employees.department_benchmark AS
SELECT
  department,
  AVG(salary) AS department_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY department;

DROP VIEW IF EXISTS mv_employees.title_benchmark;
CREATE VIEW mv_employees.title_benchmark AS
SELECT
  title,
  AVG(salary) AS title_benchmark_salary
FROM mv_employees.current_employee_snapshot
GROUP BY title;


/*---------------------------------
5. Historic Employee Deep Dive View
----------------------------------*/

DROP VIEW IF EXISTS mv_employees.historic_employee_records CASCADE;
CREATE VIEW mv_employees.historic_employee_records AS
-- we need the previous salary only for the latest record
WITH cte_previous_salary AS (
  SELECT * FROM (
    SELECT
      employee_id,
      to_date,
      LAG(amount) OVER (
        PARTITION BY employee_id
        ORDER BY to_date
      ) AS amount,
      ROW_NUMBER() OVER (
          PARTITION BY employee_id
          ORDER BY to_date DESC
      ) AS record_rank
    FROM mv_employees.salary
  ) all_salries
  -- keep only latest previous_salary records only
  WHERE record_rank = 1
),
cte_join_data AS (
  SELECT
    employee.id AS employee_id,
    employee.birth_date,
    -- calculate employee_age field
    DATE_PART('YEAR', now()) - 
      DATE_PART('YEAR', employee.birth_date) AS employee_age,
    -- employee fullname
    CONCAT_WS(' ', employee.first_name, employee.last_name) AS employee_name,
    employee.gender,
    employee.hire_date,
    title.title,
    salary.amount AS salary,
    -- need to separtely define the previous_latest_salary
    -- to differentiate between the following lag record!
    cte_previous_salary.amount AS previous_latest_salary,
    department.dept_name AS department,
    -- use the 'mamanger aliased version of employee table for manager
    CONCAT_WS(' ', manager.first_name,manager.last_name) AS manager,
    -- calculated tenure fields
    DATE_PART('YEAR', now()) -
      DATE_PART('YEAR', employee.hire_date) AS company_tenure_years,
    DATE_PART('YEAR', now()) - 
      DATE_PART('YEAR', title.from_date) AS title_tenure_years,
    DATE_PART('YEAR', now()) - 
      DATE_PART('year', department_employee.from_date) AS department_tenure_years,
    -- we also need to use age & date_part functions here to generate month difference
    DATE_PART('MONTHS', AGE(now(), title.from_date)) AS title_tenure_months,
    GREATEST(
      title.from_date,
      salary.from_date,
      department_employee.from_date,
      department_manager.from_date
    ) AS effective_date,
    LEAST(
      title.to_date,
      salary.to_date,
      department_employee.to_date,
      department_manager.to_date
    ) AS expiry_date
  FROM mv_employees.employee
  INNER JOIN mv_employees.title
    ON employee.id = title.employee_id
  INNER JOIN mv_employees.salary
    ON employee.id = salary.employee_id
  -- join with the cte_previous_salary only for the previous_latest_salary
  INNER JOIN cte_previous_salary
    ON employee.id = cte_previous_salary.employee_id
  INNER JOIN mv_employees.department_employee
    ON employee.id = department_employee.employee_id
  -- department table is joined only with the department_employee
  INNER JOIN mv_employees.department
    ON department_employee.department_id = department.id
  -- add in the department_manager information
  INNER JOIN mv_employees.department_manager
    ON department.id = department_manager.department_id
  -- join again on the employee_id field to another employee for manager's information
  INNER JOIN mv_employees.employee AS manager
    ON department_manager.employee_id = manager.id
  -- filter only valid records resulting from the join
),
-- 
cte_ordered_events AS (
  SELECT 
    employee_id,
    birth_date,
    employee_age,
    employee_name,
    gender,
    hire_date,
    title,
    LAG(title) OVER w AS previous_title,
    salary,
    -- pprevious latest salary is based off the CTE
    previous_latest_salary,
    LAG(salary) OVER w AS previous_salary,
    department,
    LAG(department) OVER w AS previous_department,
    manager,
    LAG(manager) OVER w AS previous_manager,
    company_tenure_years,
    title_tenure_years,
    title_tenure_months,
    department_tenure_years,
    effective_date,
    expiry_date,
    -- we use a reversed ordered effective date window to capture the latest 5 events
    ROW_NUMBER() OVER (
      PARTITION BY employee_id
      ORDER BY effective_date DESC
    ) AS event_order
  FROM cte_join_data
  WHERE effective_date <= expiry_date

  WINDOW 
    w AS (PARTITION BY employee_id ORDER BY effective_date)
),
-- finnally we apply our case when statements to generate the employee events
-- and generate our benchmark comparisons for the final output
-- we aliased our FROM table as "base" for compact code!
final_output AS (
  SELECT
    base.employee_id,
    base.gender,
    base.birth_date,
    base.employee_age,
    base.hire_date,
    base.title,
    base.employee_name,
    base.previous_title,
    base.salary,
    -- previous latest salary is based off the CTE
    base.previous_latest_salary,
    -- previous salary is based off the LAG records
    base.previous_salary,
    base.department,
    base.previous_department,
    base.manager,
    base.previous_manager,
    -- tenure metrics
    base.company_tenure_years,
    base.title_tenure_years,
    base.title_tenure_months,
    base.department_tenure_years,
    base.event_order,
    -- only include the laest salary change for the first event_order row
    CASE 
      WHEN event_order = 1
        THEN ROUND (
            100 * (base.salary - base.previous_latest_salary) /
            base.previous_latest_salary::NUMERIC,
            2
        )
      ELSE NULL
    END AS latest_salary_percentage_change,
    -- event type logic by comparing all of the previous lag records,
    CASE 
      WHEN base.previous_salary < base.salary
        THEN 'Salary Increase'
      WHEN base.previous_salary > base.salary 
        THEN 'Salary Decrease'
      WHEN base.previous_department <> base.department
        THEN 'Dept Transfer'
      WHEN base.previous_manager <> base.manager
        THEN 'Reporting Line Change'
      WHEN base.previous_title <> base.title
        THEN 'Title Change'
      ELSE NULL
    END AS event_name,
    -- salary change
    ROUND(
      100 * (base.salary - base.previous_salary) / base.previous_salary::NUMERIC,
      2
    ) AS salary_percentage_change,
    ROUND(base.salary - base.previous_salary) AS salary_amount_change,
    -- benchmark comparisons - we've omit the aliases for succintness!
    -- tenure
    ROUND(tenure_benchmark_salary) AS tenure_benchmark_salary,
    ROUND(
        100 * (base.salary - tenure_benchmark_salary) / tenure_benchmark_salary,
        2
    ) AS tenure_comparison,
    -- title
    ROUND(title_benchmark_salary) AS title_benchmark_salary,
    ROUND(
        100 * (base.salary - title_benchmark_salary) / title_benchmark_salary,
        2
    ) AS title_comparison,
    -- department
    ROUND(department_benchmark_salary) AS department_benchmark_salary,
    ROUND(
        100 * (base.salary - department_benchmark_salary) / title_benchmark_salary,
        2
    ) AS department_comparison,
    -- gender
    ROUND(gender_benchmark_salary) AS gender_benchmark_salary,
    ROUND(
        100 * (base.salary - gender_benchmark_salary) / title_benchmark_salary,
        2
    ) AS gender_comparison,
    -- leave the effective/expiry dates at the end of the query
    base.effective_date,
    base.expiry_date
  FROM cte_ordered_events AS base
  INNER JOIN mv_employees.tenure_benchmark
    ON base.company_tenure_years = tenure_benchmark.company_tenure_years
  INNER JOIN mv_employees.title_benchmark
    ON base.title = title_benchmark.title
  INNER JOIN mv_employees.department_benchmark
    ON base.department = department_benchmark.department
  INNER JOIN mv_employees.gender_benchmark
    ON base.gender = gender_benchmark.gender
  -- apply filter to only keep the latest 5 events per employee
  -- WHERE event_order <= 5
)
SELECT * FROM final_output;

DROP VIEW IF EXISTS mv_employees.employee_deep_dive;
CREATE VIEW mv_employees.employee_deep_dive AS
SELECT * FROM mv_employees.historic_employee_records
WHERE event_order <= 5;
```
</details>

### 4.2. Current Company Snapshot Results
```sql
SELECT * FROM mv_employees.company_level_dashboard
ORDER BY gender;
```
Query Results:
| gender | employee\_count | employee\_percentage | company\_tenure | avg\_salary | avg\_salary\_percentage\_change | min\_salary | max\_salary | median | inter\_quartile\_range | stddev\_salary |
| ---------- | ------------------- | ------------------------ | ------------------- | --------------- | ----------------------------------- | --------------- | --------------- | ---------- | -------------------------- | ------------------ |
| **M**      | 144114              | 60                       | 14                  | 72045           | 3                                   | 38623           | 158220          | 69830      | 12804                      | 17363              |
| **F**      | 96010               | 40                       | 14                  | 71964           | 3                                   | 38936           | 152710          | 69764      | 12663                      | 17230              |
### 4.3. Department Level Results
```sql
SELECT * FROM mv_employees.department_level_dashboard
ORDER BY department, gender;
```
Result (first 5 rows):
| gender | department       | employee\_count | employee\_percentage | department\_tenure | avg\_salary | avg\_salary\_percentage\_change | min\_salary | max\_salary | median\_salary | inter\_quartile\_range | stddev\_salary |
| ------ | ---------------- | --------------- | -------------------- | ------------------ | ----------- | ------------------------------- | ----------- | ----------- | -------------- | ---------------------- | -------------- |
| M      | Customer Service | 10562           | 60                   | 9                  | 67203       | 3                               | 39373       | 143950      | 65100          | 20097                  | 15921          |
| F      | Customer Service | 7007            | 40                   | 9                  | 67409       | 3                               | 39812       | 144866      | 65198          | 20450                  | 15979          |
| M      | Development      | 36853           | 60                   | 11                 | 67713       | 3                               | 39036       | 140784      | 66526          | 19664                  | 14267          |
| F      | Development      | 24533           | 40                   | 11                 | 67576       | 3                               | 39469       | 144434      | 66355          | 19309                  | 14149          |
| M      | Finance          | 7423            | 60                   | 11                 | 78433       | 3                               | 39012       | 142395      | 77526          | 24078                  | 17242          |
### 4.4. Title Level Results
```sql
SELECT * FROM mv_employees.title_level_dashboard
ORDER BY title, gender;
```
Results (first 5 rows):
| gender | title              | employee\_count | employee\_percentage | title\_tenure | avg\_salary | avg\_salary\_percentage\_change | min\_salary | max\_salary | median\_salary | inter\_quartile\_range | stddev\_salary |
| ------ | ------------------ | --------------- | -------------------- | ------------- | ----------- | ------------------------------- | ----------- | ----------- | -------------- | ---------------------- | -------------- |
| M      | Assistant Engineer | 2148            | 60                   | 6             | 57198       | 4                               | 39827       | 117636      | 54384          | 14972                  | 11152          |
| F      | Assistant Engineer | 1440            | 40                   | 6             | 57496       | 4                               | 39469       | 106340      | 55234          | 14679                  | 10805          |
| M      | Engineer           | 18571           | 60                   | 6             | 59593       | 4                               | 38942       | 130939      | 56941          | 17311                  | 12416          |
| F      | Engineer           | 12412           | 40                   | 6             | 59617       | 4                               | 39519       | 115444      | 57220          | 17223                  | 12211          |
| M      | Manager            | 5               | 56                   | 9             | 79351       | 2                               | 56654       | 106491      | 72876          | 43242                  | 23615          |
### 4.5. Salary Benchmark Results
#### 4.5.1. Company Tenure Benchmark
```sql
SELECT * FROM mv_employees.tenure_benchmark
ORDER BY company_tenure_years;

```
#### 4.5.2. Gender Benchmark
```sql
SELECT * FROM mv_employees.gender_benchmark;
```
#### 4.5.3. Department Benchmark
```sql
SELECT * FROM mv_employees.title_benchmark
ORDER BY title_benchmark_salary DESC;
```
#### 4.5.4. Title Benchmark
```sql
SELECT * FROM mv_employees.department_benchmark
ORDER BY department_benchmark_salary DESC;
```

### 4.6. Historic Employee Deep Dive 
Deep dive example for a particular employee : Leah Anguita
```sql
SELECT * FROM mv_employees.employee_deep_dive
WHERE employee_name = 'Leah Anguita'
ORDER BY event_order;
```

| employee\_id | gender | birth\_date | employee\_age | hire\_date | title           | employee\_name | previous\_title | salary | previous\_latest\_salary | previous\_salary | department       | previous\_department | manager         | previous\_manager | company\_tenure\_years | title\_tenure\_years | title\_tenure\_months | department\_tenure\_years | event\_order | latest\_salary\_percentage\_change | event\_name     | salary\_percentage\_change | salary\_amount\_change | tenure\_benchmark\_salary | tenure\_comparison | title\_benchmark\_salary | title\_comparison | department\_benchmark\_salary | department\_comparison | gender\_benchmark\_salary | gender\_comparison | effective\_date | expiry\_date |
| ------------ | ------ | ----------- | ------------- | ---------- | --------------- | -------------- | --------------- | ------ | ------------------------ | ---------------- | ---------------- | -------------------- | --------------- | ----------------- | ---------------------- | -------------------- | --------------------- | ------------------------- | ------------ | ---------------------------------- | --------------- | -------------------------- | ---------------------- | ------------------------- | ------------------ | ------------------------ | ----------------- | ----------------------------- | ---------------------- | ------------------------- | ------------------ | --------------- | ------------ |
| 11669        | M      | 1975-03-03  | 47            | 2004-04-07 | Senior Engineer | Leah Anguita   | Engineer        | 47373  | 47046                    | 47373            | Customer Service | Customer Service     | Yuchang Weedman | Yuchang Weedman   | 18                     | 2                    | 7                     | 3                         | 1            | 0.70                               | Title Change    | 0.00                       | 0                      | 77411                     | \-38.80            | 70823                    | \-33.11           | 67285                         | \-28.12                | 72045                     | \-34.84            | 2020-05-12      | 9999-01-01   |
| 11669        | M      | 1975-03-03  | 47            | 2004-04-07 | Engineer        | Leah Anguita   | Engineer        | 47373  | 47046                    | 47046            | Customer Service | Customer Service     | Yuchang Weedman | Yuchang Weedman   | 18                     | 7                    | 7                     | 3                         | 2            |                                    | Salary Increase | 0.70                       | 327                    | 77411                     | \-38.80            | 59603                    | \-20.52           | 67285                         | \-33.41                | 72045                     | \-41.39            | 2020-05-11      | 2020-05-12   |
| 11669        | M      | 1975-03-03  | 47            | 2004-04-07 | Engineer        | Leah Anguita   | Engineer        | 47046  | 47046                    | 47046            | Customer Service | Production           | Yuchang Weedman | Oscar Ghazalie    | 18                     | 7                    | 7                     | 3                         | 3            |                                    | Dept Transfer   | 0.00                       | 0                      | 77411                     | \-39.23            | 59603                    | \-21.07           | 67285                         | \-33.96                | 72045                     | \-41.94            | 2019-06-12      | 2020-05-11   |
| 11669        | M      | 1975-03-03  | 47            | 2004-04-07 | Engineer        | Leah Anguita   | Engineer        | 47046  | 47046                    | 43681            | Production       | Production           | Oscar Ghazalie  | Oscar Ghazalie    | 18                     | 7                    | 7                     | 7                         | 4            |                                    | Salary Increase | 7.70                       | 3365                   | 77411                     | \-39.23            | 59603                    | \-21.07           | 67843                         | \-34.89                | 72045                     | \-41.94            | 2019-05-11      | 2019-06-12   |
| 11669        | M      | 1975-03-03  | 47            | 2004-04-07 | Engineer        | Leah Anguita   | Engineer        | 43681  | 47046                    | 43930            | Production       | Production           | Oscar Ghazalie  | Oscar Ghazalie    | 18                     | 7                    | 7                     | 7                         | 5            |                                    | Salary Decrease | \-0.57                     | \-249                  | 77411                     | \-43.57            | 59603                    | \-26.71           | 67843                         | \-40.54                | 72045                     | \-47.59            | 2018-05-11      | 2019-05-11   |
