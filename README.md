

---

# 🟦 Project Overview  
**Project Title**:  
📌 **Real-time Change Data Capture (CDC) and Aggregation Using Delta Lake and Apache Spark Structured Streaming**

---

# ✅ **Project Description**  
This project implements an **incremental data pipeline** that performs **Change Data Capture (CDC)** on **UPI transaction data** using **Delta Lake** and **Apache Spark Structured Streaming**.  
The pipeline enables **real-time processing**, **change tracking**, and **aggregation** of transactional data for business insights like **merchant-level sales, refunds, and net sales**.

🔹 The **raw UPI transactions** are stored in a **Delta Lake table**, with **CDC enabled**, allowing the system to capture changes like inserts, updates, and deletes automatically.  
🔹 **Mock data batches** simulate transactions, including **inserts**, **updates**, and **refunds**.  
🔹 The pipeline reads the CDC stream in real-time, processes it, and writes **aggregated insights** to a separate **aggregated Delta table**, which can be used for **dashboards** or **reporting**.

---

# ✅ **Key Components**  
| **Component**               | **Description** |
|-----------------------------|-----------------|
| **Delta Lake (Raw Table)**  | Stores raw UPI transaction data with CDC enabled. Tracks inserts/updates/deletes. |
| **Mock Data Batches**       | Simulates transactional activity for inserts, updates, and refunds. |
| **Spark Structured Streaming (CDC Reader)** | Reads change data feed (CDC) in real-time from the Delta table. |
| **Aggregation Logic**       | Processes each batch from the CDC stream, performs merchant-level aggregations (sales, refunds, net sales). |
| **Delta Lake (Aggregated Table)** | Stores aggregated merchant-level metrics (live updates). |

---

# ✅ **Technology Stack**  
- **Apache Spark (Structured Streaming)**  
- **Delta Lake (Delta Tables + CDC)**  
- **PySpark (Python)**  
- **Databricks / Spark Runtime** (optional depending on environment)

---

# ✅ **Flow Diagram**

Here's a conceptual flow diagram of your solution:  

```
               ┌────────────────────────────┐
               │      Mock Data Batches     │
               │  (Insert, Update, Refund)  │
               └─────────────┬──────────────┘
                             │
                             ▼
               ┌────────────────────────────┐
               │   Raw UPI Transactions     │
               │ Delta Table (CDC Enabled)  │
               │  raw_upi_transactions_v1   │
               └─────────────┬──────────────┘
                             │
           (Change Data Feed │ CDC enabled) 
                             ▼
       ┌──────────────────────────────────────┐
       │ Spark Structured Streaming CDC Reader│
       │  (readStream + readChangeFeed = true)│
       └─────────────┬────────────────────────┘
                     │
                     ▼
     ┌─────────────────────────────────────┐
     │ Aggregation Logic (foreachBatch)    │
     │ Merchant-wise Sales, Refunds        │
     │ Sum & Calculate Net Sales           │
     └─────────────┬───────────────────────┘
                   │
                   ▼
     ┌─────────────────────────────────────┐
     │ Aggregated UPI Transactions         │
     │ Delta Table                         │
     │ aggregated_upi_transactions         │
     └─────────────────────────────────────┘
```


---
### ✅ **Aggregation Logic**
- `total_sales` sums `transaction_amount` where `transaction_status = 'completed'`.
- `total_refunds` sums **negative** `transaction_amount` where `transaction_status = 'refunded'`.
- `net_sales = total_sales + total_refunds`.

---

## 🔹 **Batch 1: Insert New Transactions**
```python
[
    ("T001", "upi1@bank", "M001", 500.0, "2024-12-21 10:00:00", "initiated"),
    ("T002", "upi2@bank", "M002", 1000.0, "2024-12-21 10:05:00", "initiated"),
    ("T003", "upi3@bank", "M003", 1500.0, "2024-12-21 10:10:00", "initiated"),
]
```

👉 All transactions are in `"initiated"` status, **no completed sales yet**, so aggregations would be:

| merchant_id | total_sales | total_refunds | net_sales |
|-------------|-------------|---------------|-----------|
| M001        | 0           | 0             | 0         |
| M002        | 0           | 0             | 0         |
| M003        | 0           | 0             | 0         |

---

## 🔹 **Batch 2: Update and Insert Transactions**
```python
[
    ("T001", "upi1@bank", "M001", 500.0, "2024-12-21 10:15:00", "completed"),  # Completed sale
    ("T002", "upi2@bank", "M002", 1000.0, "2024-12-21 10:20:00", "failed"),     # Failed, no sales
    ("T004", "upi4@bank", "M004", 2000.0, "2024-12-21 10:25:00", "initiated"),  # New initiated
]
```

👉 Processed as:  
- `T001` is now `"completed"` → 500 sales for `M001`.  
- `T002` is `"failed"` → no sales counted.  
- `T004` is `"initiated"` → no sales counted.

➡️ Aggregated results **after Batch 2**:

| merchant_id | total_sales | total_refunds | net_sales |
|-------------|-------------|---------------|-----------|
| M001        | 500         | 0             | 500       |
| M002        | 0           | 0             | 0         |
| M003        | 0           | 0             | 0         |
| M004        | 0           | 0             | 0         |

---

## 🔹 **Batch 3 (Based on Your Previous Message)**  
For reference, you mentioned:
```python
[
    ("T001", "upi1@bank", "M001", 500.0, "2024-12-21 10:30:00", "refunded"),  # Refund issued
    ("T003", "upi3@bank", "M003", 1500.0, "2024-12-21 10:35:00", "completed"), # Completed
]
```

This changes things to:  
- `T001` gets refunded → `-500` refund for `M001`.  
- `T003` becomes `"completed"` → 1500 sales for `M003`.

➡️ Final aggregation:

| merchant_id | total_sales | total_refunds | net_sales |
|-------------|-------------|---------------|-----------|
| M001        | 500         | -500          | 0         |
| M002        | 0           | 0             | 0         |
| M003        | 1500        | 0             | 1500      |
| M004        | 0           | 0             | 0         |

---

✅ **Explanation**
- **M001**: 500 sales minus 500 refund = 0 net.  
- **M002**: No completed sales, no refunds.  
- **M003**: 1500 completed sales, no refunds.  
- **M004**: No completed sales, no refunds.

---

## ✅ Final Aggregated Table (After Batch 3)
| merchant_id | total_sales | total_refunds | net_sales |
|-------------|-------------|---------------|-----------|
| **M001**    | 500         | -500          | 0         |
| **M002**    | 0           | 0             | 0         |
| **M003**    | 1500        | 0             | 1500      |
| **M004**    | 0           | 0             | 0         |

---

# ✅ TL;DR  
👉 **Batch 1**: No completed transactions → everything 0.  
👉 **Batch 2**: M001 gets 500 completed sales.  
👉 **Batch 3**: M001 refunded 500, M003 gets 1500 completed sales.  
✅ **Final Aggregation**: Matches your shared table!



---

# ✅ **Use Cases**
- **Real-time business reporting**  
- **Merchant dashboards**  
- **Refund and fraud detection**  
- **Live sales aggregation**  
- **Alerts and KPIs for merchants**  

---

# ✅ **Enhancements / Future Improvements**
- Add **window-based aggregations** (daily, weekly sales).
- Integrate **alerts** (e.g., high refunds).
- Sink data to **BI tools** (Power BI, Tableau, Looker).
- Handle **delete events** (hard deletes or soft deletes).
- Add **reconciliation** or **audit** tables for data quality.

---

# ✅ Summary
🎉 You've built a **real-time, CDC-driven data pipeline** using **Delta Lake CDC** and **Spark Structured Streaming**, ending in **aggregated insights** at the **merchant level**.

---

Let me know if you want a **diagram image** or help **writing a project report/README**!  
Want me to generate a **flow diagram image** for you?
