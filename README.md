# Microservices

## Types if Codebases

### Monolith Approach

A monolithic approach means that our codebase contains all of the following to build **ALL** features

* Routing
* Middlewares
* Business Logic
* Database Access


### Microservice Approach

A single microservice implementation contains all of the following features to **ONE** feature

* Routing
* Middleware
* Business Logic
* Database Access

## Issues with Microservices

### Data management between services

* How we store it
* How we access it

Each service has it's own database (if it needs one)
A service will never reach into another service's database

* We want every service to run independently from other services
  - If all the services shared the same DB and something went wrong with the DB, then ALL of our services will crash immediately
  - If service A goes into service B's DB but it's down, service A and B wont work.
* Database's schema/structure might change unexpectedly
  - If services share DBs, changing the schema might break another service
* Some services might function more efficiently with different types of DBs


### Simple Example

Features

* Signup
* List Products
* Buy Products

#### Monolithic Approach

Server includes:

* Code to Signup user
* Code to list available products
* Code to purchase a product

Database includes:

* User Collection
* Products Collection
* Orders Collection

New Feature:

* Code to show products ordered by a particular user

In a monolith setup you would have to

1. Lookup User in the Users Collection
2. Look at the Orders Collection for orders from that user
3. Look at the products to populate the products from that user

#### Microservice Approach

3 services

* User Service
  - Code to signup user
  - User Collection
* Products Service
  - Code to list available Products
  - Products Collection
* Orders Service
  - Code to purchase a product
  - Orders Collection

New Feature/Service:

* Ordered Products Service - Code to show products ordered by a particular user

#### Communication Strategies Between Services

* Sync - Services communicate with each other using direct requests
* Async - Services communicate with each other using events

##### Sync Example

Below the new service reaches to the services (not to the databases) to get the information.

1. Request User from User Service
2. If the User exists, make a request to the Orders Service
3. If there are Orders, make a request to the Products Service to get products

Pros:

* Easy to understand
* New service won't need a database

Cons:

* Introduces dependencies between services
* If any inter-service request fails, the overall request fails
* The entire request is only as fast as the slowest request
* Can easily introduce webs of requests

##### Async Example

**Async Example 1**

* Event Bus/Event Broker - Handles different notifications/events from our services

1. Ordered Products Service emits an event to the Event Bus `{ type: UserQuery, data: { id: 1 }}`
2. The event bus determines that the User Service needs to handle this event and passes the event
3. The User Service registers the event, gets the data, and creates a new event `{ type: UserQueryResult, data: { id: 1, name: 'David' }}` and sends it back to the Event Bus
4. The Event Bus routes that event back to the Ordered Products Service, which processes the data, then sends a new event to the Event Bus `{ type: OrderQuery, data: { id: 2 } }`
5. ... it continues passing events until it's done getting all of the information

Unfortunately async has all the pros and cons of sync:

Pros:

* Easy to understand
* New service won't need a database

Cons:

* Introduces dependencies between services
* If any inter-service request fails, the overall request fails
* The entire request is only as fast as the slowest request
* Can easily introduce webs of requests

**Async Example 2**

* Event Bus/Event Broker - Handles different notifications/events from our services

**What we actually want to do:**

Given the ID of a user, show the title and image of every product they have ever ordered.

We want to create a database for this new service, it would hold:

* Users Table with only `id` and `product_ids`
* Product Table with only `id`, `title`, and `image`

How it would work:

* Every time a new User is created, the User Service would:
  - Create a line item in it's own database about that users
  - Emit an event with that information and forward it to the Event Bus, the event might look something like this `{ type: 'UserCreated', data: { id: 1, name: 'David' }}`
  - The Ordered Products Service would receive this event and add a line item to it's own User Table
* Every time a new Product is created, the Product Service would:
  - Create a line item in it's own database about that product
  - Emit an event with that information and forward it to the Event Bus, the event might look something like this `{ type: 'ProductCreated', data: { id: 1, title: 'pants', image: 'pants.jpg' }}`
  - The Ordered Products Service would receive this event and add a new product to it's own Product Table.
* Every time a new Order is created, the Orders Service would:
  - Create a line item in it's own database about that order
  - Emit an event with that information and forward it to the Event Bus, the event might look something like this `{ type: 'OrderCreated', data: { userId: 1, cost: 1000, productIds: [1, 2, 3] }}`
  - The Ordered Product Service would receive this event and update the corresponding User's line item with the new productIds.
