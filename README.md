# KORE SSIS Home Task

## Overview:
This project uses the detailed schemas for a “Users” table by reading data from a .csv file, cleaning its contents in a staging table and then populating a corresponding production table with data from the staging table. Data cleaning consisted of converting data to the types detailed in the schema, sorting valid and invalid data and adjusting/moving data rows that were semantically incorrect. This project utilizes SSIS to handle control and data flow tasks, managing SQL queries that would perform the process’s data cleaning and migration.

## Setup:
This project required Microsoft Visual Studio in order to create the SSIS package that extracted data, clean the data up and migrate it to the database. This involves installing Visual Studio’s “Data storage and processing” toolset to be able to create SSIS solutions with the SSIS toolbox.

As well, Microsoft SQL Server Management Studio was used to host a database that the SSIS package could communicate and work with.

Additionally, Google Sheets was used to take the sample .csv data from the task description .pdf and create the .csv used in the implementation of this project. Of course, this is not required given that there is already a .csv file that can be used for the SSIS package.

## Approach/Methodologies:
### Extraction:
Data is extracted from a csv file as a flat file Source within the SSIS data flow. Outputted data from the csv is of data type unicode string to account for potentially invalid pieces of data being read that may be NULL or contain incorrect data types.
The outputted data is then run through data conversion in the data flow and is converted to the correct data type according to the given schema. Data conversion is only applied to columns that aren't of NVARCHAR in the schema.
Data that is unable to be processed in data conversion is redirected and isolated to a table called "Invalids". This would primarily be due to being incorrect data types from the source file that prevent data conversion.
Data that is successfully converted is pushed to the staging table.

### Data Cleaning:
Data after extraction must be cleaned or isolated to the invalids table if it is semantically incorrect and must be reviewed. Rows are cleaned or isolated based on the following criteria for each column:
UserID
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Move to invalids |

*UserID is converted to a four-byte unsigned integer, thus automatically moving UserIDs < 0 to invalids in data conversion. Therefore, this case was eliminated from consideration in the staging process.*

FullName
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Move to invalids |
| Not a full name | “Johnny” | Move to invalids |

Age
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Move to invalids |

*Age is converted to a single-byte unsigned integer, thus automatically moving Ages < 0 to invalids in data conversion. Therefore, this case was eliminated from consideration in the staging process.*

Email
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Move to invalids |
| Does not contain “@” | “notanemail” | Move to invalids |

RegistrationDate
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Move to invalids |
| Date being older than user (i.e. a really old date) | Age = 90, RegistrationDate = 1920-01-01 | Move to invalids |

LastLoginDate
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Move to invalids |
| LastLoginDate earlier than RegistrationDate | LastLoginDate = 1/1/2000, RegistrationDate = 1/1/2020 | Move to invalids |
| LastLoginDate in the future | 1/1/2050 | Move to invalids |

PurchaseTotal
| Case | Example | Action  |
| ---- |---------| --------|
| NULL value | NULL | Set to 0. |

Duplicate rows are all deleted, except for one. Since the only difference between truly duplicate rows was the PurchaseTotal value, it was assumed that this column could be derived to be a running sum of all PurchaseTotal values for each UserID. Therefore, this running sum would be the new value for PurchaseTotal after deleting all other duplicates.

## Load to Production Table
By this point, problematic roles have been either adjusted or migrated to the “Invalids” table under the specifications listed above. The remaining rows in the staging table are migrated to the production table. Rows that had UserIDs that did not already exist in the production table are simply inserted in. Rows that had UserIDs that matched pre existing UserIDs in the production table replaced the row in the production table that had the same UserID.
