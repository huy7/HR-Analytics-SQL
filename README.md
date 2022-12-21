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
- All of the metrics as per the company level but at a department level, including an additional department level tenure metrics split by gender.

#### 2.1.3. Title Level
- Similar to the department level metrics but instead, at a title level of granularity.

- We will also need to generate similar insights for both department and title levels where we need employee count and average tenure for each level instead of company wide numbers.

### 1.2. Employee Deep Dive
The following insights must be generated for the Employee Deep Dive tool that can spotlight recent events for a single employee over time:

- See all the various employment history ordered by effective date including salary, department, manager and title changes
- Calculate previous historic payrise percentages and value changes
- Calculate the previous position and department history in months with start and end dates
- Compare an employee’s current salary, total company tenure, department, position and gender to the average benchmarks for their current position

### 1.3. Visual Outputs

#### Current Snapshot Reporting
___
<img src=""/>

#### Historic Employee Deep Dive
___
<img src=""/>


## 2. Exploration
