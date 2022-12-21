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


