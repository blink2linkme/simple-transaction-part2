# Simple Transaction: Microservices sample architecture for .Net Core Application - Part 2

### A simple way to write a queue based messaging solution in .Net core to solve complex problems

* [Introduction](#Introduction)
* [Problem Description](#Problem-Description)
* [Application Architecture](#Application-Architecture) 
* [Solution Design](#Solution-Design)
* [Development Environment](#Development-Environment)
* [Opensource Tools / Technologies](#Opensource-Tools-Technologies)
* [WebApi Endpoints](#WebApi-Endpoints)
* [Solution Structure](#Solution-Structure)
* [Interaction between the components](#Interaction-between-the-components)
* [Scheduled background task with Cron expression](#Scheduled-background-task-with-Cron-expression)
* [Communication between the microservices](#Communication-between-the-microservices)
* [How to run the application](#How-to-run-the-application)
* [Console App : Gateway Client](#Console-App-Gateway-Client)
* [Final Thoughts](Final-Thoughts)

### Related Article

[Simple Transaction: Microservices Sample Architecture for .NET Core Application](https://www.codeproject.com/Articles/1277140/Simple-Transaction)

## Introduction 

This is Part 2 of [Simple Transaction: Sample .Net Core application](https://www.codeproject.com/Articles/1277140/Simple-Transaction), a continuation to demonstrate how to build a command-driven / Messaging based solution using .Net Core. The sample application which was used in Part 1 is extended with few additional microservices.

The original sample in [Part 1](https://www.codeproject.com/Articles/1277140/Simple-Transaction) implements microservices for a simple automated banking feature like Balance, Deposit, Withdraw in ASP.NET Core Web API with C#.Net, Entity Framework and SQL Server. This Part 2 sample is about a feature to generate monthly account statement through background service and store it in No-SQL Db (MongoDB and Redis Cache) which is then accessed through a seperate microservice.

## Problem Description

Generating reports like account statements for the users on the fly from the transactional data avaialble in the sql server would cause performance issue in a real time system due to the huge amount of data. 

A better way to handle this complex problem is to seperate the statement generation process (Write / Command) and accessing (Read / Query) the data.  This sample will show you how to seperate the command  reponsibility through background service and Query data through a seperate microservice which solves the performance problem and makes the system capable of handling large data by allowing us to scale the services independently depending on the load.

## Application Architecture

The below shown is the updated version of the architecture diagram. The highlighted part is the newly added microservice to access the account statement. This "statement" service uses MongoDb as persistent store and Redis Cache as temporary store where the data is loaded into cache from MongoDb for a period of time and it's retrived from cache on subsequent access.  

![](/images/architecture.png)



## Solution Design

The below diagram shows the design of separating the Command and Query responsibility. It has three core components that seperates the write operation using the Command / Message processing and read operation using Query data. 

* Publisher API / Scheduler Service
* Receiver Service
* Statement API

![](/images/solution-design.png)

#### Publisher API / Scheduler Service 

This microservice publishes a command into the message bus (Message Queue) when monthly account statement need to be generated for a given month. This service has a automated scheduler to trigger the statement generation process on a scheduled time and also exposes WebApi endpoints to manually trigger the process.

#### Receiver Service

Receiver service is a listener,  listens to message queue and consumes the incoming message and process it. Receiver  communicates with other depedent service to get the data and generates user's monthly account statement for a given month and stores the document in MongoDB.

#### Statement API

This microservice exposes a WepApi endpoint that communicates with the Redis Cache and MongoDb through data repository to access the account statement.

## Development Environment

- [.Net Core 2.2 SDK](https://dotnet.microsoft.com/download)
- [Visual Studio .Net 2017](https://visualstudio.microsoft.com/downloads/)

## Opensource Tools, Technologies

- Mongo DB as Persistent Store
- Redis Cache as Provisional Store
- RabbitMQ as Message Broker
- EasyNetQ as RabbitMQ Client
- NCrontab for time-based scheduling
- .Net Core IHostedService as background service

## WebApi Endpoints

#### End-points implemented in "Publisher API" to publish command in to message queue

1. Route: **"api/publish/statement"** [HttpPost] - To publish a message in to queue to trigger the background process. Defaults to previous month.
2. Route: **"api/publish/statement/{month}"** [HttpPost] - To publish a message in to queue to trigger the background process.
3. Route: **"api/publish/statement/{month}/accountnumbers"** [HttpPost] - To publish a message in to queue to trigger the background process for the given list of account numbers.

#### End-point implemented in "Statement API" to access account statement

1. Route: **"api/statement/{month}"** [HttpPost] - To access the account statement of authenticated user for a given month

#### End-point configured and accessible through API Gateway to access account statement

1. Route: **"statement/{month}"** [HttpPost] - To access the account statement of authenticated user.

## Solution Structure

The highlighted part in the below diagram shows the new services / components added to the sample.

![](/images/solution-structure.png)


## Interaction between the components

The below diagram shows the interaction between the components / objects in the background "Receiver" service.

![](/images/object-interaction.png)


## Scheduled background task with Cron expression

A cron expression is a format to specify a time based schedule to execute a task. NCrontab is the Nuget package included in the "Publisher.Service" to parse the expression and determine the next run. The service is configured to trigger the background process at the start of every month to generate the account statement for the previous month.

![](/images/cron-schedule.png)

Reference: [Run scheduled background tasks in ASP.NET Core](https://thinkrethink.net/2018/05/31/run-scheduled-background-tasks-in-asp-net-core/)

## Communication between the microservices

There's a need for the microservices to communicate with each other to let other services know when changes happen or to get dependent information. Here, in our sample , both Synchronous and Asynchronous mechanism are used to establish communication between microservices.

Message based asynchronous communication is used to trigger a event. It's a point-to-point communication with a single receiver. This means when a message is published to the queue, there's a single receiver that consumes the message for processing. The below diagram depicts the communication between "Publisher / Scheduler" and "Receiver" 

![](/images/asynchronous-communication.png)

The background service "Receiver" communicates with dependent services (Identity and Transaction) through Http request to get the actual data for background processing.

![](/images/synchronous-communication.png)


## How to run the application

Follow the same instruction that was given in the [previous article](https://www.codeproject.com/Articles/1277140/Simple-Transaction) to run the application. In addition to that, the newly added services (Publisher, Receiver and Statement API) should also be up and running. 

You can update the MongoDB and Redis Cache connection string in the appsettings.json file of "Receiver.Service" and "Statement.Service" if required.

For the database changes, refer to database project included in this version of code which has the updated script. 

## Console App : Gateway Client

![](/images/statement-result.png)

## Final Thoughts

Not necessary that you need to have two no-sql db for a single microservice to improve the performance of a system. In this case, just the mongoDb should do the need but included redis cache to show an alternate way of acheiving better performance for frequently accessed data.