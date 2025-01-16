# courier-cost-validator

### README.md

# Freight Charge Analysis Tool

This Python-based solution analyzes and compares freight charges between Company X and its courier partners. The goal is to compute expected charges as per Company X’s standards and compare them with charges billed by courier companies.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Solution Overview](#solution-overview)
- [Steps and Code Breakdown](#steps-and-code-breakdown)
  - [1. Import Data](#1-import-data)
  - [2. Compute Total Weight](#2-compute-total-weight)
  - [3. Assign Delivery Zone](#3-assign-delivery-zone)
  - [4. Merge Data](#4-merge-data)
  - [5. Weight Slab Assignment](#5-weight-slab-assignment)
  - [6. Compute Expected Charges](#6-compute-expected-charges)
  - [7. Final Data Merging](#7-final-data-merging)
- [Expected Output](#expected-output)
- [How to Run](#how-to-run)
- [Author](#author)

---

## Prerequisites

Ensure the following Python libraries are installed:
- `pandas`
- `numpy`
- `openpyxl`

Install dependencies using:
```bash
pip install pandas numpy openpyxl
```

---

## Solution Overview

This script processes and analyzes multiple datasets to:
1. Compute the total weight of each order.
2. Determine the appropriate delivery zone.
3. Assign weight slabs based on company standards.
4. Calculate the expected freight charges and compare them with courier charges.
5. Generate a final report summarizing differences.

---

## Steps and Code Breakdown

### 1. Import Data
Load all required Excel files using `pandas`:
```python
order_report = pd.read_excel('path_to_order_report.xlsx')
pin_code_zone = pd.read_excel('path_to_pincode_zone.xlsx')
SKU_master = pd.read_excel('path_to_sku_master.xlsx')
Courier_Company_Invoice = pd.read_excel('path_to_courier_invoice.xlsx')
Courier_Company_Rates = pd.read_excel('path_to_courier_rates.xlsx')
Output_sample_summary = pd.read_excel('path_to_expected_result.xlsx', 'Summary')
Output_sample_calculation = pd.read_excel('path_to_expected_result.xlsx', 'Calculations')
```

### 2. Compute Total Weight
Merge order data with SKU data and calculate the total weight:
```python
Company_X = order_report.merge(SKU_master, on='SKU')
Company_X['Total weight as per X (KG)'] = Company_X['Order Qty'] * Company_X['Weight (g)'] / 1000
Company_X = Company_X.groupby(by=['ExternOrderNo']).sum().reset_index()
Company_X.rename({'ExternOrderNo': 'Order ID'}, axis=1, inplace=True)
```

### 3. Assign Delivery Zone
Match the order `Customer Pincode` with delivery zone information:
```python
dfa = pin_code_zone.drop_duplicates(subset=['Customer Pincode'])
dfb = Courier_Company_Invoice[['Order ID', 'Customer Pincode', 'Type of Shipment']]
pin_code = dfb.merge(dfa, on='Customer Pincode')
pin_code.rename({'Zone': 'Delivery Zone as per X'}, axis=1, inplace=True)
Company_X = Company_X.merge(pin_code, on='Order ID')
```

### 4. Merge Data
Combine relevant data columns:
```python
Courier_Company_Invoice.rename({'Charged Weight': 'Total weight as per Courier Company (KG)'}, axis=1, inplace=True)
Subset_Courier_Company = Courier_Company_Invoice[['AWB Code', 'Order ID', 'Total weight as per Courier Company (KG)', 'Zone', 'Billing Amount (Rs.)', 'Weight slab charged by Courier Company (KG)']]
merged_data = Company_X.merge(Subset_Courier_Company, on='Order ID', how='left')
```

### 5. Weight Slab Assignment
Define a function to assign weight slabs:
```python
def weight_slab(weight):
    i = round(weight % 1, 1)
    if i == 0.0:
        return weight
    elif i > 0.5:
        return int(weight) + 1.0
    else:
        return int(weight) + 0.5

Company_X['weight slab as per X (KG)'] = Company_X['Total weight as per X (KG)'].apply(weight_slab)
```

### 6. Compute Expected Charges
Calculate expected charges based on delivery zone and shipment type:
```python
total_Expected_charge = []
for i in range(len(Company_X)):
    fwd_category = 'fwd_' + Company_X['Delivery Zone as per X'][i]
    rto_category = 'rto_' + Company_X['Delivery Zone as per X'][i]
    fwd_fixed = Courier_Company_Rates[fwd_category + '_fixed'][0]
    rto_fixed = Courier_Company_Rates[rto_category + '_fixed'][0]
    fwd_additional = Courier_Company_Rates[fwd_category + '_additional'][0]
    rto_additional = Courier_Company_Rates[rto_category + '_additional'][0]
    
    weight = Company_X['weight slab as per X (KG)'][i]
    if Company_X['Type of Shipment'][i] == 'Forward charges':
        total_Expected_charge.append(fwd_fixed if weight <= 0.5 else fwd_fixed + (weight - 0.5) / 0.5 * fwd_additional)
    elif Company_X['Type of Shipment'][i] == 'Forward and RTO charges':
        total_Expected_charge.append(fwd_fixed + rto_fixed if weight <= 0.5 else fwd_fixed + (weight - 0.5) / 0.5 * (fwd_additional + rto_additional))

Company_X['Expected Charge as per X (Rs.)'] = total_Expected_charge
```

### 7. Final Data Merging
Generate the final report:
```python
merged_data = Company_X.merge(Subset_Courier_Company, on='Order ID', how='left')
merged_data['Difference Between Expected Charges and Billed Charges (Rs.)'] = merged_data['Expected Charge as per X (Rs.)'] - merged_data['Billing Amount (Rs.)']
```

---

## Expected Output
The final output contains a detailed comparison between expected and actual courier charges, including discrepancies. Columns include:
- `Order ID`
- `Expected Charge as per X (Rs.)`
- `Billing Amount (Rs.)`
- `Difference Between Expected Charges and Billed Charges (Rs.)`

---

## How to Run
1. Place all input Excel files in the correct paths.
2. Update file paths in the script.
3. Run the script in any Python IDE or Jupyter Notebook.

---

## Author
**Samruddhi Tambe**  
Feel free to reach out for any queries or improvements!
