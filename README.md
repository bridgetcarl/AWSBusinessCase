# Cloud-Based Fulfillment App
### Bridget Carl, on behalf of Acme Supply Co.

## Description of the Problem
Acme Supply Co. is an e-commerce fulfillment company specializing in medical supplies and hardware. Customers rely on Acme for fast and accurate fulfillment of orders around the clock and around the nation. Acme Supply Co. aims to fulfill 3,000 daily orders from each of their four fulfillment centers within 30 minutes of a customer placing an order. Packages are shipped daily via parcel and freight carriers, with many customers receiving their orders the same or next day. As a result of this fast fulfillment process, Acme’s customers view Acme as the most reliable supplier for responding to their immediate supply needs. Maintaining a lean, fast, and technically resilient operation is vital to meeting customer commitments and maintaining Acme’s image in the medical supplies market. 

Given that Acme Supply Co. runs a very lean operation, strong day-to-day operation and capacity management is vital to meeting customer commitments. Today, Acme staffs a team of 5 individuals at each fulfillment center to manually track daily order queues and move employees to meet order volumes across the fulfillment operation. After a customer places an order through the website or via a conversation with a customer service representative, the order is placed in a internally maintained database. The fulfillment process runs from a system of files deployed internally that directs the order selection system to assign fulfillment work to individual employees based off where said individual is staffed in the operation. The fulfillment logic is deployed in an old, slow, and technically weak network of static files. The system is very vulnerable to technical limitations and computing speed. It is also not resilient in the long-run due to fast employee turnover and no internally staffed team of software engineers to maintain the system. While order data is not accessible by outside actors, all operations management employees can override system logic to force a fulfillment event through, leading to control and security vulnerabilities. This current system is not in line with Acme’s underlying goals of maintaining profitability and meeting customer commitments daily.

### Current System Architecture:

![Acme_current](https://github.com/user-attachments/assets/fac0bdb5-4871-4292-8ea2-5d581151c589)

Acme Supply Co. aims to deploy a cloud-based fulfillment app that handles gathering unfulfilled orders and assigns to fulfillment employees according to daily staffing. The app must securely parse order queues and assign fulfillment work to individuals based on daily staffing. It must allow for manual overrides in the case of an extenuating order or operational situation. The app must be able to track high-level operational performance data daily. The budget is $500,000 for the app start-up, and $350,000 yearly.

## Requirements 
Acme Supply Co. is seeking to deploy all fulfillment logic to a cloud-based app. The following are app components required by Acme for this development:

### Compute Power
The app must include several high-level components that are specific to the fulfillment operation. The app must store order queues, updated live as customers place orders throughout the day. Acme Supply Co. fulfills 12,000 – 15,000 orders daily across 4 fulfillment centers. First, the app must be able to sort orders in the order queue by carrier cut-off time to ensure that all customer shipping commitments are met. At 200 fulfillment employees at each center, on average, the app must be able to process 800 fulfillment “events” (assigning work to employee) simultaneously every 10 minutes. Fulfillment logic must ingest both live order queues and daily staffing data from each fulfillment center uniquely. Finally, the app must be able to compute daily operational performance metrics, including predicting if the fulfillments center will meet all customer commitments, measuring how productive each employee and department is at any given point, and indicating if there is a mismatch in where employees are staffed across the fulfillment center and from where orders are being fulfilled from.

### Compute Speed
The app must ingest data instantaneously, including live orders and daily operational staffing as provided by operational management. The app must be able to compute logic for the fulfillment event in less than 30 seconds for each event, every 10 minutes. This allows for the most recently placed orders to be included in the fulfillment and order queue logic. For additional operational performance metrics, the app must be able to process requests within 5 seconds of the request being placed. 

### Resiliency and Manual Overrides 
Any downtime of the app, and in turn the fulfillment process, results in real-world revenue loss. As such, technical resiliency of the app is vital. Additionally, operational management must be able to manually override the fulfillment logic and forcibly assign work if specific circumstances arise. 

### Security
Acme Supply Co. values information privacy and security very highly, and this app will contain data on daily order volumes (implying revenue), personal staffing data, and performance metrics. Security from outside actors is essential, but internal security is also vital. Only select members of operational management will be granted access to each of the app’s components separately. 

### Start-up Time
Acme Supply Co. expects this app to be operational and live in less than 6 months. App developers can expect regular check-ins and collaboration with Acme Supply Co. team members.


## Proposed Solution 
The following infrastructure is proposed to solve the system requirements as listed by Acme Supply Co.

### Order entry
There are ultimately two “events” involved in the app’s process. The first is the act of a customer placing an order and that order being added to a live order queue. Customers can place orders directly through Acme’s website, or a customer service representative can place an order on behalf of the customer. Customer service representatives are granted access to enter orders to the live order queue. Customers are required to enter log-in information or create an account to control for fraudulent behavior. For returning customers, a Lambda function is deployed to verify log-in information via a SQL database of customer log-in information. New customers can create an account, effectively writing to the database. A very limited audience of internal employees will have read-access to the customer information database to protect customer information privacy.

Data on the order products, quantities, shipping address, and other relevant customer information is collected via an AWS Lambda function acting as a point-of-sale service. A Lambda function is used over an EC2 instance since the point-of-sale functionality is not very complicated. Ultimately this saves on cost and system complexity. The point-of-sale Lambda function also results in an entry to a SQL database. This preserves customer order information, which is vital for accounting and inventory management purposes. No user will have write-access to the SQL database to avoid risk of fraudulent or accidental behavior by company employees. A limited audience of company employees are granted read-access to the SQL database. Order history implies operating revenue and profitability, as well as divulges customer behaviors and purchases. Acme Supply Co. is a private corporation with loyal and long-standing customers. Information security at this step is vital.

### Sorting Live Order Queue
Orders are placed either directly by a customer on the website or by a customer service representative placing the order on their behalf. In both scenarios, the order is placed in a live order queue to be fulfilled first-in-first-out and depending on carrier cutoff times. A S3 bucket serves as the live order queue. The Lambda function results in an entry in the S3 bucket for each order placed through the point-of-sale component. S3 buckets allow for rapid changes to the order queue and fast data entry, again while saving cost and system complexity. A Lambda function sorts orders in the queue based on entry time and shipping carrier cutoffs. Orders delivered same day will be fulfilled before later delivery estimates. The Lambda function dumps the updated queue into a second S3 bucket to be worked.

### Order Fulfillment
The next “event” in the app’s operation is the order fulfillment. This level of compute power is higher than in other steps of the app, so an EC2 instance is deployed to manage all fulfillment logic. The EC2 instance ingests data from two sources. First, the instance reads the updated (sorted) order queue from a S3 bucket. Every 10 minutes the app will process the fulfillment logic. The second input is from daily operational staffing data. The operational management will link a file to the app at the start of each fulfillment shift that indicates which employees are working that that and, more importantly, where in the operation they are working. This is important because the operation staffs individuals of different skillsets and responsibilities. Also, the end user is the fulfillment employee. This data will indicate who the app must assign work to. Daily operational staffing will be linked via an Excel file and can be updated throughout the day as employees come and go, so the EC2 instance must re-read the file before each fulfillment event occurs (every 10 minutes). 

A fulfillment cycle lasts 10 minutes. At 9 minutes and 30 seconds into the cycle, the EC2 instance will ingest staffing data for the most up-to-date information on employee location. The EC2 instance will then determine how many orders can be selected from the queue based on staffing data. Within 30 seconds, the app will have a set of orders that can be worked. The final step in the app’s process is to assign work to the fulfillment employee, the end user. The EC2 instance delivers work to each fulfillment employee via an AWS Direct Connect. Each employee is issued a tablet that is connected to the EC2 instance directly. When the employee completes each piece of work assigned, their personal tablet will send that information back to the EC2 instance for tracking and performance analytics. Work is completed and the cycle begins again.

### Operational Analytics
An additional functionality required by the app is live operational analytics. This will include daily order volumes, remaining orders in the live queue, work that has not been completed by the fulfillment employee, and which areas of the operation may be understaffed given the current order queue. This component of the app will be handled by AWS QuickSight, a dedicated analytics component. QuickSight will connect to the app via an API. Only operational staff will have read-access to the analytics provided by QuickSight.

### Proposed System Architecture:

![Acme_proposed](https://github.com/user-attachments/assets/0979d8f9-ea15-416f-ba97-7b3c3d09b90c)


## Budget 
Acme Supply Co. plans to test this implementation at a single fulfillment center before rolling out to all centers. A successful implementation of the cloud-based fulfillment app will save Acme Supply Co. roughly 5 full time employees at each fulfillment center yearly, averaging $500,000 in yearly wage and benefits. As such, Acme Supply Co. is willing to withstand significant up-front and monthly costs to see cost savings in the long run. The consulting and app development budget then is $500,000 for the team and development, over a 6 month period of consultation. The team will consist of 2-3 dedicated developers, who will also serve as consultants to the project team at Acme Supply Co. The team can expect monthly check-ins with Acme Supply Co. and ad-hoc consultations as needed.

As for the app features and cost to AWS, Acme Supply Co. is willing to withstand up to $35,000 monthly in the long-run. The cost estimate for the features described in the proposed architecture is $26,000 per month, allowing for a buffer for unforeseen charges and services and for added profitability to the company. The estimated yearly cost is $313,000. 

In total, Acme Supply Co. is expecting to pay $800,000 in 2024 for the development of a cloud-based order fulfillment app. With an estimated yearly cost of $313,000, the company expects to see a return on investment within 2 years. Along with the benefit of a stronger, faster, and more accurate fulfillment operation, Acme Supply Co. expects to see major profit wins with this development. 

## Implementation

### Internet gateway:

<img width="783" height="374" alt="image" src="https://github.com/user-attachments/assets/8679104a-a41b-490b-9b61-561f758dbbad" />

### Log-in verification SQL connected to Authenticate Lambda:

<img width="774" height="347" alt="image" src="https://github.com/user-attachments/assets/e0246790-f506-4e1d-aa35-1c9d5fc870ab" />

<img width="779" height="337" alt="image" src="https://github.com/user-attachments/assets/754335bf-45c1-4097-aa87-52c434bcda10" />

### Lambda point-of-sale connected to Order History SQL and Order Queue 1 S3:

<img width="787" height="277" alt="image" src="https://github.com/user-attachments/assets/62c0b792-fb01-4066-adfe-5fde62ea987c" />

<img width="784" height="351" alt="image" src="https://github.com/user-attachments/assets/c09598bb-cc86-4c45-9e80-86c52ae9b231" />

<img width="794" height="426" alt="image" src="https://github.com/user-attachments/assets/cd69457c-06ea-4c95-9f3d-a41540bbc864" />

### Sort Queue Lambda connected to Sorted Order Queue 2 S3:

<img width="771" height="300" alt="image" src="https://github.com/user-attachments/assets/1fe2d9fb-18fd-47a7-b85c-f2179d864ff0" />

### Fulfillment EC2:

<img width="786" height="405" alt="image" src="https://github.com/user-attachments/assets/90c6c462-fd1d-4684-8770-bc0c224e021d" />

### Assign work DirectConnect:

<img width="819" height="398" alt="image" src="https://github.com/user-attachments/assets/75ca947f-ceee-4675-b6cc-1b7d49d2a176" />

### Analytics dashboard QuickSights:

<img width="819" height="327" alt="image" src="https://github.com/user-attachments/assets/eddca0e1-57da-4fdb-9297-2dbc4f67327c" />


