<div align="center">
<img src="https://github.com/melloware/temporal-purchase-order/blob/main/docker-compose/quarkus-petstore-logo.png" width="400" height="294" >
  
# Quarkus Temporal Petstore
</div>

Quarkus Temporal Petstore is a comprehensive demonstration using Quarkus and Temporal. It simulates placing a new order on your Petstore website and fulfilling it using a microservice architecture.

While the code is surprisingly simple, under the hood this is using:

- [Quarkus](https://quarkus.io/) for super subatomic microservices
- [Temporal](https://www.temporal.io/) for indestructible microservices and workflow orchestration
- [Mailpit](https://mailpit.axllent.org/) email & SMTP testing tool
- [PostgresSQL](https://www.postgresql.org/) the world's most advanced open source relational database

## Overview

The following workflow is orchestrated across these microservices.
* Purchase Order Gateway service receives a new order through a REST service
* Email is sent to the customer notifying them the order has been received and is processing
* Order is persisted to the database
* Credit Card is verified as valid with enough funds
* Warehouse checks if there is enough inventory to fulfill this order
* Shipment service registers the shipment and creates a tracking number
* Order is marked as complete in the database with updated information
* Email is sent to the customer notifying them order is on the way with tracking number

If anything in this process fails a "compensating transaction" must occur with the following steps:
* Payment service must reverse the payment or release the credit card hold
* Order service must update the database with the failure information
* Email is sent to the customer notifying them something went wrong and to call customer service with a reference number.

Temporal handles retrying and waiting for your services to come back up.  So it will track the workflow until it is completed or issue a failure if the whole workflow is not completed in time (by default 24 hours).

## Microservices

The following microservices have been created to simulate this system:

| Service                | Description                                                                               | Port | Dev UI URL                                                         |
|------------------------|-------------------------------------------------------------------------------------------|------|--------------------------------------------------------------------|
| Purchase Order Gateway | Workflow gateway service that starts and manages the workflow when the order is received. | 8082 | [http://localhost:8082/q/dev-ui/](http://localhost:8082/q/dev-ui/) |
| Notification Service   | Email notifications to customers about order progress, completion, or cancellation.       | 8089 | [http://localhost:8089/q/dev-ui/](http://localhost:8089/q/dev-ui/) |
| Order Service          | Tracking the order in the database.                                                       | 8090 | [http://localhost:8090/q/dev-ui/](http://localhost:8090/q/dev-ui/) |
| Payment Service        | Validating the payment such as credit card.                                               | 8086 | [http://localhost:8086/q/dev-ui/](http://localhost:8086/q/dev-ui/) |
| Warehouse Service      | Validating inventory availability and decrementing inventory.                             | 8989 | [http://localhost:8989/q/dev-ui/](http://localhost:8989/q/dev-ui/) |
| Shipment Service       | Registering the order with a shipment provider and getting a tracking number.             | 8989 | [http://localhost:8877/q/dev-ui/](http://localhost:8877/q/dev-ui/) |

## Docker Compose

To run this demo you will need Docker or Podman to run the compose stack.  To run it:

```shell
$ cd docker-compose
$ docker-compose up -d
```

The above will start the following services:

| Service    | Description                                                | Port / URL                                       |
|------------|------------------------------------------------------------|--------------------------------------------------|
| Temporal   | Temporal server and UI                                     | [http://localhost:8080](http://localhost:8080)   |
| Mailpit    | SMTP Mail collector to view emails generated by the system | [http://localhost:8025](http://localhost:8025)   |
| pgAdmin    | Administrative UI for PostgreSQL                           | [http://localhost:16543](http://localhost:16543) |
| PostgreSQL | Database for both the orders and for Temporal storage      | 5432                                             |

## Running Workflow

You will need to have all 6 services running using `mvn quarkus:dev`, I recommend running them each in their own command prompt window for easy testing and startup/shutdown.

To start the workflow go to [Gateway Service DevUI](http://localhost:8082/q/dev-ui/) and find the OpenAPI card.  There you can use the OpenAPI UI to call the REST Service with the following example payload (make changes as you like):

```js
{
  "creditCard": {
    "cardNumber": "67915919403732",
    "cardHolderName": "Homer Simpson",
    "expiryDate": "04/09",
    "cvv": "5091",
    "type": "VISA"
  },
  "customerEmail": "homer.simpson@springfield.gov",
  "products": [
    {
      "sku": "8675309",
      "quantity": 1,
      "price": 50
    }
  ]
}
```

You should see the new Workflow in the Temporal UI as well as both an Order Received and Order Completed emails in the Mailpit UI.
