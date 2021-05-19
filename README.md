# n-booking

[![Go Report Card](https://goreportcard.com/badge/github.com/nimesh-mittal/n_booking)](https://goreportcard.com/report/github.com/nimesh-mittal/n_booking)

## Background

Aim of this service is to provide ability to book products available in the inventory. This service using multiple other services to complete the booking system. Some of the primary services are:

1. Catalog service - holds product information
2. User service - holds customer and seller information
3. Pricing service - calculates selling price based on MRP, service charges, taxes and offers
4. Offer service - help manage various offers and their evaluation
5. Billing service - manages ledger and help settling payments, refunds
6. Payment service - take cares of integration with payment gateways, payment status and refunds
7. Shippment service - manages shipment of product
8. Inventory service - manages inventory of products
9. Communication service - manages communication by sending email, sms and push notification to customers and sellers

The primary responsibility of n_booking service is to take order and execute them by updating inventory. User check various products details, price and its availability in a particular location by consulting Catalog service, Pricing service and Inventory service. If user likes a product and agree with price and availability it adds the product to cart. When user agree to buy the product it initiates payments by consulting Payment service and Billing service. On successful completion of payment user receives product with the help of Shippment service, Inventory service and Communication service.

In summary Booking service takes following responsibility

1. Create order
2. Update order on payment initiation and lock inventory
3. Update order on payment completion and reserve inventory
4. Update order on shipment and initiate invoice
5. Send communication on every order update

## Requirments

Booking service should provide following abilities

- Ability to create and cancel order
- Ability to update order status
- Ability to list orders

## Service SLA

- Availability
Booking service should target 99.99% uptime
- Latency
Booking service should aim for less than 100 ms P95 latency
- Throughput
Booking service should provide QPS of 2000 per node
- Freshness
Booking service should provide newly created orders in search results immediately

## Architecture

![image](https://github.com/nimesh-mittal/n_booking/blob/main/.github/images/n_booking.png)

## Implementation

### API Interface

```go

type Interface{
  // create new order
  Create(order Order, tenant string) (string, error)
  // cancel exisiting order
  Delete(orderID uuid, tenant string) (bool, error)
  // update order status
  Update(orderID uuid, fieldsToUpdate map[string]interface{}, filter map[string]interface{}, tenant string) (bool, error)
  // search orders by keyword
   Search(query string, limit int, offset int, sortBy string, tenant string) ([]Order, error)
}

```

## Data Model

| Table Name | Description | Columns |
| ------- | ---- | ---- |
| order | Represents order | (tenant_name, order_id, order_type, attributes..., who...)
| order_items | Represents order items| (tenant_name, order_id, item_id, attributes..., who...)

where attributes for order are:

TenantID string
OrderID string
UserID string
Amount float64
BillingAddress string
ShipmentAddress string
Status string
OrderDate time.Time
DeliveryInstructions string
PaymentInstructions string
PaymentID string

where attributes for order_items are:

TenantID string
OrderID string
ItemID string
ItemQuantity float64
ItemPrice float64

Who columns are:

Active bool
CreatedBy string
CreatedAt int64
UpdatedBy string
UpdatedAt int64
DeletedBy string
DeletedAt int64

## Database choice

A close look at the API interface reveals equal amount to read and write requests. While data consistency is required, need for strong transaction semantics is not required. Given order status needs to be highly consistent, any eventual consistent database like mongo db can be rolledout.

For high write throughput horizontally scalable database is prefered, but given data can be easily sharded by users, we can use relational database like Postgre.

Postgre provide high consistency and transactional semantics that is required when multiple services trying to see consistent view of order status.

## Scalability and Fault tolerance

Inorder to survive host failure, multiple instances of the service needs to be deployed behind load balancer. Load balance should detect host failure and transfer any incoming request to only healthy node. One choice is to use ngnix server to perform load balancing.

Given one instance of service can serve not more than 2000 request per second, one must deploy more than one instance to achive required throughput from the system

Load balancer should also rate limiting incoming requests to avoid single user/client penalising all other user/client due to heavy load.

## Functional and Load testing

Service should implement good code coverage and write functional and load test cases to maintain high engineering quality standards.

## Logging, Alerting and Monitoring

Service should also expose health end-point to accuretly monitor the health of the service.

Service should also integrate with alerting framework like new relic to accuretly alert developers on unexpected failured and downtime

Service should also integrate with logging library like zap and distributed tracing library like Jager for easy debugging

## Security

Service should validated the request using Oauth token to ensure request is coming from authentic source

## Documentation

This README file provides complete documentation. Link to any other documentation will be provided in the Reference section of this document.

## Local Development Setup

### Setup environment variables:

```
export AWS_REGION="ap-south-1"
export AWS_SECRET_ID=""
export AWS_SECRET=""
export POSTGRE_URL_VALUE="<url in the form postgres://userid:password@host:port/databasename>"
```

### Start service:

```go run main.go```

### Run testcases with coverage:

```go test ./... -cover```

### How to generate mocks in local

- go get github.com/golang/mock/gomock

- go get github.com/golang/mock/mockgen

- <path to bin>/mockgen -destination=mocks/mock_orderrepo.go -package=mocks n_booking/repo OrderRepo
