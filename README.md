
# MySQL Agent using Ollama, Qdrant
This notebook runs through the process of using the `vanna` Python package to generate SQL using AI (RAG + LLMs) including connecting to a database and training.

## Setup

```
%pip install 'vanna[qdrant,ollama,mysql]'
```

## Run Qdrant

```
docker run -p 6333:6333 -p 6334:6334 \
    -v "$(pwd)/qdrant_storage:/qdrant/storage:z" \
    qdrant/qdrant
```

## Run Ollama

```
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

## Download Model 

```
ollama run deepseek-r1:671b
```

## Init Project

```
from vanna.ollama import Ollama
from vanna.qdrant import Qdrant_VectorStore
from qdrant_client import QdrantClient



class MyVanna(Qdrant_VectorStore, Ollama):
    def __init__(self, config=None):
        Qdrant_VectorStore.__init__(self, config=config)
        Ollama.__init__(self, config=config)

# Instantiate the QdrantClient properly
client = QdrantClient(url="http://localhost")

# Now pass the client object instead of the string representation
vn = MyVanna(config={'client': client, 'model': 'deepseek-r1:8b'})
```

## Database Connection

```
vn.connect_to_mysql(host='localhost', dbname='sales_db', user='root', password='admindb', port=3306)
```


## Training Agent


```

# The information schema query may need some tweaking depending on your database. This is a good starting point.
df_information_schema = vn.run_sql("SELECT * FROM INFORMATION_SCHEMA.COLUMNS")


# This will break up the information schema into bite-sized chunks that can be referenced by the LLM
plan = vn.get_training_plan_generic(df_information_schema)
plan

# If you like the plan, then uncomment this and run it to train
# vn.train(plan=plan)



# The following are methods for adding training data. Make sure you modify the examples to match your database.

# DDL statements are powerful because they specify table names, colume names, types, and potentially relationships
vn.train(ddl="""
    CREATE TABLE IF NOT EXISTS my-table (
        id INT PRIMARY KEY,
        name VARCHAR(100),
        age INT
    )
""")

# Sometimes you may want to add documentation about your business terminology or definitions.
vn.train(documentation="Our business defines OTIF score as the percentage of orders that are delivered on time and in full")

# You can also add SQL queries to your training data. This is useful if you have some queries already laying around. You can just copy and paste those from your editor to begin generating new SQL.
vn.train(sql="SELECT * FROM my-table WHERE name = 'John Doe'")

```

# Train Model based on your context 

```

# Business Terminology Documentation
vn.train(documentation="""
    In our distribution company, we use several key entities within the ERPNext system to manage our operations efficiently. 
    Below are the key business terms we define and use in our system:

    - **Item**: Represents a product or service that we sell or purchase.
    - **Sales Invoice**: A document that reflects the sale of goods or services, including taxes, pricing, and payment terms.
    - **Sales Invoice Item**: An individual item listed within a sales invoice, detailing the specific product sold.
    - **Purchase Order**: A request document created by us to order goods or services from a supplier.
    - **Purchase Receipt**: A document indicating that goods have been received from a supplier, often tied to a purchase order.
    - **General Ledger**: The central record in the accounting system, capturing all financial transactions, including assets, liabilities, revenues, and expenses.
    - **Delivery Note**: A document confirming the dispatch of goods, typically used for deliveries to customers.
    - **Payment Entry**: A document representing a payment made or received, typically linking to an invoice or order.
    - **Territory**: A geographical area or market segment to which a sales or customer team is assigned.
    - **Customer**: A person or organization that buys goods or services from us.
    - **Warehouse**: A storage location used for managing inventory and goods.
    - **Customer Group**: A categorization of customers for easy segmentation, often based on sales volume or region.
    - **Item Group**: A categorization of items used for grouping similar products, helping with inventory management and reporting.
    - **Project**: An initiative or contract that involves a series of tasks, often used to manage customer orders, deliveries, and payments.
    - **Sales Person**: The employee responsible for managing customer relationships and sales transactions.
    - **User**: A person who interacts with the ERPNext system, with different roles and permissions assigned.
    
    These definitions help us streamline our processes, maintain consistency across departments, and ensure accurate data flow through our ERPNext system.
""")

# SQL Query for Managing Sales Data
vn.train(sql="""
    -- Get Payment details on date range
    SELECT 
        SUM(wi.paid_amount) AS total_payment
    FROM `tabPayment Entry` wi
    WHERE wi.posting_date BETWEEN '2023-01-01' AND '2023-02-28'

    -- Select all items associated with a specific sales invoice, including item details and quantities
    SELECT
        si.name AS sales_invoice,
        sii.item_code,
        sii.qty,
        sii.rate,
        sii.amount,
        sii.discount_amount
    FROM `tabSales Invoice` si
    JOIN `tabSales Invoice Item` sii ON si.name = sii.parent
    WHERE si.customer = 'Customer XYZ' AND si.docstatus = 1;

    -- Select purchase receipts linked to a specific purchase order, with received quantities and item details
    SELECT
        pr.name AS purchase_receipt,
        pri.item_code,
        pri.received_qty,
        pri.amount
    FROM `tabPurchase Receipt` pr
    JOIN `tabPurchase Receipt Item` pri ON pr.name = pri.parent
    WHERE pr.purchase_order = 'PO-1234' AND pr.docstatus = 1;

    -- Query to fetch payment entries linked to specific sales invoices
    SELECT
        pe.name AS payment_entry,
        pe.party_type,
        pe.party,
        pe.amount,
        pe.posting_date
    FROM `tabPayment Entry` pe
    JOIN `tabSales Invoice` si ON pe.reference_name = si.name
    WHERE si.customer = 'Customer XYZ' AND pe.docstatus = 1;

    -- Retrieve general ledger entries for a specific customer account
    SELECT
        gl.account,
        gl.debit,
        gl.credit,
        gl.posting_date
    FROM `tabGL Entry` gl
    WHERE gl.party_type = 'Customer' AND gl.party = 'Customer XYZ';

    -- Query for warehouse stock levels for a particular item
    SELECT
        wi.warehouse,
        wi.item_code,
        wi.actual_qty
    FROM `tabStock Ledger Entry` wi
    WHERE wi.item_code = 'Item ABC' AND wi.docstatus = 1;
""")

```

## Asking the AI


```

vn.ask(question="What are the last invoices?")

```




![image](https://github.com/user-attachments/assets/698b3cd4-ea4b-45c7-b511-56282fee28ce)

## Response

![image](https://github.com/user-attachments/assets/40039bef-e694-4f25-a8df-4ea88c2a69e1)
