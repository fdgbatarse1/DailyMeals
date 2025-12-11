# DailyMeals
DailyMeals is a service that prepares and delivers fresh meals to people who want convenient, healthy food without having to cook every day. Customers can subscribe to weekly meal plans or order meals whenever they want. The goal of the system is to make the entire experience smooth — from choosing meals, to preparing them in the kitchen, to getting them delivered on time.

DailyMeals has several types of users:
- Customers, who look at the weekly menu, pick their meals, pay, and track deliveries.
- Kitchen staff, who see what meals need to be cooked each day and how many portions to prepare.
- Delivery drivers, who get their assigned routes and confirm when meals have been delivered.
- Administrators, who manage the menu, update prices, review orders, and handle issues.

Because all these people interact with the system differently, DailyMeals provides several different interfaces: a website or app for customers, internal dashboards for the kitchen and admin teams, and a simple mobile view for drivers.

The system also connects with outside services to handle things like payments, sending email or text updates, and mapping routes for deliveries.

Behind the scenes, DailyMeals keeps track of customers, meals, orders, ingredients, weekly menus, and delivery schedules. Some tasks happen automatically — like generating the list of meals the kitchen must prepare each morning, charging weekly subscribers, or sending notifications as meals move through the delivery process.

Overall, DailyMeals is meant to represent a realistic, modern application: one that has multiple user roles, several parts working together, and real-world challenges like timing, coordination, and communication between different teams. The project will explore how to design this system in a clear and organized way so each part fits together smoothly.

## Project elements

### What are the elements of your project?
- **Customer App/Web**: where people browse meals, manage subscriptions, place orders, and track deliveries.
- **Kitchen Dashboard**: shows cooking plans, quantities needed, ingredient lists, and daily preparation tasks.
- **Delivery Driver App**: gives drivers their assigned routes and lets them confirm deliveries.
- **Admin Dashboard**: used by staff to publish menus, modify meals, review orders, handle refunds, and manage users.
- **Backend Services**: handle orders, subscriptions, menus, user accounts, payments, and weekly batch processing.
- **Data Storage**: for users, meals, ingredients, orders, delivery records, and meal-prep batches.
- **External Integrations**: payment provider, email/SMS notifications, and mapping/delivery tools.

### Which architectural patterns will apply?
DailyMeals will follow a modular service-based architecture (not fully microservices, but more separated than a monolith). Each major area—orders, menus, users, operations—acts as its own module, making the system easier to maintain and expand.
- **API-first design**: each frontend communicates through well-defined APIs.
- **Asynchronous processing**: for tasks like generating weekly cooking plans, creating batch orders, and sending notifications.
- **Event-driven moments**: when certain actions trigger automatic updates (e.g., “order paid” → “send confirmation” → “add to kitchen schedule”).
- **Role-based access**: so each user only sees what they need.

### How will the pieces communicate with each other?
- REST APIs between the frontends and the backend services.
- Internal service-to-service APIs for operations like payment validation, generating cooking lists, or updating delivery status.
- Message queues (or another async mechanism) for longer-running tasks such as:
  - creating a daily preparation schedule,
  - processing subscription renewals,
  - sending emails or SMS updates.
 
### What kind of authN and authZ will be needed where?
#### Customers
- Sign up/sign in with email + password or OAuth.
- Can manage their own orders, account details, and preferences.

#### Kitchen Staff
- Internal login.
- Access only to the kitchen dashboard: meal quantities, ingredient lists, and daily tasks.

#### Delivery Drivers

- Lightweight login (mobile-friendly).
- Access only to their assigned routes and delivery confirmations.

#### Administrators
- Stronger login requirements.
- Full access to menus, orders, refunds, reports, and user management.

## Project User Flows

### Core User Flows
1. User signs up for an account
2. User logs in and gets a session/token
3. Customer views weekly menu
4. Customer places an order (one-time or subscription)
5. Subscription renewal processing (automatic behind the scenes)
6. Kitchen staff views daily prep list
7. Delivery driver updates delivery status
8. Admin creates or updates a menu item
9. System sends notifications to user (order confirmation, delivery update)
10. Customer tracks delivery progress

### Unique Flows
#### Customer places an order (one-time or subscription)
```mermaid
sequenceDiagram
    %% ACTORS / COMPONENTS (from your architecture)
    actor C as Customer
    participant APP as Customer Web/Mobile App
    participant GW as Gateway
    participant AUTH as Authentication
    participant MENU as Menu
    participant MEAL as Meal
    participant DB as Relational DB
    participant CACHE as Cache DB
    participant ORD as Order
    participant SUB as Subscription
    participant PAY as Payment
    participant OQ as Order queue
    participant KIT as Kitchen
    participant DEL as Delivery
    participant NOTIF as Notifications
    participant NQ as Notifications queue
    participant NPROV as "Notifications provider"
    participant PPROV as "Payment provider"

    %% 1. CUSTOMER BROWSES MENU
    C->>APP: Open app
    C->>APP: View weekly menu
    APP->>GW: GET /menu
    GW->>MENU: Fetch menu
    MENU->>CACHE: Get cached menu

    alt Cache hit
        CACHE-->>MENU: Menu data
    else Cache miss
        MENU->>DB: Select meals and prices
        DB-->>MENU: Menu rows
        MENU->>CACHE: Store menu
        CACHE-->>MENU: OK
    end

    MENU-->>GW: Menu DTO
    GW-->>APP: Menu JSON
    APP-->>C: Show meals
    C->>APP: Build cart

    %% 2. CUSTOMER STARTS CHECKOUT
    C->>APP: Choose delivery slot
    C->>APP: Choose one-time or subscription
    C->>APP: Tap Checkout

    %% 3. CREATE DRAFT ORDER
    APP->>GW: POST /orders
    GW->>AUTH: Validate token
    AUTH-->>GW: OK
    GW->>ORD: createOrder(request)

    ORD->>MENU: Validate items and prices
    MENU->>DB: Verify meals and availability
    DB-->>MENU: Validation result
    MENU-->>ORD: Validation OK

    ORD->>DB: Insert order (pending_payment)
    DB-->>ORD: Order ID
    ORD-->>GW: Draft order with totals
    GW-->>APP: Draft order response
    APP-->>C: Show order summary

    %% 4. PAYMENT
    C->>APP: Confirm payment method
    APP->>GW: POST /payments
    GW->>AUTH: Validate token
    AUTH-->>GW: OK
    GW->>PAY: processPayment(orderId, amount)

    PAY->>PPROV: Charge customer
    PPROV-->>PAY: Payment result

    alt Payment successful
        PAY->>DB: Insert payment record (success)
        DB-->>PAY: OK
        PAY->>ORD: markPaid(orderId)
        ORD->>DB: Update order to paid
        DB-->>ORD: OK

        %% 5. BRANCH: ONE-TIME VS SUBSCRIPTION
        alt Order type is subscription
            ORD->>SUB: createSubscription(orderId)
            SUB->>DB: Insert subscription
            DB-->>SUB: OK
            SUB->>PAY: storePaymentMethodToken
            PAY->>PPROV: Save payment method
            PPROV-->>PAY: Token saved
            PAY-->>SUB: Token reference
        else Order type is one-time
            ORD-->>ORD: No subscription created
        end

        %% 6. ENQUEUE ORDER FOR OPERATIONS
        ORD->>OQ: Enqueue OrderPaid event
        OQ-->>KIT: Message for kitchen planning
        OQ-->>DEL: Message for delivery scheduling

        %% 7. SEND CONFIRMATION NOTIFICATIONS
        ORD->>NOTIF: OrderPaid event
        NOTIF->>NQ: Enqueue notification job
        NQ-->>NOTIF: Dequeue job
        NOTIF->>NPROV: Send confirmation message
        NPROV-->>NOTIF: Accepted

        PAY-->>GW: Payment success
        GW-->>APP: 200 OK with final order
        APP-->>C: Show confirmation
    else Payment failed
        PAY->>DB: Insert payment record (failed)
        DB-->>PAY: OK
        PAY-->>GW: Failure with reason
        GW-->>APP: Error response
        APP-->>C: Show payment error
        ORD->>DB: Optionally update order status
        DB-->>ORD: OK
    end

```
#### Kitchen staff view of preparation list
```mermaid
sequenceDiagram
    %% ACTORS / COMPONENTS
    actor KS as Kitchen Staf
    participant KD as Kitchen Dashboard
    participant GW as Gateway
    participant AUTH as Authentication
    participant KIT as Kitchen
    participant ORD as Order
    participant MENU as Menu
    participant DB as Relational DB

    %% 1. STAFF OPENS DASHBOARD AND REQUESTS PREP LIST
    KS->>KD: Open dashboard
    KS->>KD: Select date (today)
    KD->>GW: GET /kitchen/prep-list?date=today
    GW->>AUTH: Validate staff token
    AUTH-->>GW: OK
    GW->>KIT: getPrepList(today)

    %% 2. KITCHEN SERVICE HANDLES PREP LIST REQUEST
    KIT->>DB: Select prep list for today
    alt Prep list exists
        DB-->>KIT: Prep list rows
        KIT-->>GW: Prep list data
        GW-->>KD: Prep list response
        KD-->>KS: Show preparation list
    else Prep list missing or stale
        DB-->>KIT: No rows

        %% 2A. GENERATE PREP LIST FROM ORDERS
        KIT->>DB: Select paid orders for today
        DB-->>KIT: Orders for today
        KIT->>MENU: Get meal details
        MENU->>DB: Select meals by ids
        DB-->>MENU: Meal records
        MENU-->>KIT: Meal info

        KIT->>KIT: Aggregate meals and quantities
        KIT->>DB: Insert new prep list for today
        DB-->>KIT: Insert OK

        KIT-->>GW: New prep list data
        GW-->>KD: Prep list response
        KD-->>KS: Show preparation list
    end

    %% 3. OPTIONAL: STAFF UPDATES ITEM STATUS
    KS->>KD: Mark item as in progress or done
    KD->>GW: PATCH /kitchen/prep-list/items/{id}
    GW->>AUTH: Validate staff token
    AUTH-->>GW: OK
    GW->>KIT: updatePrepItemStatus(id,status)
    KIT->>DB: Update prep item row
    DB-->>KIT: Update OK
    KIT-->>GW: Updated item
    GW-->>KD: Updated item response
    KD-->>KS: Reflect new status in UI
```
#### Delivery driver updates
```mermaid
sequenceDiagram
    actor KS as Kitchen Staff
    participant KD as Kitchen Dashboard
    participant GW as Gateway
    participant KIT as Kitchen Service
    participant DB as Relational DB
    participant ORD as Order Service
    participant MENU as Menu Service

    %% 1. STAFF REQUESTS PREP LIST
    KS->>KD: Open dashboard (today)
    KD->>GW: GET /kitchen/prep-list?date=today
    GW->>KIT: Request prep list

    %% 2. KITCHEN SERVICE CHECKS DATA
    KIT->>DB: Load prep list for today

    alt Prep list exists
        DB-->>KIT: Prep list data
        KIT-->>GW: Prep list
        GW-->>KD: List of meals + quantities
        KD-->>KS: Display prep list
    else No prep list
        DB-->>KIT: No data

        %% Generate the prep list
        KIT->>ORD: Get today’s paid orders
        ORD->>DB: Query orders
        DB-->>ORD: Order items
        ORD-->>KIT: Orders

        KIT->>MENU: Get meal & ingredient details
        MENU->>DB: Fetch meals
        DB-->>MENU: Meal info
        MENU-->>KIT: Meal details

        KIT->>KIT: Aggregate totals
        KIT->>DB: Save new prep list
        DB-->>KIT: OK

        KIT-->>GW: New prep list
        GW-->>KD: Return prep list
        KD-->>KS: Display list
    end

```
#### Admin creates or updates a menu item
```mermaid
sequenceDiagram
    %% ACTORS / COMPONENTS (from your architecture)
    actor A as Administrator
    participant AD as Administrator Dashboard
    participant GW as Gateway
    participant AUTH as Authentication
    participant MENU as Menu
    participant MEAL as Meal
    participant ING as Ingredients
    participant DB as Relational Database
    participant CACHE as Cache Database
    participant NOTIF as Notifications
    participant NQ as Notifications queue
    participant NPROV as Notifications provider

    %% 1. ADMIN OPENS MENU MANAGEMENT PAGE
    A->>AD: Open "Menu management" page
    AD->>GW: GET /admin/menu
    GW->>AUTH: Validate admin token
    AUTH-->>GW: OK
    GW->>MENU: Fetch current menu
    MENU->>DB: SELECT current menu items + meals
    DB-->>MENU: Menu data
    MENU-->>GW: Menu DTO
    GW-->>AD: Menu list to render

    %% 2. ADMIN CREATES OR EDITS A MENU ITEM
    A->>AD: Fill form (name, price,<br/>ingredients, availability, week)
    A->>AD: Click "Save"

    alt Create new menu item
        AD->>GW: POST /admin/menu-items (payload)
    else Update existing menu item
        AD->>GW: PUT /admin/menu-items/{id} (payload)
    end

    GW->>AUTH: Check admin permissions
    AUTH-->>GW: Authorized
    GW->>MENU: createOrUpdateMenuItem(payload)

    %% 3. MENU SERVICE VALIDATION & PERSISTENCE
    MENU->>MEAL: Ensure meal definition exists / update recipe
    MEAL->>ING: Validate / link ingredients
    ING->>DB: Upsert ingredients
    DB-->>ING: OK

    MEAL->>DB: Upsert meal (name, description, nutrients)
    DB-->>MEAL: OK

    MENU->>DB: Insert/Update menu item<br/>(mealId, price, dates, flags)
    DB-->>MENU: OK

    MENU->>CACHE: Invalidate or refresh cached menu
    CACHE-->>MENU: OK

    MENU-->>GW: Success + updated menu item
    GW-->>AD: 200 OK + updated menu data
    AD-->>A: Show success message & refreshed menu

    %% 4. OPTIONAL: PUBLISH & NOTIFY CUSTOMERS
    alt Admin chose "Publish & notify customers"
        MENU->>NOTIF: Event: MenuUpdated / NewMenuPublished
        NOTIF->>NQ: Enqueue notification jobs
        NQ-->>NOTIF: Jobs dequeued for processing (async)
        NOTIF->>NPROV: Send email/push "New weekly menu available"
        NPROV-->>NOTIF: Provider accepted
    end
```
