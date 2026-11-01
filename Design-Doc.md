# Design Doc 



## Overview 
Rodney's Bike Shop system allows clients to reserve a bike(s) as well the client should be able to pay for it online. The staff should be able to list bikes for reservation, set prices, and put bikes into maintence as needed. 

## Actors 
* **Customer**: Place Order, Insert Payment, Take receipt 
* **Employee**: Check logs, Cancel-Orders, Update-Bookings, Excute Refunds, Remove Bikes, Add Bikes
* **Kiosk**: Take-Payment, Give-Change, Provide Receipt, Scroll, Add to Cart, Remove Bike
* **Online Website** Take-payment, provide receipt, scroll, add to cart, remove bike

## Goals/Non-Goals 

### Goals
* No account needed 
* Log all orders
* Alert staff automically 
* Print receipt 
* Charge correct prices  

### Non-Goals 
* No automatic discounts for now.
* Selling bikes is a diffrent system 
* No current inventory system, it's on paper
* No auto system for overbooking, from damage and late rental returns. 
* No auto system for stock limitatiosn 
* No system preventing rentals based on crazy weather clients could rent if they want  
* Replacement charge is human intervaention not auto
* Ad system handled by other team


## Constraints and Assumptions

### Constraints 
* There are 60 bikes, so about 60 rentals per day is the peak. This is not a high-concurrency system
* Rentals are capped at 7 days, in 4-hour intervals, and a day's fee equals 12 hours
* The card is pre-authorized at reservation and charged after return. Pre-authorization, charges, and refunds all use the processor's API. 
* Role base permissons for ownerm manager, employee
* All bike pickups, returns, extensions, size swaps, and damage assessments go through an employee during business hours. There is no self-service or after-hours pickup.

### Assumptions
* A bike out of service will be avaiable the next day, so let it be rented out for anyday after the day of service
* Most returned bikes are ready to be rented again after 4 hours, allow customers to book 4 hours after the expected return time.
* Card failure are caught 
* Everyone will use the dollar so no need for conversion rates/software/management 
* Abuse like mass no show reservations is low enough risk so don't design against it

## Non-Functional Requirements

| Requirement | Measure |
|---|---|
| Printed Receipt  | 100% of the time under 5 seconds |
| Order Sent  | Staff recieve order for bike from kiosk under 10 seconds |
| Payment Confirmation | 95% of the time payment confirmation is confirmed in under 9.99 seconds |
| System Restart | If system needs restart, system returns to normal state under 3 minutes |
| Wrong Payment Info | 0 Instances where wrong payment info is able to place a order |

## Component Diagram and Description

```mermaid
flowchart LR
    Customer([Customer]) --> UI[Screen & Keypad UI] --> SC
    Customer --> CR[Card Reader]
    CR --> Kiosk([Kiosk]) 
    Employee([Employee])
    SC[Session Controller] --> UI
    SC --> CR
    SC --> Bike-Catalog
    SC --> BG[Bank Gateway] --> Payment-Handler  --> SC
    SC --> RP[Receipt Printer]
    SC --> TL[(Transaction Log)]
    SC --> Order-Sent --> Employee-Dashboard 
    Employee --> Update-Inventory --> Kiosk
    Kiosk --> SC
    Employee-Dashboard --> Employee
```

| Component | Responsibility |
|---|---|
|Receipt Printer  | Prints receipts  |
| Card Reader | Read card data |
| Kisosk | Terminal where customers can pick the bike and rent it for the day, as well pay |
| Session Controller | high level software ran on the kisok to provide information about the machine and the functionally needed from other componets |
| Bank Gateway | Connects the users information to the payment processor's API|
| Transaction Log| Contains logs all of transactions occuring on the terminal|
| Employee Dashboard | Pulls the order from session controller and lets employee see into the order|
| Screen & keypad UI | Allows customer to input pin and select the bike of their choosing|
| Update Inventory | Pulls from the employee dashboard and informs the kisok of any new changes made to the website or stock of bikes for rent|


## Data Model


### Entity Relationship Diagram
``` mermaid 
erDiagram
    direction TB
    Customer {
        int customer_id PK
        string name
        string email
        int transaction_id FK "References OnlineWebsite.transaction_id"
        int transaction_id FK "References Kisosk.transaction_id"

    }

    Kisosk {
        int customer_id FK "References Customer.customer_id"
        string name
        int kisok_num PK
        string email
        int transaction_id 
    }

    Employee {
        int employee_id PK
        bool CheckedBike 
    }

    OnlineWebsite {
        int customer_id FK "References Customer.customer_id"
        string name
        string email
        int ip_address PK
        int transaction_id 
    }

    RentedBike {
        int customer_id FK "References Customer.customer_id"
        string name
        bool CheckedBike FK "References Employee.CheckedBike"

    }
    Employee ||..o{ RentedBike : "referenced by"

    Customer ||..o{ Kisosk : "referenced by"
    Customer ||..o{ OnlineWebsite : "referenced by"
    Customer ||..o{ RentedBike : "referenced by"
```




### Class Diagram
``` mermaid
classDiagram
    direction TB

    class Customer {
        int customer_id
        string name
        string email
        int website_transaction_id
        int kiosk_transaction_id
    }

    class Kisosk {
        int customer_id
        string name
        int kisok_num
        string email
        int transaction_id
    }

    class Employee {
        int employee_id
        bool CheckedBike
    }

    class OnlineWebsite {
        int customer_id
        string name
        string email
        int ip_address
        int transaction_id
    }

    class RentedBike {
        int customer_id
        string name
        bool CheckedBike
    }

    Employee "1" -- "0..*" RentedBike : checks
    Customer "1" -- "0..*" Kisosk : uses
    Customer "1" -- "0..*" OnlineWebsite : uses
    Customer "1" -- "0..*" RentedBike : rents
```
## Interfaces
``` python
printReceipt(transactionID: TransID, Customer_name: Name,) -> printed
```

| Row | Meaning |
|---|---|
| **Purpose** | Prints the receipt  |
| **Requires** | Must have a transaction ID and a name for the custome  |
| **Outcomes** | Approved : Prints the receipt Declined : No ID or No Name does not print|
| **Guarantees** | The transaction ID must be vaild if it prints or it doesn't |
| **Notes** | This is not required if the client does not want the receipt  |

``` python
logTransaction(transactionID: TransID, Customer_name: Name,) -> Transaction_Log_DB
```

| Row | Meaning |
|---|---|
| **Purpose** | Sends the client information to the transcation database|
| **Requires** | Transaction confirmed flag must be set for the account/transaction |
| **Outcomes** | Approved : The transaction is logged and sent to staff Declined : Invaild information try again |
| **Guarantees** | A temp log is put in place in the databse and updated with the correct information if the log works |
| **Notes** | If the call to log can't log within 1 minute staff is flagged |

``` python
cardReader(transactionID: TransID, card: Card, pin: Pin,  Customer_name: Name,) -> sentToBankGateway
```

| Row | Meaning |
|---|---|
| **Purpose** | Reads the card |
| **Requires** | The machine needs to have a vaild transcation ID before paymen |
| **Outcomes** | Approved : The customer is charged and payment is confirmed  Declined : Invaild information try again  OR Payment failed try again? |
| **Guarantees** | Pin is not saved |
| **Notes** | Pin not saved, information about the card wiped as soon as possible |

``` python
bikeQuery(transactionID: TransID, Customer_name: Name, bikeModels: Model, price: Price) -> Requested Bike Page
```

| Row | Meaning |
|---|---|
| **Purpose** | Searches the list for a bike model |
| **Requires** | Bike catalog must be up to date|
| **Outcomes** |Approved : The page shows the requested model Declined : Retry the search once ? |
| **Guarantees** | The bike catalog is up to date before the search calls   |
| **Notes** | Calls update bike catalog and auto retries once |


## Invariants 
- Bike is only given when the transaction is confirmed 
- No bike has two overlapping active reservations.
- Every captured payment has exactly one transaction-log entry.
- Customers pins are secured and always wiped
- A bike in maintenance cannot be reserved for that window.
-  Kisosk can accommodate any type of customer needs

## Failure Modes

| # | Failure | Detection | Response | Customer Sees |
|---|---|---|---|---|
| 1 | Card unreadable or unsupported | `awaitCard` returns **Unreadable** or **Unsupported** | Eject card. No transaction is created. | Card could not be read |
| 2 |  Third Party Gateway  | Failure to connect to thirdparty gateway API | Retry once in 5 seconds then in another 30 seconds if the retry fails | Third Party Gateway down Please report to front desk | 
| 3 | Wifi Loss | Wifi-status returns false | Wait 45 seconds for wifi to try and reconnected alert staff | Wifi lost transcation canceled staff is coming! |
| 4 | Kisosk drive fails| Unscheduled restarts/data differs from earlier scan | Disables the kisok | If card in unlock card 




