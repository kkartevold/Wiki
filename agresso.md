<style>
h1{
    border-bottom: none;
}

hr { 
  display: block;
  margin-before: 0.5em;
  margin-after: 0.5em;
  margin-start: auto;
  margin-end: auto;
  overflow: hidden;
  border-style: solid;
  border-width: 5px;
  height: 0px;
  color: rgb(185, 205, 145);
}
</style>


<div align="right"><img width=300 src="eye-share_logo.png" />
<div align="right">www.eye-share.com</div>
<br />
<div align="right">
Eye-share AS<br/>
Maskinveien 15<br/>
4033 Stavanger</div>

<br /><br /><br /><br /><br /><br /><br /><br /><br /><br />

<h1><div style="color:rgb(0, 0, 0)" align="center">Eye-share</div></h1>
<hr />
<h1><div style="color:rgb(185, 205, 145)" align="center">UBW</div></h1>

<div align="left" class="page" />

<h2>Contents</h2>

- [Introduction](#introduction)
- [Integration methods](#integration-methods)
- [Supported modules](#supported-modules)
- [Requrements](#requrements)
- [Other functionality](#other-functionality)
- [Master data](#master-data)
  - [Chart of accounts](#chart-of-accounts)
  - [Account rule](#account-rule)
  - [Dimension rule](#dimension-rule)
  - [Vat](#vat)
  - [Payment group](#payment-group)
  - [Currency rate](#currency-rate)
  - [Employee](#employee)
  - [Supplier](#supplier)
  - [Allocation](#allocation)
  - [Dimension](#dimension)
- [Purchase order](#purchase-order)
  - [Product meta info](#product-meta-info)
- [Transfer to UBW](#transfer-to-ubw)
  - [Transfer service](#transfer-service)
    - [ABWTransaction](#abwtransaction)
    - [ServerProcessInput](#serverprocessinput)
  - [Cost invoice](#cost-invoice)
    - [ABWTransactionVoucher](#abwtransactionvoucher)
      - [ABWTransactionVoucherTransaction-Head](#abwtransactionvouchertransaction-head)
        - [ABWTransactionVoucherTransactionGLAnalysis-Head](#abwtransactionvouchertransactionglanalysis-head)
        - [ABWTransactionVoucherTransactionApArInfo-Head](#abwtransactionvouchertransactionaparinfo-head)
        - [ABWTransactionVoucherTransactionAmounts-Head](#abwtransactionvouchertransactionamounts-head)
      - [ABWTransactionVoucherTransaction-Line](#abwtransactionvouchertransaction-line)
        - [ABWTransactionVoucherTransactionGLAnalysis-Line](#abwtransactionvouchertransactionglanalysis-line)
        - [ABWTransactionVoucherTransactionAmounts-Line](#abwtransactionvouchertransactionamounts-line)
        - [ABWTransactionVoucherTransactionApArInfo-Line](#abwtransactionvouchertransactionaparinfo-line)
  - [Purchase invoice](#purchase-invoice)
  - [User Import](#user-import)
  - [Mandate import](#mandate-import)
    - [Amount limits](#amount-limits)
    - [Supported mandate dimensions](#supported-mandate-dimensions)

<div class="page" />

## Introduction
This document gives a brief overview over the integration between Eye-share and UBW. The integration consists of three main areas: 
- Import of master data (Suppliers, accounts, dimensions, account rules etc) 
- Transfer of invoices from Eye-share to UBW for payment 
- Matching of purchase orders

## Integration methods
UBW makes master data available for eye-share by exposing custom views using QueryEngine WebService.
eye-share transfers invoices to UBW by requesting a tranfer job using ExecuteServerProcessAsynchronously for a batch of ABWTransactionVouchers and GetResult to confirm their status.


## Supported modules

- Purchase order
- Cost invoice
  - Accrual, is done by selecting an allocation key on line
- General ledger
- Expense

## Requrements

- QueryEngine URL 
- Import URL 
- Username, password and company codes in UBW
- Dimensions in use
  - To be able to replicate master data for dimensions, eye-share has to know which dimensions to replicate and their attribute IDs
  - Example dimension Department is Dim1 in UBW with attribute ID C1

<div class="page" />

## Other functionality
- Cost invoice, expense, general ledger
  - Blocking of dimensions not allowed in account rules
  - Derrive default tax code from account rule
  - Derrive dimensions from account rule
- Purchase invoice
  - Lock line if LineStatus is not O

<div class="page" />

## Master data
We use http://services.agresso.com/QueryEngineService/QueryEngineV201101 for querying data.

This API uses a concept of templates which is a custom query making certain data available through a template name and -ID.

We use GetTemplateList to retrieve the template ID for a given type of data such as "Eye Kontoplan" or "Eye Fullmakter".
We then build a query using GetSearchCriteria.
Finally we retrieve data using GetTemplateResultAsDataSet.

<br /><br />

### Chart of accounts
- Template name: Eye Kontoplan

| UBW              | Type   | eye-share     | Type          | Description                                                                           |
| :---             | :---   |  :---         |  :---         | :---                                                                                  |
| account          | string | Key           | string        | Account number                                                                        |
| description      | string | Description   | string        | Account description                                                                   |
| status           | string | Status        | string        | Used for filtering active accounts, accounts with status "N" will show in eye-share   |
| head_account     | string | HeadAccount   | string        | Not currently in use                                                                  |
| account_rule     | int    | AccountRule   | Object        | Account rules are joined, see section [Account rule](#account-rule)                   |
| period_from      | int    | PeriodFrom    | DateTime      | Used for filtering active accounts, period on invoice must be greater than this date  |
| period_to        | int    | PeriodTo      | DateTime      | Used for filtering active accounts, period on invoice must be lesser than this date   |

UBW web service SQL: 

```SQL
select client, account, description, account_rule, head_account, period_from, period_to, status from aglaccounts
```

<div class="page" />

### Account rule
- Template name: Eye Konteringsregler

| UBW              | Type   | eye-share        | Type          | Description                                                                                      |
| :---             | :---   |  :---            |  :---         | :---                                                                                             |
| account_rule     | int    | RuleKey          | string        | Internal uniqe identifier                                                                        |
| rel_1_id         | string | Rel1Id           | string        | Not in use                                                                                       |
| status           | string | RuleStatus       | string        | Not in use                                                                                       |
| tax_code         | string | TaxCode          | string        | If tax flag is F, vat must be set to this fixed value                                            |
| tax_flag         | string | TaxFlag          | string        | If tax flag is M, vat is mandatory. If tax flag is F and tax code is not empty, vat is mandatory |
|                  |        | DimensionRules   | Object        | See section [Dimension rule](#dimension-rule)                                                    |

UBW web service SQL: 

```SQL
select account_rule, att_1_id, att_2_id, att_3_id, att_4_id, att_5_id, att_6_id, att_7_id, client, dim_1, dim_1_flag, dim_2, dim_2_flag, dim_3, dim_3_flag, dim_4, dim_4_flag, dim_5, dim_5_flag, dim_6, dim_6_flag, dim_7, dim_7_flag, matrix_id_cur, matrix_id_ts, matrix_id_tx, matrix_id_1, matrix_id_2, matrix_id_3, matrix_id_4, matrix_id_5, matrix_id_6, matrix_id_7, rel_cur_id, rel_tax_id, rel_ts_id, rel_1_id, rel_2_id, rel_3_id, rel_4_id, rel_5_id, rel_6_id, rel_7_id, status, tax_code, tax_flag, tax_system, user_id from aglrules 
```


<div class="page" />

### Dimension rule

| UBW             | Type   | eye-share        | Type          | Description                                                                         |
| :---            | :---   |  :---            |  :---         | :---                                                                                |
|                 |        | DimensionNumber  | string        | Dimension number 1-7                                                                |
|                 |        | DimensionName    | string        | Dimension name                                                                      |
| dim_{1-7}       | string | LockedDimension  | string        | Not in use                                                                          |
| att_{1-7}_id    | string | DimensionAttID   | string        | Used for mapping to eye-share field/dimension                                       |
| dim_{1-7}_flag  | string | DimensionFlag    | string        | Used for validating, M = Required values, O, V, F = Optional, null = Must be blank  |
| matrix_id_{1-7} | int    | DerivationFrom   | string        | Name of the dimension we should derrive from                                        |

<div class="page" />

### Vat

- Template name: Eye Avgiftskode

| UBW         | Type        | eye-share     | Type          | Description                                                                                                                                   |
| :---        | :---        |  :---         |  :---         | :---                                                                                                                                          |
| tax_code    | string      | Key           | string        | Vat code                                                                                                                                      |
| description | string      | Description   | string        | Vat description                                                                                                                               |
| var_pct     | double      | Percentage    | double        | Vat percent                                                                                                                                   |
| ap_flag     | int?        | ApFlag        | int?          | Value 1 when in use for supplier ledger. These will be shown for cost invoice plugin                                                          |
| gl_flag     | int?        | GlFlag        | int?          | Value 1 when in use for general ledger. These will be shown for general ledger plugin                                                         |
| date_from   | DateTime?   | PeriodFrom    | DateTime      | Used for filtering active vat codes, period on invoice must be greater than this date. If null, 6 months subtracted from today will be added. |
| date_to     | DateTime?   | PeriodTo      | DateTime      | Used for filtering active vat codes, period on invoice must be lesser than this date. If null, 6 months added from today will be added.       |


UBW web service SQL: 

```SQL
select client,tax_code,description,vat_pct,acc_vat from agltaxcode where status = 'N' and getdate() BETWEEN date_from and date_to and tc_type = 'N' and ((bflag/16) % 2 = 1 or (bflag/64) % 2 = 1) 
```

<div class="page" />

### Payment group
- Template name: Eye Valutadokument

Field with lookup on document headers(cost invoice/purchase invoice/expense/general ledger) 


| UBW         | Type        | eye-share     | Type          | Description            |
| :---        | :---        |  :---         |  :---         | :---                   |
| text1       | string      | Key           | string        |                        |
| description | string      | Description   | string        |                        |

UBW web service SQL: 

```SQL
select t.text1,t.description from aagsysvalues t where t.name = 'CURR_LICENCE' and t.sys_setup_code = 'NO'  
```

<div class="page" />

### Currency rate
- Template name: Eye Valutakurser

| UBW         | Type        | eye-share     | Type          | Description                                             |
| :---        | :---        |  :---         |  :---         | :---                                                    |
| currency    | string      | Key           | string        |   Currency code                                         |
| exch_rate   | double      | Rate          | double        |   Exchange rate                                         |
| date_from   | DateTime?   | FromDate      | DateTime      |   From date                                             |
|             |             | Factor        | enum          |   Seems like all currencies from UBW has a factor of 1  |

UBW web service SQL: 

```SQL
select r.client, r.currency, r.cur_unit, r.date_from, r.date_to, r.exch_rate from acrexchrates r where r.date_from = (select max(date_from) from acrexchrates r2 where r2.client =r.client and r2.currency = r.currency group by client, currency, cur_unit ) and cur_type = '1' 
```

<div class="page" />

### Employee
- Template name: Eye LeverandÃ¸r

Suppliers from UBW will be joined with users in eye-share by bank account. Users in eye-share having a bank account that matches a suppliers bank account will be created as an employee. (For use in travel and expense plugin)

| UBW           | Type        | eye-share     | Type          | Description                                   |
| :---          | :---        |  :---         |  :---         | :---                                          |
| apar_id       | string      | Key           | string        |   Supplier id                                 |
| apar_name     | string      | Description   | string        |   Supplier name                               |
| bank_account  | string      | BankAccount   | string        |   Used for mapping to a user in eye-share     |
|               |             | CompanyCode   | string        |   Which company does this employee belong to  |
|               |             | UserId        | Guid          |   Internal id of an user in eye-share         |

<div class="page" />

### Supplier
- Template name: Eye LeverandÃ¸r

| UBW           | Type        | eye-share           | Type          | Description                                                              |
| :---          | :---        |  :---               |  :---         | :---                                                                     |
| apar_id       | string      | Key                 | string        |   Supplier id                                                            |
| apar_name     | string      | Description         | string        |   Supplier name                                                          |
| bank_account  | string      | BankAccount         | string        |   Used to find correct supplier during import                            |
| country_code  | string      | CountryCode         | string        |   Used for derriving payment group on header                             |
| currency      | string      | Currency            | string        |   Used for derriving currency                                            |
| short_name    | string      | ShortName           | string        |   Not in use                                                             |
| terms_id      | string      | TermsId             | string        |   Not in use                                                             |
| status        | string      | Status              | string        |   Used for filtering. Suppliers with status N are available              |
| comp_reg_no   | string      | OrganizationNumber  | string        |   Internal id of an user in eye-share                                    |

UBW web service SQL: 

```SQL
select a.apar_id, a.apar_name, a.bank_account, a.comp_reg_no, a.currency, a.terms_id, a.short_name, a.status, a.country_code, c.client from asuheader a, acrclient c where a.client = c.pay_client and a.apar_gr_id NOT IN ('50','53','60') 
```

<div class="page" />

### Allocation
- Template name: Eye PeriodiseringsnÃ¸kler


| UBW           | Type        | eye-share     | Type          | Description                                   |
| :---          | :---        |  :---         |  :---         | :---                                          |
| no            | int         | Key           | int           |   Allocation key                              |
| description   | string      | Description   | string        |   Allocation description                      |

<br /><br />

### Dimension
- Template name: Eye Begrepsverdier

| UBW           | Type        | eye-share           | Type          | Description                                                                               |
| :---          | :---        |  :---               |  :---         | :---                                                                                      |
| dim_value     | string      | Key                 | string        |   Dimension code                                                                          |
| description   | string      | Description         | string        |   Dimension description                                                                   |
| attribute_id  | string      | AttributeId         | string        |   Used for mapping to correct field/dimension in eye-share                                |
| status        | string      | Status              | string        |   Dimensions with status N are available                                                  |
| rel_value     | string      | RelValue            | string        |   Not in use                                                                              |
| period_from   | int?        | PeriodFrom          | DateTime      |   Used for filtering active dimensions, period on invoice must be greater than this date  |
| period_to     | int?        | PeriodTo            | DateTime      |   Used for filtering active dimensions, period on invoice must be lesser than this date   |

UBW web service SQL: 

```SQL
select a.client, b.att_name, a.attribute_id, a.dim_value, a.description, b.dim_position, a.rel_value, a.period_from, a.period_to, a.status, a.value_1 FROM agldimvalue a, agldimension b where a.client = b.client and a.attribute_id = b.attribute_id and b.dim_position between '1' and '7' 
```

<div class="page" />

## Purchase order
We fetch purchase orders using UBW REST API.
URI: v1/purchase-orders/{poNumber}?companyId={companyId}

Only orders with OrderStatus O will be available in eye-share.

| UBW                       | Type    | eye-share                 | Type        | Description                                                           |
| :---                      | :---    |  :---                     |  :---       | :---                                                                  |
| WorkflowState             | string  | WorkflowState             | string      | Purchase orders with state N and T are possible to match in eye-share |
|                           |         | UseAmountNotUnits         | bool        | See section [Product meta info](#product-meta-info)                   |
| LineStatus                | string  | LineStatus                | string      | Only lines with status O is matchable in eye-share                    |
| ProductDescription        | string  | Description               | string      | Item description                                                      |
| HEADER.CurrencyCode       | string  | CurrencyCode              | string      | Currency of the order, must match with invoice currency               |
| HEADER.OrderNumber        | int     | OrderNumber               | string      | Purchase order number                                                 |
| HEADER.Responsible        | string  | Purchaser                 | string      | This user will get the invoice in eye-share if any mismatch           |
| Unit                      | string  | Unit                      | string      | Unit of meassure                                                      |
| LineExchangeRate          | float   | ExchangeRate              | double      | Used when transferring to UBW, calculation of postedInvoiceAmount     |
| Quantity                  | float?  | Ordered                   | double      | Number of items/services ordered                                      |
| DeliveredNumber           | float?  | Received                  | double      | Number of items/services received                                     |
| OriginalQuantity          | decimal?| Invoiced                  | double      | Number of items/services allready invoiced                            |
| PostedInvoiceAmount       | decimal | PostedInvoiceAmount       | double      | Amount allready invoiced                                              |
| UnitPrice                 | float   | OriginalPrice             | double      | Unit price                                                            |
| LineNumber                | int     | Line                      | int         | Line number                                                           |
| ProductId                 | string  | Item                      | string      | Item code                                                             |
| SellerProduct             | string  | SupplierItem              | string      | Suppliers item code                                                   |
| SellerProductDescription  | string  | SupplierItemDescription   | string      | Suppliers item description                                            |
| LineDimension1            | object  | LineDimension1            | object      |                                                                       |
| LineDimension2            | object  | LineDimension2            | object      |                                                                       |
| LineDimension3            | object  | LineDimension3            | object      |                                                                       |
| LineDimension4            | object  | LineDimension4            | object      |                                                                       |
| LineDimension5            | object  | LineDimension5            | object      |                                                                       |
| LineDimension6            | object  | LineDimension6            | object      |                                                                       |
| LineDimension7            | object  | LineDimension7            | object      |                                                                       |
| Account                   | string  | Account                   | object      |                                                                       |
| TaxCode                   | string  | Vat                       | object      |                                                                       |


### Product meta info
URI: v1/objects/products?companyId={companyId}

eye-share replicates products from UBW for use when fetching purchase order lines.
When fetching purchse order lines, eye-share checks if flag UseAmountNotUnits in collection product meta info is set for product on purchase order line.

| UBW               | Type      | eye-share           | Type   | Description                                                              |
| :---              | :---      |  :---               |  :---  | :---                                                                     |
| ProductId         | string    | Key                 | string |   Item code                                                              |
| UseAmountNotUnits | bool      | UseAmountNotUnits   | bool   |   Flag for if the product is a service or item. True=Service, False=Item |


<div class="page" />

## Transfer to UBW


### Transfer service
- Batches will be split per companycode.
- Batches will also be split on if exchange rate is overriden in eye-share.
- eye-share validates XML with XML schema
- eye-share checks if invoice exists in UBW before transferring

#### ABWTransaction
| UBW               | Type                      | eye-share           | Type   | Description                                                              |
| :---              | :---                      |  :---               |  :---  | :---                                                                     |
| BatchId           | string                    |                     |        |   Generated in eye-share                                                 |
| Interface         | string                    |                     |        |  Setting in eye-share, default "CW"                                      |
| ReportClient      | string                    |                     | string |  invoice companycode, batches will be split by companycode               |
| Voucher           | [ABWTransactionVoucher](#abwtransactionvoucher)[]   |                     |        |                                                                       |

#### ServerProcessInput
| UBW               | Type   | eye-share           | Type   | Description                                                              |
| :---              | :---   |  :---               |  :---  | :---                                                                     |
| ServerProcessId   | string |                     |        |  Fixed: GL07                                                             |
| MenuId            | string |                     |        |  Fixed: "BI88"                                      |
| Variant           | string |                     |        |  Setting in eye-share, default: 103               |
| ParameterList     | object |                     |        |  recalc_amt will be sent with value 1 if no invoices has overrided exchange rate in eye-share                           |
| Xml               | string |                     |        |  Serialized [ABWTransaction](#abwtransaction). If UBW is milestone 7 and later, xml will be wrapped in a CDATA section  |

<div class="page" />

### Cost invoice

#### ABWTransactionVoucher
| UBW               | Type      | eye-share           | Type     | Description                                                                          |
| :---              | :---      |  :---               |  :---    | :---                                                                                 |
| CompanyCode       | string    | CompanyCode         | string   |                                                                                      |
| Period            | string    | Period              | DateTime |   Format: yyyyMM                                                                     |
| VoucherDate       | DateTime  | InvoiceDate         | DateTime |                                                                                      |
| VoucherType       | string    |                     |          | Two settings in eye-share: VoucherTypeInvoice and VoucherTypeCredit, default: "CI"   |
| VoucherNo         | string    |                     |          | Fixed to 0, will be fetched from UBW at a later time                                 |
| Transaction       | object    |                     |          | [ABWTransactionVoucherTransaction-Head](#abwtransactionvouchertransaction-head) and [ABWTransactionVoucherTransaction-Line](#abwtransactionvouchertransaction-line) |


##### ABWTransactionVoucherTransaction-Head
| UBW             | Type      | eye-share   | Type    | Description                                           |
| :---            | :---      |  :---       |  :---   | :---                                                  |
| TransType       | string    |             |         | Fixed: AP                                             |
| Description     | string    | VoucherText | string  | Voucher text                                          |
| TransDate       | DateTime  |             |         | DateTime.Now                                          |
| AllocationKey   | string    |             |         | Fixed: 0                                              |
| ExternalRef     | string    |   Id        | Guid    | Document Id                                           |
| GLAnalysis      | object    |             |         | [ABWTransactionVoucherTransactionGLAnalysis](#abwtransactionvouchertransactionglanalysis-head)                     |
| ApArInfo        | object    |             |         | [ABWTransactionVoucherTransactionApArInfo](#abwtransactionvouchertransactionaparinfo-head)                         |
| Amounts         | object    |             |         | [ABWTransactionVoucherTransactionAmounts](#abwtransactionvouchertransactionamounts-head)                           |

###### ABWTransactionVoucherTransactionGLAnalysis-Head
| UBW             | Type      | eye-share   | Type    | Description         |
| :---            | :---      |  :---       |  :---   | :---                |
| Currency        | string    | Currency.Key| string  | Invoice currency    |
| TaxCode         | string    |             |         | Fixed: 0            |
| TaxId           | int       |             |         | Fixed: 0            |

###### ABWTransactionVoucherTransactionApArInfo-Head
| UBW            | Type       | eye-share         | Type      | Description                       |
| :---           | :---       |  :---             |  :---     | :---                              |
| DueDate        | DateTime   | DueDate           | DateTime  |                                   |
| ApArNo         | string     | Supplier.Key      | string    |                                   |
| ApArType       | enum       |                   |           | Fixed: P = supplier invoice       |
| InvoiceNo      | string     | InvoiceNumber     | string    |                                   |
| CurrLicense    | string     | PaymentGroup?.Key | string    |                                   |
| BacsId         | string     | CID               | string    |                                   |
| PayMethod      | string     |                   |           | Setting in eye-share, default: IP |

###### ABWTransactionVoucherTransactionAmounts-Head
| UBW             | Type      | eye-share   | Type    | Description                                                       |
| :---            | :---      |  :---       |  :---   | :---                                                              |
| CurrAmount      | string    | Amount      | double  | Format: 0.##   . as decimal separator                             |
| Amount          | string    |             |         | CurrencyRate * Amount, Format: 0.##   . as decimal separator      |



##### ABWTransactionVoucherTransaction-Line
| UBW             | Type      | eye-share   | Type    | Description                         |
| :---            | :---      |  :---       |  :---   | :---                                |
| ExternalRef     | string    | Id          | Guid    | Document Id                         |
| Description     | string    | Description | string  |                                     |
| TransType       | string    |             |         | Fixed: GL                           |
| AllocationKey   | int       |             |         | Fixed: 0                            |
| TransDate       | string    |             |         | DateTime.Now                        |
| GLAnalysis      | object    |             |         | [ABWTransactionVoucherTransactionGLAnalysis](#abwtransactionvouchertransactionglanalysis-line)   |
| Amounts         | object    |             |         | [ABWTransactionVoucherTransactionAmounts](#abwtransactionvouchertransactionamounts-line) |
| ApArInfo        | object    |             |         | [ABWTransactionVoucherTransactionApArInfo](#abwtransactionvouchertransactionaparinfo-line)                         |

###### ABWTransactionVoucherTransactionGLAnalysis-Line
| UBW             | Type      | eye-share   | Type    | Description         |
| :---            | :---      |  :---       |  :---   | :---                |
| Currency        | string    | Currency.Key| string  | Invoice currency    |
| Account         | string    | Account.Key | string  |                     |
| TaxId           | int       |             |         | Fixed: 0            |
| TaxCode         | string    |             |         | Setting in eye-share for m1 account, if m1 account is equal to line account, set m1 account tax code and m1 account tax system |
| TaxSystem       | string    |             |         | See field Taxcode. This will be set to empty string if m1 account and line account does not match                              |
| Dim1            | string    | AgrDim.Dim1 | string  |                     |
| Dim2            | string    | AgrDim.Dim2 | string  |                     |
| Dim3            | string    | AgrDim.Dim3 | string  |                     |
| Dim4            | string    | AgrDim.Dim4 | string  |                     |
| Dim5            | string    | AgrDim.Dim5 | string  |                     |
| Dim6            | string    | AgrDim.Dim6 | string  |                     |
| Dim7            | string    | AgrDim.Dim7 | string  |                     |
| PayMethod       | string    |             |         | Setting in eye-share, default: IP |


###### ABWTransactionVoucherTransactionAmounts-Line
| UBW             | Type      | eye-share             | Type    | Description                                                                    |
| :---            | :---      |  :---                 |  :---   | :---                                                                           |
| CurrAmount      | string    | Amount                | double  | Format: 0.##   . as decimal separator                                          |
| Amount          | string    |                       |         | CurrencyRate * Amount, Format: 0.##   . as decimal separator                   |
| Number1         | int       | AgressoAllocation.Key |         | Workaround bug in UBW: Send allocation key to number1 instead of ABWTransactionVoucherTransaction.AllocationKey |

###### ABWTransactionVoucherTransactionApArInfo-Line
| UBW            | Type       | eye-share         | Type      | Description                       |
| :---           | :---       |  :---             |  :---     | :---                              |
| DueDate        | DateTime   | DueDate           | DateTime  |                                   |
| ApArNo         | string     | Supplier.Key      | string    |                                   |
| ApArType       | enum       |                   |           | Fixed: P = supplier invoice       |
| InvoiceNo      | string     | InvoiceNumber     | string    |                                   |
| CurrLicense    | string     | PaymentGroup?.Key | string    |                                   |
| BacsId         | string     | CID               | string    |                                   |
| PayMethod      | string     |                   |           | Setting in eye-share, default: IP |

<div class="page" />

### Purchase invoice

We transfer purchase invoices using UBW Rest API. URI: v1/purchase-orders/{poNumber}?companyId={companyId}
Only lines with LineStatus = O and having an amount greater than 0 will be transferred.

To be able to update purchase orders in UBW, we use the HTTP verb PATCH. To be able to update correct lines, we use the index of a line in UBW.
During transfer we fetch all purchase order lines to get the correct indexes if there has been any changes in UBW since we fetched lines last time.

<br />

**There are five patches that are sent from eye-share to UBW Web service**
- originalQuantity
  - If UseAmountNotUnits is false (See section [Product meta info](#product-meta-info)) we update originalQuantity with Invoiced + InvoicedNow
- requisition
  - Field requisition is set to 1. This is to prevent workflow being triggered in UBW
- postedInvoiceAmount
  - Field postedInvoiceAmount is updated with what was allready in postedInvoiceAmount + (total line amount * line exchange rate)
- lineStatus
  - Field lineStatus is updated to F if whats ordered is lesser than or equal to whats invoiced now + allready invoiced.
- orderStatus
  - Field orderStatus is updated to F on purchase order if sum of invoiced + invoiced now for all lines is greater than sum of ordered on all lines.


**Transfer to GL07**<br />
- Not possible to have more than one PO on an invoice

eye-share use the same transfer methods as cost invoice. Additionally for purchase invoices, we send OrderNo on head and lines. (See section [Cost invoice](#cost-invoice))


<div class="page" />

### User Import

- Template name: Eye Brukere

| UBW               | Type       | eye-share   | Type      | Description                       |
| :---              | :---       |  :---       |  :---     | :---                              |
| user_id           | string     | Username    | string    |                                   |
| first_name        | string     | FirstName   | string    |                                   |
| surname           | string     | LastName    | string    |                                   |
| e_mail            | string     | Email       | string    |                                   |
| client            | string     | CompanyCode | string    |                                   |
| resource_typ      | string     | Active      | bool      |  If value 1 or 2 its an active user. |
| overstyr_tilgang  | string     | Active      | bool      | if resource_typ has value 7 and overstyr_tilgang is set to JA its an active user  |
| Manager           | string     | Superior    | string    |                                    |

```SQL
select a.first_name, a.surname, u.user_id, b.e_mail, a.client 
from aaguser u, ahsresources a 
left join agladdress b 
on  a.client = b.client and a.resource_id = b.dim_value and b.attribute_id = 'C0' and b.address_type = '1' and b.sequence_no = 0 
where a.client in ('NA','NR') 
and a.resource_id = b.dim_value 
and u.user_id = a.short_name 
```

<div class="page" />

### Mandate import

- Template name: Eye Fullmakter

#### Amount limits
- Amount limit 1: 50000
- Amount limit 2: 250000
- Amount limit 3: 1000000
- Amount limit 4: 15000000
- Amount limit 5: 999999999

#### Supported mandate dimensions
- Account
- Department
- Project



| UBW             | Type       | eye-share      | Type      | Description                       |
| :---            | :---       |  :---          |  :---     | :---                              |
| date_from       | DateTime   |                |           |   Only import mandate if this date is lesser or equal to todays date |
| date_to         | DateTime   |                |           |   Only import mandate if this date is greater or equal to todays date |
| user_id         | string     |  UserId        | string    |                                   |
| att_1_id        | string     |  Name          | string    |  We resolve this value to eye-share dimension name, see [Supported mandate dimensions](#supported-mandate-dimensions) |
| att_2_id        | string     |  Name          | string    |  We resolve this value to eye-share dimension name, see [Supported mandate dimensions](#supported-mandate-dimensions) |
| att_3_id        | string     |  Name          | string    |  We resolve this value to eye-share dimension name, see [Supported mandate dimensions](#supported-mandate-dimensions) |
| att_1_from      | string     |  From          | string    | From value  |
| att_2_from      | string     |  From          | string    | From value  |
| att_3_from      | string     |  From          | string    | From value  |
| att_1_to        | string     |  To            | string    | To value    |
| att_2_to        | string     |  To            | string    | To value    |
| att_3_to        | string     |  To            | string    | To value    |
| amount_limit1   | int        |   AmountLimit  |  int      | If value 1 this rule is applied, see [Amount limits](#amount-limits) |
| amount_limit2   | int        |   AmountLimit  |  int      | If value 1 this rule is applied, see [Amount limits](#amount-limits) |
| amount_limit3   | int        |   AmountLimit  |  int      | If value 1 this rule is applied, see [Amount limits](#amount-limits) |
| amount_limit4   | int        |   AmountLimit  |  int      | If value 1 this rule is applied, see [Amount limits](#amount-limits) |
| amount_limit5   | int        |   AmountLimit  |  int      | If value 1 this rule is applied, see [Amount limits](#amount-limits) |

```SQL
select amount_limit1, amount_limit2, amount_limit3, amount_limit4, amount_limit5, 'MNAL' AS att_1_id, att_1_from, att_1_to, 'C1' AS att_2_id, att_2_from, att_2_to, '' AS att_3_id, att_3_from, att_3_to, budget, client, date_from, date_to, delegate_flag, delegated_by, last_update, type, updated_by, user_id from egfullmaktdetail 
```