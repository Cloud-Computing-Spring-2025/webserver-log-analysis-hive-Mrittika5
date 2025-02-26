# Web Server Log Analysis using Apache Hive

## **Project Overview**
This project implements a Hive-based analytical process for analyzing web server logs. The goal is to extract meaningful insights about website traffic patterns from a CSV dataset containing log entries. The project includes:
- Counting total web requests
- Analyzing HTTP status codes
- Identifying the most visited pages
- Performing traffic source analysis
- Detecting suspicious activities
- Analyzing traffic trends
- Implementing partitioning to optimize query performance

## **Step 1:  run docker compose up -d 
### **Open Hue and navigate to Query Editors → Hive**
1. **Create a database for the web server logs:**
   ```sql
   CREATE DATABASE IF NOT EXISTS web_logs;
   ```
2. **Use the database:**
   ```sql
   USE web_logs;
   ```

## **Step 2: Create External Table in Hive**
Since the dataset is in CSV format, create an external table pointing to an HDFS directory.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs_table (
    ip STRING,
    timest STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/web_logs/';
```
This table is stored in HDFS under `/user/hive/warehouse/web_logs/`.

## **Step 3: Load Data into HDFS**
1. Ensure the web server log file (`web_server_logs.csv`) is available on your local machine. or you can upload it in hues file manager
2. Upload it to HDFS using the command line (if using Docker, run this inside the container):
   ```sh
   hdfs dfs -mkdir -p /user/hive/warehouse/web_logs
   hdfs dfs -put web_server_logs.csv /user/hive/warehouse/web_logs/
   ```
3. Load data into Hive:
   ```sql
   LOAD DATA INPATH '/user/hive/warehouse/web_logs/web_server_logs.csv' INTO TABLE web_logs_table;
   ```

## **Step 4: Execute Analysis Queries**
### **Count Total Web Requests**
```sql
SELECT COUNT(*) AS total_requests FROM web_logs_table;
```

### **Analyze Status Codes**
```sql
SELECT status, COUNT(*) AS count FROM web_logs_table GROUP BY status;
```

### **Identify Most Visited Pages**
```sql
SELECT url, COUNT(*) AS visits 
FROM web_logs_table 
GROUP BY url 
ORDER BY visits DESC 
LIMIT 3;
```

### **Traffic Source Analysis**
```sql
SELECT user_agent, COUNT(*) AS count 
FROM web_logs_table 
GROUP BY user_agent 
ORDER BY count DESC 
LIMIT 3;
```

### **Detect Suspicious IPs**
```sql
SELECT ip, COUNT(*) AS failed_requests 
FROM web_logs_table 
WHERE status IN (404, 500) 
GROUP BY ip 
HAVING COUNT(*) > 3;
```

### **Analyze Traffic Trends**
```sql
SELECT SUBSTR(timest, 1, 16) AS time_minute, COUNT(*) AS requests 
FROM web_logs_table 
GROUP BY SUBSTR(timest, 1, 16) 
ORDER BY time_minute;
```

## **Step 5: Implement Partitioning**
```sql
CREATE TABLE web_logs_partitioned (
    ip STRING,
    timest STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

### **Insert Data into Partitioned Table**
```sql
INSERT OVERWRITE TABLE web_logs_partitioned PARTITION (status)
SELECT ip, timest, url, user_agent, status FROM web_logs_table;
```

## **Step 6: Export Results**
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/warehouse/output/'
SELECT * FROM web_logs_table;
```
```sh
hdfs dfs -get /user/hive/warehouse/output/ output.csv
```

## **Challenges Faced**
- **Data Formatting Issues:**
  - Solution: Ensured the CSV file was properly formatted and fields were correctly delimited.
- **Partitioning Optimization:**
  - Solution: Used partitioning by `status` to improve query performance.
- **HDFS Data Loading Errors:**
  - Solution: Verified file paths and used `hdfs dfs -ls` to check for successful uploads.

## **Sample Input and Output**
### **Sample Input (CSV file format)**
```
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0
```

### **Expected Output**
#### **Total Web Requests**
```
Total Requests: 100
```
#### **Status Code Analysis**
```
200: 80
404: 10
500: 10
```
#### **Most Visited Pages**
```
/home: 50
/products: 30
/checkout: 20
```
#### **Traffic Source Analysis**
```
Mozilla/5.0: 60
Chrome/90.0: 30
Safari/13.1: 10
```
#### **Suspicious IP Addresses**
```
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests
```
#### **Traffic Trends Over Time**
```
2024-02-01 10:15: 5 requests
2024-02-01 10:16: 7 requests
```

---
This project provides an efficient way to analyze web server logs using Apache Hive, helping to uncover key insights about website traffic and potential security threats.
