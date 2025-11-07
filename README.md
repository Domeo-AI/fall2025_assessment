
# SWE Internship Take-Home Assessment

Please donwload (.zip) this repository and make your changes locally. Once you have completed the assesment, please email a .zip file containig all the files detailed in the **submission** section of this README to kiryl@domeo.ai **and** donghan@domeo.ai .

## The Challenge

We have just signed a new client, Unicorn Inc!
As part of the implementation process, they have sent over a .csv file containing all of their AR data. Your job is to parse the file, and load it into a relational database of your choosing (SQLite, Postgres, etc.). Once the data is loaded, you will also need to create a couple of GET API endpoints so that we can extract and display the data on our web application.

**Time**: 3-4 hours

## What We're Looking For

You'll find AR data in `data/unicorn_inc.csv`. Your job is to:

1. **Build a data ingestion pipeline** that loads this data into a database
2. **Design a database schema** that makes sense for this data
3. **Create a REST API** to query the data
4. **Document your solution** and explain how you'd deploy it in production

## Understanding the Data

Before you start building, take time to understand the structure of `data/unicorn_inc.csv`. Here's what each column represents:

### Column Descriptions

| Column             | Description                                          | Data Type | Example                     |
| ------------------ | ---------------------------------------------------- | --------- | --------------------------- |
| `CustomerName`     | The name of the customer company                     | String    | "LifeSpring Medical"        |
| `InvoiceNumber`    | Unique identifier for each invoice                   | String    | "DF2024988"                 |
| `InvoiceDate`      | Date when the invoice was issued                     | Date      | "11/19/24"                  |
| `CustomerPoNumber` | Customer's purchase order reference number           | String    | "PO-000001"                 |
| `BillTotal`        | Total invoice amount in the specified currency       | Decimal   | 3150                        |
| `Applied`          | Amount that has been applied/paid toward the invoice | Decimal   | 3150                        |
| `Status`           | Current status of the invoice                        | String    | "Closed", "Pending"         |
| `Currency`         | Currency code for the transaction                    | String    | "USD"                       |
| `ContactName`      | Primary contact person at the customer               | String    | "Angela Scott"              |
| `ContactPhone`     | Contact's phone number                               | String    | "555-513-2964"              |
| `ContactEmail`     | Contact's email address                              | String    | "angela@lensandlight.com"   |
| `CustomerTerms`    | Payment terms for the invoice                        | String    | "Net 30", "Net 15", "Net 7" |
| `DueDate`          | Date when payment is due                             | Date      | "12/19/24 00:00"            |

### Key Insights for Database Design

1. **Data Types to Consider**:
   - Use `DECIMAL` or `NUMERIC` for monetary values (BillTotal, Applied) to avoid floating-point precision issues
   - Use proper `DATE` or `DATETIME` types for InvoiceDate and DueDate
   - Consider whether InvoiceNumber should be the primary key or if you need a separate ID field

2. **Business Logic**:
   - When `Applied` equals `BillTotal`, the Status is typically "Closed"
   - The difference between `BillTotal` and `Applied` represents the outstanding balance
   - Payment terms (Net 30, etc.) determine the DueDate calculation from InvoiceDate IFF the DueDate is missing

3. **Open AR Calculation**: To calculate total open AR (accounts receivable) for a customer, you'll need to sum up all invoices with outstanding balance, and where the `Status = 'Pending'`.

Please note the data provided is **completely made up**. You are welcome to upload the data to chatGPT or use any other AI tools.

### Business Logic

- **Outstanding Balance**: `max(BillTotal - Applied, 0)`
- **Past-Due Rule**: `Outstanding > 0 AND DueDate < as_of`
- **Status Values**: `"Closed"` or `"Pending"` (Pending here means the invoice is still open)
- **DueDate Calculation**: if missing, derive using `CustomerTerms` (`Net 7|15|30`)
- **Date Parsing**: Please Use the following date parsing format: `MM/DD/YY`

## Requirements

### 1. Data Pipeline

- Parse and load the CSV data into a database
- Validate data quality (handle bad data gracefully)
- Make it safe to run multiple times (idempotent)
- Add logging so we can see what's happening

### 2. Database

- Design a schema that makes sense
- Use appropriate data types (think about money and dates!)
- Consider relationships, indexes, and constraints
- Use `NUMERIC(18,2)` for monetary fields and enforce non-negative values
- Use ISO `YYYY-MM-DD` format for stored dates

### 3. API

You must build the following endpoints:

#### GET `/invoices/past-due`

Returns invoices with positive outstanding balance (`BillTotal - Applied > 0`) whose `DueDate` is before `as_of`.

**Query params**
- `as_of` (optional, ISO date, default = server “today” in America/New_York)
- `limit` (int, default 50, max 200)
- `offset` (int, default 0)
- `sort` (optional: `due_date.asc` | `due_date.desc`, default `due_date.asc`)

**Response (200)**
```json
{
  "items": [
    {
      "invoice_number": "DF2024988",
      "customer_name": "LifeSpring Medical",
      "invoice_date": "2024-11-19",
      "due_date": "2024-12-19",
      "bill_total": "3150.00",
      "applied": "3000.00",
      "outstanding": "150.00",
      "currency": "USD",
      "status": "Pending",
      "days_past_due": 325
    }
  ],
  "total": 123,
  "limit": 50,
  "offset": 0
}
```

#### GET `/invoices/summary/month`

Returns the **sum of BillTotal** for invoices whose `InvoiceDate` falls in the target month.

**Query params**
- `month` (required, `YYYY-MM`)
- `customer_name` (optional; filters by case-insensitive exact match)

**Response (200)**
```json
{
  "month": "2025-07",
  "currency": "USD",
  "sum_bill_total": "123456.78",
  "count_invoices": 42
}
```
#### GET `/customers/contact`

Fetches contact info by customer name (case-insensitive).

**Query params**
- `name` (required)
- `limit` (int, default 10)
- `offset` (int, default 0)

**Response (200)**
```json
{
  "customer_name": "LifeSpring Medical",
  "contacts": [
    {
      "contact_name": "Angela Scott",
      "contact_email": "angela@lensandlight.com",
      "contact_phone": "555-513-2964",
      "last_seen_invoice_date": "2025-01-31"
    }
  ],
  "total": 1
}
```

**Common errors**
- `400` invalid date/month
- `404` for contact endpoint if zero matches (optional)

### 4. Documentation

Write a `SOLUTION.md` that explains:

- How to run your code
- API endpoints with examples
- Database design and reasoning
- AWS deployment design (services, flow, trade-offs)

### 5. Constraints

- Python 3.10+ (use Flask, FastAPI, or similar)
- SQL database (SQLite acceptable)
- Must run locally
- Production-quality code expected

### 6. Submission

Submit:
1. `schema_template.sql` (schema)
2. `parse_data.py` (parse CSV)
3. `load_data.py` (load parsed data)
4. `SOLUTION.md` (docs)
5. `requirements.txt` (dependencies)
6. `app.py` or `main.py` (API entrypoint)

### 7. Evaluation Criteria

| Area            | Weight |
| --------------- | ------ |
| Data Pipeline   | 30%    |
| Code Quality    | 10%    |
| Database Design | 35%    |
| API Design      | 15%    |
| Documentation   | 10%    |

### 8. Notes

- Use ISO date format in DB
- Use NUMERIC for money
- Upsert invoices by `InvoiceNumber` for idempotency
- Add logging on row-level warnings and summary info

## Questions?

Email Kiryl [kiryl@domeo.ai] or DH [donghan@domeo.ai]

---

**Good luck!** We're excited to see what you build.
