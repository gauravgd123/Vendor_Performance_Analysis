

# Vendor Sales Performance

### Dashboard Link : https://app.powerbi.com/links/ewbPeoAbU4?ctid=6716165f-2d41-437c-b927-752275b2a814&pbi_source=linkShare

## Problem Statement

To analyse vendor sales performance by evaluating key metrics such as total sales, revenue contribution, profit margins, and brand distribution. The goal is to identify top-performing and underperforming vendors, uncover sales trends, and provide actionable insights for improving vendor relationships, optimizing product assortment, and maximizing overall business profitability.


### Steps followed 

- Step 1 : Gathering the Data files required to perform analysis.
- Step 2 : Using Jupyter notebook to create connection with database(Microsoft SQL Server).

           server = 'GAURAV'  
           database = 'VendorSales' 
           driver = 'ODBC Driver 17 for SQL Server'   

           # Connection string
           connection_string = f"mssql+pyodbc:/ / {server}/{database}?driver={driver.replace(' ', '+')}"

           # Create SQLAlchemy engine
            engine = create_engine(connection_string)

- Step 3 : Importing data tables into database and perform necessary queries.
- Step 4 : Bringing the data tables to jupyter notebooks and performing cleaning and join operations to create a meaningful summary table.
          
        inspector = inspect(engine)
        tables = inspector.get_table_names()

        print("Tables in the database are: ")

        for table in tables:
        print(table)
- Step 5 : Various joins table used to merge the data of different tables. Joins used are:
           
        1- Join between purchases and purchase_prices

                pd.read_sql("""SELECT 
                p.VendorNumber,
                p.VendorName,
                p.Brand,
                p.PurchasePrice,
                pp.Volume,
                pp.Price AS ActualPrice,
                SUM(p.Quantity) AS TotalPurchaseQuantity,
                SUM(p.Dollars) AS TotalPurchaseDollars
                FROM purchases AS p
                JOIN purchase_prices AS pp
                ON p.Brand = pp.Brand
                where p.PurchasePrice>0
                GROUP BY p.VendorNumber, p.VendorName, p.Brand,p.PurchasePrice,pp.Volume,pp.Price
                ORDER BY TotalPurchaseDollars """, engine)
                
        2- Sales Table Group BY

                pd.read_sql("""Select
                VendorNo,
                Brand,
                SUM(SalesDollars) as TotalSalesDollars,
                SUM(SalesPrice) as TotalSalesPrice,
                SUM(SalesQuantity) as TotalSalesQuantity,
                SUM(ExciseTax) as TotalExciseTax
                from sales 
                GROUP BY VendorNo,Brand
                ORDER BY TotalSalesDollars""",engine)

        3- Final Summary table

        vendor_sales_summary = pd.read_sql("""WITH FreightSummary AS (
        SELECT
                VendorNumber,
                SUM(Freight) AS FreightCost
        FROM vendor_invoice
        GROUP BY VendorNumber
        ),

        PurchaseSummary AS (
        SELECT 
                p.VendorNumber,
                p.VendorName,
                p.Brand,
                p.Description,
                p.PurchasePrice,
                pp.Volume ,
                pp.Price AS ActualPrice,
                SUM(p.Quantity) AS TotalPurchaseQuantity,
                SUM(p.Dollars) AS TotalPurchaseDollars
        FROM purchases AS p
        JOIN purchase_prices AS pp
                ON p.Brand = pp.Brand
        WHERE p.PurchasePrice > 0
        GROUP BY 
                p.VendorNumber, p.VendorName, p.Brand, p.Description, p.PurchasePrice, pp.Price, pp.Volume
        ),

        SalesSummary AS (
        Select
                VendorNo,
                Brand,
                SUM(SalesQuantity) as TotalSalesQuantity,
                SUM(SalesDollars) as TotalSalesDollars,
                SUM(SalesPrice) as TotalSalesPrice,
                SUM(ExciseTax) as TotalExciseTax
        FROM sales 
        GROUP BY VendorNo,Brand
        )

        SELECT
        ps.VendorNumber,
        ps.VendorName,
        ps.Brand,
        ps.Description,
        ps.PurchasePrice,
        ps.ActualPrice,
        ps.Volume,
        ps.TotalPurchaseQuantity,
        ps.TotalPurchaseDollars,
        ss.TotalSalesQuantity,
        ss.TotalSalesDollars,
        ss.TotalSalesPrice,
        ss.TotalExciseTax,
        fs.FreightCost
        From PurchaseSummary ps
        LEFT JOIN SalesSummary ss
        ON ps.VendorNumber = ss.VendorNo
        AND ps.Brand = ss.Brand
        LEFT JOIN FreightSummary fs
        ON ps.VendorNumber = fs.VendorNumber
        ORDER BY ps.TotalPurchaseDollars DESC""",engine)

- Step 6 : Final Summary table created from the tables available which now will be used for analysis for vendor sales performance. 
Snap of newly created table

![Snap_1](https://github.com/user-attachments/assets/d264c388-36a8-444f-841d-f6b9cb1b2372)

- Step 7 : Further EDA(Exploratory Data Analysis) was performed to check parameters such as null values, data types etc. 

1 - Finding for null values:
        

![Snap_1](https://github.com/user-attachments/assets/b0fa2881-b584-47f7-a2b8-6279cb701429)

 - As we see Null values in 4 columns.We replace the null values with 0.

                - vendor_sales_summary.fillna(0, inplace = True)  

        
2 - Checking the data types of the data columns.

![Snap_Count](https://github.com/user-attachments/assets/3e79664f-e78b-40bf-bf0c-1511f1bbb9b1)

Some column such as Volume, TotalSalesQuantity have integer values but data types are not integer.
            
        Convertng Volume DataTypes: 

        vendor_sales_summary['Volume'] = vendor_sales_summary['Volume'].astype('float64')
        vendor_sales_summary['TotalSalesQuantity'] = vendor_sales_summary['TotalSalesQuantity'].astype('int')

 - Step 8 : Further calculated some new columns and added to our summary table.

 - Following columns were added:

# Gross profit

        vendor_sales_summary['GrossProfit'] =     vendor_sales_summary['TotalSalesDollars'] - vendor_sales_summary['TotalPurchaseDollars']

# Profit Margin

        vendor_sales_summary['ProfitMargin'] = (vendor_sales_summary['GrossProfit'] / vendor_sales_summary['TotalPurchaseDollars'])*100

# sale-purchase ratio
        vendor_sales_summary['SalePurchaseRatio'] = vendor_sales_summary['TotalSalesDollars']/vendor_sales_summary['TotalPurchaseDollars']

 
 Step 9 : Updated our final summary table with filtering to take only positive gross profit.

        # Updated vendor_sales_summary

        vendor_sales_summary = pd.read_sql("SELECT * FROM vendor_sales_summary  where GrossProfit>0 ", engine)
 
Step 10 : Describing the summary table.
 
 ![Snap_Percentage](https://github.com/user-attachments/assets/37459c6c-921b-49af-8113-31eb96be1eed)

 
 - Step 11 : The final summary table is pushed into the database. 

 - Step 12 : Now we import the summary table to Power BI from the Microsoft SQL server and transform our data using Power Query editor.
 
 - Step 13 : The final dashboard created using Power BI:
 
 ![Snap_3](https://github.com/user-attachments/assets/12ca79aa-ebba-437b-be9a-035c7825f3c2)
 
 - Step 14 : The report was then published to Power BI Service.
 
 
![Publish_Message](https://github.com/user-attachments/assets/878f9aaa-89da-4135-8ff5-e22a4db65416)


# Insights:

A single page report was created on Power BI Desktop & it was then published to Power BI Service.

Following inferences can be drawn from the dashboard;

 1 - Total Sales - 441 M

 2 - Gross Profit - 134 M

 3 - Average Profit Margin – 38.7 %

 4 - Unsold Quantity- 197K

 ### 5 - Top 10 Vendors by Purchase Quantity:
 
 ![Snap_3](https://github.com/user-attachments/assets/18b8759c-c372-42a6-8192-a220639a4dba)
 
 ### 6 - Top 10 Brands by Sales Performance:
 
  ![Snap_3](https://github.com/user-attachments/assets/1aec33f8-b7e3-42d9-99d2-e2abb7227d17)
         
### 7 - Top Vendors by Purchase Dollars : 

  ![Snap_3](https://github.com/user-attachments/assets/1aec33f8-b7e3-42d9-99d2-e2abb7227d17)

8 - Total Sales Quantity & Sales Dollars - Some products show zero sales, indicating they were purchased but never sold.

9 - Freight Cost: Extreme variation from 0.09 to 257,032 suggests logistics inefficiencies , bulk shipments or erratic shipping cost across different products.

10 - Top 10 Vendors Total Sales contribution to Total Vendors Sale: So around    99% of total sales dollars contribution came from top 10% Vendors and others contributed to 1%. This shows the dominance of certain Vendors and power distribution financially among them.


### Recommendations :

•	Diversify vendor partnerships to reduce dependency on a fewer supplier and mitigate supply chain risks.

•	Providing Bulk purchasing advantages to maintain competitive prices and encouraging more participations.

•	Enhance marketing and distribution strategies for low performing vendors to drive higher sales volumes without compromising profit margins

•	Managing high Freight Prices and tax related disparity to provide better margins to vendor.
