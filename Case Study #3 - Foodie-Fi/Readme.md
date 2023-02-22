# 🍌 Case Study #3 - Foodie-Fi 

<p align="center">
  <img width="400" height="400" src="https://user-images.githubusercontent.com/115451301/220280761-9af49a31-6877-47cb-baf9-8c2d5aee78cb.png">
</p>

## **What I Learned?** 



## 📘 Table of Contents
- [Introduction](#introduction)
- [Problem Statement](#problem-statement)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Case Study Questions](#case-study-questions)

***

## **Introduction**

Danny finds a few smart friends to launch his new startup Foodie-Fi in 2020 and started selling monthly and annual subscriptions, 
giving their customers unlimited on-demand access to exclusive food videos from around the world!

Danny created Foodie-Fi with a data driven mindset and wanted to ensure all future investment decisions and new features were decided using data.

## **Problem Statement**

Danny has shared the data design for Foodie-Fi and also short descriptions on each of the database tables - our case study focuses on only 2 tables but there will be a challenge to create a new table for the Foodie-Fi team.
## **Entity Relationship Diagram**

<p align="center">
  <img width="600" height="300" src="https://user-images.githubusercontent.com/115451301/220281469-816d67ae-bb58-41c8-8c99-ed544f4ce07a.png">
</p>

## **Case Study Questions** 

### A. Customer Journey

View my solution [here](https://github.com/hieucabo/8-Week-SQL-Challenge/blob/main/Case%20Study%20%233%20-%20Foodie-Fi/A.%20Customer%20Journey.md).

Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.

### B. Data Analysis Questions

View my solution [here](https://github.com/hieucabo/8-Week-SQL-Challenge/blob/main/Case%20Study%20%233%20-%20Foodie-Fi/B.%20Data%20Analysis%20Questions.md).

1. How many customers has Foodie-Fi ever had?

2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value

3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name

4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?

6. What is the number and percentage of customer plans after their initial free trial?

7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

8. How many customers have upgraded to an annual plan in 2020?

9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)

11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

### C. Challenge Payment Question

View my solution [here](https://github.com/hieucabo/8-Week-SQL-Challenge/blob/main/Case%20Study%20%233%20-%20Foodie-Fi/C.%20Challenge%20Payment%20Question.md).

The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:

- monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
- upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
- upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
- once a customer churns they will no longer make payments
