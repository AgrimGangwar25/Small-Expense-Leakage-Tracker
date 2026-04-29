# Expense Leakage Tracker: Comprehensive Project Report

---

## 3. Abstract

In today's fast-paced digital economy, tracking personal finances has become increasingly complex. While traditional expense trackers focus merely on recording transactions, they often fail to address the core issue of financial mismanagement: "expense leakage" or money wasted on non-essential, impulsive purchases. The **Expense Leakage Tracker** is a full-stack, automated web application designed specifically to identify, categorize, and curb financial leakage in real-time. 

Built using a robust technology stack comprising **Python (Flask)**, **PostgreSQL**, and a modern **Vanilla JavaScript/CSS** frontend, this system introduces an innovative webhook-based architecture. Rather than relying solely on manual data entry, the system is designed to ingest raw transaction data directly from bank alerts (e.g., via an Android SMS interceptor). Uncategorized transactions trigger a real-time "intervention modal" on the user's dashboard via short-polling, forcing the user to consciously categorize the expense as either 'Essential' or 'Leakage'. With dynamic data visualization powered by Chart.js and an intuitive glassmorphism UI, the platform provides users with immediate, actionable insights into their spending habits, ultimately fostering better financial discipline and budget management.

---

## 1. Problem Statement

Personal financial health is heavily dependent on the ability to distinguish between essential needs (e.g., rent, groceries, utilities) and non-essential wants (e.g., dining out, impulse buys, subscriptions). The latter category represents "expense leakage"—small, frequent expenditures that cumulatively drain a budget without the user noticing. 

Existing solutions present several critical problems:
1. **Manual Entry Fatigue**: Users abandon budgeting apps because manually entering every $5 coffee is tedious and unsustainable.
2. **Lack of Immediate Accountability**: By the time a user reviews their monthly credit card statement, the budget has already been exceeded. There is no real-time friction to make the user conscious of their spending.
3. **Binary Tracking**: Most apps treat all expenses equally, failing to visually isolate "wasted" money from necessary survival spending.

**Objective**: To develop an automated system that intercepts raw financial transactions, forces immediate user accountability for unclassified spending, and visually contrasts essential spending against financial leakage against a defined monthly budget limit.

---

## 2. Design Constraints

The development of the Expense Leakage Tracker was guided by strict technical and architectural constraints:

### Technical & Framework Constraints
- **Backend Stack**: Must be developed using **Python 3** and the **Flask** microframework to ensure lightweight, rapid API routing.
- **Frontend Stack**: Must utilize **Vanilla HTML, CSS, and JavaScript**. The use of heavy frontend frameworks (like React, Angular, or Vue) or utility CSS libraries (like Tailwind) is strictly prohibited to maintain raw performance and absolute control over the DOM.
- **Database Architecture**: Must rely on **PostgreSQL** as the relational database, accessed via raw SQL queries using `psycopg2` rather than relying on heavy ORMs (like SQLAlchemy), ensuring highly optimized queries.

### Functional Constraints
- **Asynchronous UI Updates**: The frontend dashboard must detect new transactions in the database (ingested via webhooks) within seconds without requiring a full page reload. This necessitates an efficient short-polling mechanism.
- **Strict Schema Enforcement**: The database must enforce data integrity through Foreign Key constraints (e.g., an expense cannot be logged without an existing category and user).
- **Security & Scope**: The current iteration is scoped as a single-tenant local environment (hardcoded `user_id = 1`) to focus on core functionality before scaling to multi-tenant authentication.

---

## 4. Description of the System

The Expense Leakage Tracker is an end-to-end web application divided into three core environments: the Database Layer, the Flask API Layer, and the Client Dashboard.

### 4.1 The Frontend Dashboard
The user interface is built using a modern CSS Grid layout featuring a "glassmorphism" aesthetic (frosted glass, soft shadows, rounded borders) to ensure a premium user experience.
- **Summary Cards**: Three interactive cards display *Total Expenses*, *Money Wasted* (Leakage), and the *Remaining Budget*. The budget card includes dynamic CSS logic to turn bright red if the user exceeds their monthly limit.
- **Data Visualization**: A center-aligned Donut Chart (powered by Chart.js) provides a stark visual contrast between Essential Spending (Green), Wasted Leakage (Red), and Unspent Budget (Muted Gray).
- **Recent Expenses Table**: A tabular history of transactions featuring dynamic color-coded badges to instantly identify the nature of the expense.
- **Intervention Modal**: A hidden, absolute-positioned modal that locks the screen when a new unclassified transaction is detected, requiring immediate categorization.

### 4.2 The Flask API Backend
The backend serves as the central nervous system, exposing RESTful endpoints:
- **`GET /api/dashboard_data`**: A heavy-lifting aggregation endpoint that executes complex SQL `JOIN`s and `COALESCE` sums to return budget limits, total spent, total wasted, and a serialized list of all expenses in a single network request.
- **`POST /api/webhook/bank_transaction`**: The ingestion engine. It accepts raw JSON payloads from external sources (like an Android app intercepting SMS), inserting the data with a default `pending_intervention` status.
- **`PUT /api/expenses/<id>/categorize`**: Updates the transaction status to `categorized` and links the appropriate Foreign Key.
- **`PUT /api/budget`**: Allows real-time updating of the user's monthly limit via an upsert (`ON CONFLICT` or `Rowcount` check) query.

### 4.3 The Workflow Pipeline
1. **Trigger**: A bank transaction occurs.
2. **Ingestion**: The webhook endpoint receives the data and saves it as `pending_intervention`.
3. **Detection**: The dashboard's JavaScript `setInterval` loop (pinging every 5 seconds) detects the pending status.
4. **Intervention**: The UI halts, displaying the modal with the transaction details.
5. **Resolution**: The user categorizes the expense. The backend updates the database.
6. **Sync**: The dashboard re-fetches the `/api/dashboard_data` endpoint, smoothly updating the ring chart, summary cards, and tables.

---

## 5. Methodology

The project was executed using an iterative, Agile-inspired methodology divided into four distinct phases:

### Phase 1: Data Modeling & Environment Setup
- Designed the Entity-Relationship (ER) diagram mapping Users to Categories, Budgets, and Expenses.
- Initialized the PostgreSQL database (`expense_db`) and executed DDL scripts to create the tables.
- Set up the Python virtual environment and installed core dependencies (`Flask`, `psycopg2-binary`).

### Phase 2: Core Backend Logic & Webhook Creation
- Developed the database connection utility (`get_db_connection()`).
- Implemented the `POST` webhook to securely accept external data.
- Encountered and resolved database schema conflicts (e.g., mapping column names like `category_name` and removing non-existent `date` columns).

### Phase 3: UI/UX Engineering
- Developed the static HTML structure and CSS variables for the dark-mode aesthetic.
- Implemented the 5-second asynchronous polling loop using the JavaScript `fetch` API.
- Integrated the Chart.js CDN and built the dynamic rendering logic to handle chart destruction/updates.

### Phase 4: Refinement & Architecture Overhaul
- Consolidated fragmented API calls (e.g., individual routes for leakage, totals, and tables) into a single, highly-optimized `/api/dashboard_data` endpoint to reduce network overhead.
- Added interactive UX features, such as the clickable budget card and dynamic `Math.max()` clamping for negative budget scenarios in the Chart.js rendering.

---

## 6. Database Design

The relational database (`expense_db` in PostgreSQL) is designed in the Third Normal Form (3NF) to ensure data integrity and prevent redundancy. 

### Core Tables and Relationships:

#### 1. `users` Table
- `user_id` (Primary Key, INT, Auto-increment)
- `username` (VARCHAR)
- `email` (VARCHAR, Unique)
- `created_at` (TIMESTAMP)
- *Role*: The root entity. All other tables rely on this to establish data ownership.

#### 2. `user_settings` Table
- `setting_id` (Primary Key, INT)
- `user_id` (Foreign Key referencing `users(user_id)`)
- `monthly_limit` (DECIMAL/NUMERIC) - Defines the global budget limit.
- `alert_enabled` (BOOLEAN)

#### 3. `categories` Table
- `category_id` (Primary Key, INT)
- `user_id` (Foreign Key referencing `users(user_id)`)
- `category_name` (VARCHAR) - E.g., "Groceries", "Uber", "Dining Out".
- `is_essential` (BOOLEAN) - **Crucial logic pivot**: Determines if an expense linked to this category is "Leakage" (`false`) or "Essential" (`true`).

#### 4. `expenses` Table
- `expense_id` (Primary Key, INT)
- `user_id` (Foreign Key referencing `users(user_id)`)
- `category_id` (Foreign Key referencing `categories(category_id)`, Nullable for pending transactions)
- `amount` (DECIMAL)
- `description` (TEXT)
- `source` (VARCHAR) - The merchant or bank source.
- `expense_date` (DATE/TIMESTAMP)
- `status` (VARCHAR) - Acts as a state machine (`'pending_intervention'`, `'categorized'`).

#### 5. `alerts` Table (For Future Scope)
- `alert_id` (Primary Key, INT)
- `user_id` (Foreign Key referencing `users(user_id)`)
- `message` (TEXT)
- `is_read` (BOOLEAN)

### ER Relationship Summary
- A **User** *Logs* multiple **Expenses** (1:N).
- A **User** *Creates* multiple **Categories** (1:N).
- A **Category** *Classifies* multiple **Expenses** (1:N).
- A **User** *Has* one **User Settings** profile (1:1).

---

## 7. Future Scope

While the current MVP successfully automates expense tracking and forces behavioral intervention, the architecture is designed to support several future scaling opportunities:

1. **Native Android SMS Interceptor (Currently Designed)**
   - Developing a background Kotlin Android app utilizing a `BroadcastReceiver`.
   - The app will silently read incoming bank SMS messages, use Regular Expressions (Regex) to extract the currency amount and merchant, and automatically fire the HTTP POST request to the Flask webhook, removing the need for any manual input entirely.
2. **Machine Learning Categorization**
   - After sufficient data is collected, training a lightweight NLP model (like a Naive Bayes classifier) to auto-predict the `category_id` based on the `description` and `source` strings, reserving the UI modal only for low-confidence predictions.
3. **Multi-Tenant Authentication**
   - Replacing the hardcoded `user_id = 1` logic with JWT (JSON Web Tokens) or Flask-Login session management.
   - Implementing secure password hashing (`bcrypt`) to allow multiple users to utilize the application simultaneously on a cloud deployment (e.g., AWS EC2 or Heroku).
4. **Push Notifications**
   - Integrating with Firebase Cloud Messaging (FCM) or Twilio to send proactive push notifications/SMS to the user when they hit 80%, 90%, and 100% of their `monthly_limit`.
5. **Advanced Analytics**
   - Adding historical line charts to track month-over-month leakage trends and PDF export capabilities for tax or personal auditing purposes.
