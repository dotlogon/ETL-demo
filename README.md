# ETL-demo
ETL Demo using Pentaho , mysql, SQLite and CSV
https://github.com/dotlogon/ETL-demo
The Ultimate PDI Workflow Build Guide (Step-by-Step, UI Focused)
This transformation is built around three parallel streams: POS Sales (GBP), Online Sales (EUR), and Market Rates (Lookup), which are then combined and processed.

Sample Workflow developed on Pentaho
 
________________________________________
a. Database connections (sqlite)
https://sqlite.org/download.html
sqlite-tools-win-x64-3500400.zip

PS D:\sqlite> .\sqlite3.exe D:\sqlite\pos_data.sqlite

-- Example DDL for your POS data
sqlite> CREATE TABLE IF NOT EXISTS pos_sales (
    Transaction_ID TEXT PRIMARY KEY,
    StoreID INTEGER,
    Date_of_Sale TEXT, -- SQLite often uses TEXT for dates
    Item_Sold TEXT,
    Amount_Paid REAL,
    Item_Cost_GBP REAL
);

sqlite> .tables

sqlite> INSERT INTO pos_sales (Transaction_ID, StoreID, Date_of_Sale, Item_Sold, Amount_Paid, Item_Cost_GBP)
VALUES ('POS2001', 12, '2025-10-25', 'Item X', 75.00, 40.00);

sqlite> INSERT INTO pos_sales (Transaction_ID, StoreID, Date_of_Sale, Item_Sold, Amount_Paid, Item_Cost_GBP)
VALUES ('POS2002', 15, '2025-10-26', 'Item Y', 200.00, 110.00);

sqlite> INSERT INTO pos_sales (Transaction_ID, StoreID, Date_of_Sale, Item_Sold, Amount_Paid, Item_Cost_GBP)
VALUES ('POS2003', 12, '2025-10-27', 'Item X', 75.00, 40.00);

sqlite> SELECT * FROM pos_sales LIMIT 5;

sqlite> .quit
Install Pentaho
https://pentaho.com/pentaho-developer-edition/#communityProducts
Enter any details and then you get download list, select below one and download.
pdi-ce-10.2.0.0-222.zip

D:\pentaho > Spoon.bat
In Pentaho UI , follow this.

File  New  Transformations  (Save the transformation immediately )
Design  Input   Table input  New
 

Connection type (sqlite)
Generic database
Custom connection URL:
jdbc:sqlite:D:/sqlite/pos_data.sqlite

Custum driver class name
org.sqlite.JDBCClick test to test connection

1. Phase 1: Extraction & Unification
A. POS Sales Stream (SQLite/GBP)
1.	Extract POS Sales (LiteSQL) (Table Input)
o	Double-click the step.
o	Select your SQLite database connection.
o	Enter your SELECT query (e.g., SELECT Sale_ID, Sale_Date, Product_Item, Amount, Cost FROM pos_sales;).
2.	Convert POS Date to Date Type (Select Values)
o	Meta-data Tab: Click Get fields to change.
o	Find the Sale_Date field.
o	Change the Type column value to Date. This is critical for joining.
 
3.	Standardize POS Fields (Select Values)
o	Select & Alter Tab: Click Get fields.
o	Ensure the fields are present in the order they will be merged (e.g., Sale_ID, Sale_Date, Product_Item, Amount, Cost).
 
4.	Add GBP Currency (Add Constants)
o	Add Field:
	Name: Source_Currency
	Type: String
	Value: GBP
 
B. Online Sales Stream (MySQL/EUR)
Important : Copy mysql-connector-j-9.5.0.jar to 
D:\pentaho\lib\ mysql-connector-j-9.5.0.jar
mysql details

Database etl_source_db 
user : etl_user
password : abcd 
mysql shell    --> from windows programs (after installing mysql )
 MySQL  JS > \connect root@localhost
<no password set>
MySQL  localhost:33060+ ssl  JS > \sql
DROP USER IF EXISTS 'etl_user'@'localhost';
CREATE USER 'etl_user'@'localhost' IDENTIFIED BY 'abcd';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE ON etl_source_db.* TO 'etl_user'@'localhost';
FLUSH PRIVILEGES;
SELECT host, user, authentication_string FROM mysql.user WHERE user = 'etl_user';

 MySQL  JS > \connect root@localhost
<no password set>
MySQL  localhost:33060+ ssl  JS > \sql
MySQL  localhost:33060+ ssl  SQL > show database;
MySQL  localhost:33060+ ssl  SQL > use etl_source_db
MySQL  localhost:33060+ ssl  SQL > show tables;
CREATE USER 'etl_user'@'localhost' IDENTIFIED BY 'abcd';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE ON etl_source_db.* TO 'etl_user'@'localhost';
FLUSH PRIVILEGES;

Populating Sample Data
-- Insert Command 1 (Sale on 2025-10-25)
INSERT INTO online_sales (Online_Sale_ID, Customer_ID, Sale_Date, Product_Name, Sale_Amount, Product_Cost) VALUES ('ON1001', 'CUST001', '2025-10-25', 'Widget A', 150.00, 80.00);
-- Insert Command 2 (Sale on 2025-10-25)
INSERT INTO online_sales (Online_Sale_ID, Customer_ID, Sale_Date, Product_Name, Sale_Amount, Product_Cost) VALUES ('ON1002', 'CUST002', '2025-10-25', 'Gadget B', 50.00, 20.00);
-- Insert Command 3 (Sale on 2025-10-26)
INSERT INTO online_sales (Online_Sale_ID, Customer_ID, Sale_Date, Product_Name, Sale_Amount, Product_Cost) VALUES ('ON1003', 'CUST003', '2025-10-26', 'Widget A', 150.00, 80.00);
select * from online_sales;

Create final FACT table. 

CREATE TABLE IF NOT EXISTS fact_profitability (Profit_Key INT PRIMARY KEY AUTO_INCREMENT,Sale_ID VARCHAR(20),Sale_Date DATE,Source_Currency VARCHAR(10), Product_Item VARCHAR(100),Sale_Amount_USD DECIMAL(10, 2),Profit_USD DECIMAL(10, 2),Currency_Rate_Used DECIMAL(5, 4)   );




Design  Input   Table input  New
 

Connection type
MySql

Access:
Nate(JDBC)
5.	Mysql online sales (Table Input)
o	Double-click the step.
o	Select your MySQL database connection.
o	Enter your SELECT query (e.g., SELECT Sale_ID, Sale_Date, Product_Item, Amount, Cost FROM online_sales;).
6.	Standardize Online Fields (Select Values)
o	Use the Select & Alter Tab to ensure the field names, types, and order exactly match the POS stream's fields (Step 3).
7.	Add EUR Currency (Add Constants)
o	Add Field:
	Name: Source_Currency
	Type: String
	Value: EUR
8.	Combine Sales Data (Append Streams)
o	Connect the output of Add GBP Currency (First Stream) and Add EUR Currency (Second Stream) to this step.
o	No configuration is needed if the field metadata matches.
________________________________________
2. Phase 2: Enrichment & Cleansing (The Fixes)
A. Market Rates Cleansing Path (The Lookup Stream)
9.	Load Market Rates (CSV) (CSV File Input)
o	Content Tab: Specify the file path for daily_rates.txt.
o	Fields Tab: Click Get Fields.
o	CRITICAL FIX: For the EUR_to_USD and GBP_to_USD fields, manually change the Type column value to String. This prevents the step from failing on the "Error" text.
10.	Convert Rates to String (Select Values)
o	Meta-data Tab: Confirm the GBP_to_USD field's Type is String.
11.	Cleanse GBP Rate (Replace in string)
o	CRITICAL FIX: Configure the step to replace the error text:
	In stream field: GBP_to_USD
	Out stream field: GBP_to_USD
	use RegEx: Y
	Search: ^.*Error.*$ (This matches "Error" regardless of hidden spaces or characters).
	Replace with: 1.20 (A valid numeric string for the fallback rate).
12.	Convert Rates to Number (Select Values)
o	CRITICAL FIX: Meta-data Tab:
	Find EUR_to_USD and change Type to Number.
	Find GBP_to_USD and change Type to Number.
o	This completes the lookup data preparation.
B. Order Control and Lookup
13.	Wait for Rates Load (Block this step until steps finish)
o	Draw Hop: Draw a hop from Combine Sales Data to this step (This is the Main Stream).
o	Configuration: Click Get steps.
	Ensure the only step listed is Convert Rates to Number. This forces the lookup data to finish loading before the sales data proceeds.
14.	Lookup Exchange Rate (Stream Lookup)
o	Draw Hops (Order is Critical!):
	Hop 1 (Lookup Stream): From Convert Rates to Number to Lookup Exchange Rate. (Draw this first).
	Hop 2 (Main Stream): From Wait for Rates Load to Lookup Exchange Rate.
o	Configuration:
	Lookup step: Select Convert Rates to Number.
	Key(s) to look up the value(s):
	Field (Main Stream): Sale_Date
	LookupField (Lookup Stream): market_date
	Specify the fields to retrieve: Click Get lookup fields and select EUR_to_USD and GBP_to_USD.
________________________________________
3. Phase 3: Transformation & Load
15.	Filter by Currency (Filter Rows)
o	Condition: Source_Currency = EUR (Select the (String) option).
o	True Destination: Calc EUR to USD
o	False Destination: Calc GBP to USD
16.	Calc EUR to USD / Calc GBP to USD (Calculator)
o	New field: Sale_Amount_USD
o	Calculation: A * B
o	Field A: Amount
o	Field B (EUR): EUR_to_USD
o	Field B (GBP): GBP_to_USD
17.	Recombine Calculated Data (Append Streams)
o	Merge the outputs of the two Calc... steps.
18.	Final Field Selection (Select Values)
o	Select & Alter Tab: Rename: Find Source_Currency and set Rename to as Source_System (if that's the name in your MySQL table).
o	Remove Tab: Click Get fields to remove and select the now-redundant fields: EUR_to_USD and GBP_to_USD.
19.	Load Fact Profitability (Table Output)
o	Configuration: Select your MySQL database connection and the target table, fact_profitability.
o	Database Fields Tab: CRITICAL FINAL FIX: Click Get fields. Manually verify that the column names in the Table field column exactly match the names in the Stream field column (e.g., ensure Source_System in the table maps to the renamed field from Step 18).

