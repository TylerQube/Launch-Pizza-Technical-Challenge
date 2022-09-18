# Launch-Pizza Technical Outline

The design and description of a simple backend API for a fictional pizza delivery service Launch-Pizza.
Written as part of the application process for the UBC LaunchPad software club.

## Outline
1. Features
2. Authentication
3. Endpoints
    * Placing Orders
    * Tracking Orders
    * Viewing Receipts

## API Features
- Place order from set menu
- Track order status
- View order receipt for up to one year

## Authentication
Firstly, to ensure users can securely place orders, track order status, and view order history, each endpoint of this API must be verified with some form of client authentication.

Most recently, my authentication method of choice has been **JSON Web Tokens**. When a user logs in, for example, the backend generates a unique token signed by a backend secret key (optionally with an expiration date or other configuration).

> Example implementation with Node.js jsonwebtoken package
```javascript
/* The JWT can include user data, allowing the backend to quickly authenticate the user's identity */

const user = /* user data from login */ 

const jwt = require("jsonwebtoken");

const token = jwt.sign({ 
    id: user.id, 
    name: user.username, 
    email: user.email, 
},
secret,
{
    // must generate new token after expiration
    expiresIn: "2h"
});

// then include the token in response to the client
```

All subsequent http requests that require authentication must then include that same web token as an authorization header. 

## Order Placement Endpoint

```
POST https://[launch-pizza-api]/order/place
```
An order could be placed with a POST request, including the previously mentioned web token to attach the order to an account, as well as a request body of items from the menu.

Part of an example request body:
```json
{
    "items": [
        {
            "item_id": "...",
            "item_name": "Classic Pepperoni",
            "options": {
                "size": "large",
                ...
            }
        },
        ...
    ],
    "address": {
        "street": "2300 Lower Mall",
        "city": "Vancouver",
        "zip": "V6T 1Z4"
    }
}
```
In addition to checking request data on the client side, the backend must do the same, such as validating: 

* Order items
    * All items exist on the menu
    * Options (toppings, size, etc.) are valid and match the item type (e.g. only pizzas have toppings)
* Delivery Address
    * Exists
    * Within delivery range

If successful, the backend then returns/emails a tracking number (elaborated on in Order Tracking) to the client/customer.

## Order Tracking

```
GET https://[launch-pizza-api]/order/track/{order-number}
```

After successfully placing an order, customers could view it through the generation of a temporary tracking number.

The api stores this number in a temporary database entry for the duration of the order or other short period (e.g. 3 hours) before deletion.

When used in a GET request to the tracking endpoint, the backend returns data on the order's status, driver's location, and other necessary info for use on a client side webpage. 

## Order Receipts

```
GET https://[launch-pizza-api]/order/history/{start-ind}/{end-ind}
```

To view all past orders up to a year old, the frontend application would send a request including the start and end index of receipts to fetch. The backend then returns an array of receipt documents.

An example request might look like:
```
GET https://[launch-pizza-api]/order/history/0/10
```

Loading documents in batches (of maybe 10 or less at a time) would cut down on the large number of requests required to fetch receipts one by one, as well as avoids unnecessarily loading hundreds of receipts at once if fetching all receipts in one request. 


To remove receipts after one year or some arbitrary interval of time, the backend could schedule a daily job to iterate receipts and remove those with a delivery date from more than one year ago. 

Example iteration in JavaScript:
```javascript

// given:
//  array receipts
//  utility function daysSince(date)
//  some function removeDocument(document_id) corresponding to database 

const receiptExp = 365;

for(let i = 0; i < receipts.length; i++) {
    const receipt = receipts[i];
    if(daysSince(receipt.orderDate) >= receiptExp) removeDocument(receipt.id);
}
```

Alternatively, some systems might allow for the direct deletion of batches of old documents, e.g. MongoDB's deleteMany method
```javascript
// snippet sourced from: 
// https://stackoverflow.com/questions/46441006/how-can-i-remove-older-records-from-a-collection-in-mongodb
db.collection.deleteMany( { 
    orderExpDate : {
        // "$lt" selects documents where value of orderExpDate is 
        // less than the provided value (new Date)
        "$lt" : new Date(YEAR, MONTH, DATE) 
    } 
});
```

This method of iterating all receipts could become inefficient if Launch-Pizza were to grow to storing hundreds of thousands of receipts.

In that case, receipts could be evaluated for expiration when requested by the user.
