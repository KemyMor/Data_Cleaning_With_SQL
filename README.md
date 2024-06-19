##  Data_Cleaning_With_SQL
- SQL Project - Data Cleaning
The following steps will be taken for the data cleaning process;
 1. check for duplicates and remove any
 2. standardize data and fix errors
 3. Look at null values and see what 
 4. remove any columns and rows that are not necessary - few ways

# Step 1
First thing is to create a staging table. 
This is the one we will work in and clean the data. We want a table with the raw data in case something happens
CREATE TABLE [dbo].[Nashville_Housing_staging2](
	[UniqueID] [int] NOT NULL,
	[ParcelID] [nvarchar](50) NOT NULL,
	[LandUse] [nvarchar](50) NOT NULL,
	[PropertyAddress] [nvarchar](50) NULL,
	[SaleDate] [date] NOT NULL,
	[SalePrice] [money] NOT NULL,
	[LegalReference] [nvarchar](50) NOT NULL,
	[SoldAsVacant] [bit] NOT NULL,
	[OwnerName] [nvarchar](100) NULL,
	[OwnerAddress] [nvarchar](50) NULL,
	[Acreage] [float] NULL,
	[TaxDistrict] [nvarchar](50) NULL,
	[LandValue] [int] NULL,
	[BuildingValue] [int] NULL,
	[TotalValue] [int] NULL,
	[YearBuilt] [smallint] NULL,
	[Bedrooms] [tinyint] NULL,
	[FullBath] [tinyint] NULL,
	[HalfBath] [tinyint] NULL,
	[row_num] [int] 
) ON [PRIMARY]
GO

INSERT INTO Nashville_Housing_staging2
SELECT *,
ROW_NUMBER() OVER (
PARTITION BY UniqueID, ParcelID, LandUse, PropertyAddress, SaleDate, SalePrice, 
	LegalReference, SoldAsVacant, OwnerAddress, Acreage, TaxDistrict, LandValue, BuildingValue,
	TotalValue, YearBuilt, Bedrooms, FullBath, HalfBath 
ORDER BY (SELECT NULL)) AS row_num
FROM Nashville_Housing_staging;
    OR
SELECT *
INTO Nashville_Housing_staging2
FROM Nashville_Housing_staging
WHERE 1 = 0;
# Inserting data
INSERT INTO Nashville_Housing_staging2 (UniqueID, ParcelID, LandUse, PropertyAddress, SaleDate, SalePrice, LegalReference, SoldAsVacant, OwnerName, OwnerAddress, Acreage, TaxDistrict, LandValue, BuildingValue, TotalValue, YearBuilt, Bedrooms, FullBath, HalfBath)
SELECT UniqueID, ParcelID, LandUse, PropertyAddress, SaleDate, SalePrice, LegalReference, SoldAsVacant, OwnerName, OwnerAddress, Acreage, TaxDistrict, LandValue, BuildingValue, TotalValue, YearBuilt, Bedrooms, FullBath, HalfBath
FROM Nashville_Housing_staging;

Select * from Nashville_Housing_staging2

# Step 2: Remove Duplicates
let's check for duplicates and remove any
**a. filter by row-num**
SELECT *,
ROW_NUMBER() OVER(
    PARTITION BY UniqueID, ParcelID, LandUse, PropertyAddress, SaleDate, SalePrice, 
	LegalReference, SoldAsVacant, OwnerAddress, Acreage, TaxDistrict, LandValue, BuildingValue,
	TotalValue, YearBuilt, Bedrooms, FullBath, HalfBath 
    ORDER BY (SELECT NULL)) AS row_num
FROM Nashville_Housing_staging;
N/B: row_num with (2 or more) indicates the number of duplicates. if its 1, it means no duplicate---

**b. finding the duplicates across columns.** 
WITH duplicate_cte AS
(
SELECT *,
ROW_NUMBER() OVER (
PARTITION BY UniqueID, ParcelID, LandUse, PropertyAddress, SaleDate, SalePrice, 
	LegalReference, SoldAsVacant, OwnerAddress, Acreage, TaxDistrict, LandValue, BuildingValue,
	TotalValue, YearBuilt, Bedrooms, FullBath, HalfBath 
ORDER BY (SELECT NULL)) AS row_num
FROM Nashville_Housing_staging
) 
SELECT *
FROM duplicate_cte
WHERE row_num > 1;

**3. confirm the duplicates.**
SELECT *
FROM Nashville_Housing_staging
WHERE SoldAsVacant = ' NULL' ;

**4. After identifying the duplicates, its not possible to delete from a CTE Statement.**
-----so follow this steps;
-----a. create a new table; right click on existing table, click on 'script table as' 
-----then select 'create to' then pick 'new query editor window' options. 
-----this bring up the create table query, then include the new coloum it----
CREATE TABLE [dbo].[Layoffs_staging2](
	[company] [nvarchar](50) NULL,
	[location] [nvarchar](50) NULL,
	[industry] [nvarchar](50) NULL,
	[total_laid_off] [nvarchar](50) NULL,
	[percentage_laid_off] [nvarchar](50) NULL,
	[date] [nvarchar](50) NULL,
	[stage] [nvarchar](50) NULL,
	[country] [nvarchar](50) NULL,
	[funds_raised_millions] [nvarchar](50) NULL,
	[row_num] [INT]----new column---
) ON [PRIMARY]
GO

**5. Insert CTE data into the table.**
INSERT INTO Layoffs_staging2
SELECT *,
ROW_NUMBER() OVER (
PARTITION BY company,location, industry, total_laid_off, percentage_laid_off, date, stage, country, funds_raised_millions
ORDER BY (SELECT NULL)) AS row_num
FROM Layoffs_staging;

**6. after creating the new table with the row_num(filter).**
-----you can now proceed to delete the duplicates---
DELETE
FROM Layoffs_staging2
WHERE row_num > 1;

**After checking all duplicates, you can drop the row_num column.**
One solution, which I think is a good one. Is to create a new column and add those row numbers in. Then delete where row numbers are over 2, then delete that column
-- so let's do it!!

ALTER TABLE Nashville_Housing_staging2
DROP COLUMN row_num;

# Step 2: Standardize Data and fix errors; this implies finding issues in the datasets.
-----i. trim the excess space before each data amd after each data----
SELECT * 
FROM Nashville_Housing_staging2;


**2.I also noticed the Residential has multiple different variations. We need to standardize that.**
SELECT DISTINCT industry
FROM Nashville_Housing_staging2
ORDER BY industry;

SELECT *
FROM Nashville_Housing_staging2
WHERE LandUse LIKE 'RESIDENTIAL CO%';

UPDATE Nashville_Housing_staging2
SET LandUse = 'RESIDENTIAL CONDO'
WHERE LandUse LIKE 'RESIDENTIAL CO%'; 

SELECT *
FROM Nashville_Housing_staging2
WHERE OwnerAddress IS NULL---- = 'NULL';
AND PropertyAddress IS NULL;

DELETE
FROM Nashville_Housing_staging2
WHERE PropertyAddress IS NULL
AND OwnerAddress IS NULL;

**we should set the blanks to nulls since those are typically easier to work with.**
SELECT *
FROM Nashville_Housing_staging2
WHERE OwnerAddress IS NULL---- = 'NULL';
AND PropertyAddress IS NULL;

DELETE
FROM Nashville_Housing_staging2
WHERE PropertyAddress IS NULL
AND OwnerAddress IS NULL;

-- we should set the blanks to nulls since those are typically easier to work with
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';

-- now if we check those are all null


SELECT 
    address AS original_address,
    LEFT(address, CHARINDEX(',', address) - 1) AS street,
    SUBSTRING(address, CHARINDEX(',', address) + 2, CHARINDEX(',', address, CHARINDEX(',', address) + 1) - CHARINDEX(',', address) - 2) AS city,
    SUBSTRING(address, CHARINDEX(',', address, CHARINDEX(',', address) + 1) + 2, CHARINDEX(',', address, CHARINDEX(',', address, CHARINDEX(',', address) + 1) + 1) - CHARINDEX(',', address, CHARINDEX(',', address) + 1) - 2) AS state,
    SUBSTRING(address, CHARINDEX(',', address, CHARINDEX(',', address, CHARINDEX(',', address) + 1) + 1) + 2, LEN(address)) AS zip_code
FROM your_table;

 **Breaking out Address into Individual Columns (Address, City, State)**
Select *
From Nashville_Housing_staging2
Where PropertyAddress is null
order by ParcelID

SELECT
SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1 ) as Address
, SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) + 1 , LEN(PropertyAddress)) as Address
From Nashville_Housing_staging2;

ALTER TABLE Nashville_Housing_staging2
Add PropertyCity Nvarchar(255);

Update Nashville_Housing_staging2
SET PropertyCity = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) + 1 , LEN(PropertyAddress))

ALTER TABLE Nashville_Housing_staging2
ADD PropertyAddressNew Nvarchar(255);

UPDATE Nashville_Housing_staging2
SET PropertyAddressNew = SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1)

Select OwnerAddress
From Nashville_Housing_staging2;

Select
PARSENAME(REPLACE(OwnerAddress, ',', '.') , 3) AS OwnerStreetAddress
,PARSENAME(REPLACE(OwnerAddress, ',', '.') , 2) AS City
,PARSENAME(REPLACE(OwnerAddress, ',', '.') , 1) AS State
From Nashville_Housing_staging2;

ALTER TABLE Nashville_Housing_staging2
Add OwnerStreetAddress Nvarchar(255);

Update Nashville_Housing_staging2
SET OwnerStreetAddress = PARSENAME(REPLACE(OwnerAddress, ',', '.') , 3)


ALTER TABLE Nashville_Housing_staging2
Add OwnerCity Nvarchar(255);

Update Nashville_Housing_staging2
SET OwnerCity = PARSENAME(REPLACE(OwnerAddress, ',', '.') , 2)

ALTER TABLE Nashville_Housing_staging2
Add OwnerState Nvarchar(255);

Update Nashville_Housing_staging2
SET OwnerState = PARSENAME(REPLACE(OwnerAddress, ',', '.') , 1)

**Change Y and N to Yes and No in "Sold as Vacant" field.**

Select Distinct(SoldAsVacant), Count(SoldAsVacant)
From Nashville_Housing_staging2
Group by SoldAsVacant
order by 2;

Select SoldAsVacant
, CASE When SoldAsVacant = 'Y' THEN 'Yes'
	   When SoldAsVacant = 'N' THEN 'No'
	   ELSE SoldAsVacant
	   END
From Nashville_Housing_staging2;

SELECT SoldAsVacant,
    CASE 
        WHEN SoldAsVacant = 1 THEN 'Yes'
        WHEN SoldAsVacant = 0 THEN 'No'
        ELSE 'Unknown'
    END AS FormattedSoldAsVacant
FROM Nashville_Housing_staging2;

Update Nashville_Housing_staging2
SET SoldAsVacant = CASE When SoldAsVacant = '1' THEN '1'
	   When SoldAsVacant = '0' THEN '0'
	   ELSE 'Unknown'
	   END;

## Step 4: remove any columns and rows we dont need to
Delete Unused Columns

Select *
From Nashville_Housing_staging2

ALTER TABLE Nashville_Housing_staging2
DROP COLUMN OwnerAddress, TaxDistrict, PropertyAddress

Select a.ParcelID, a.PropertyAddressNew, b.ParcelID, b.PropertyAddressNew, ISNULL(a.PropertyAddressNew,b.PropertyAddressNew)
From Nashville_Housing_staging2 a
JOIN Nashville_Housing_staging2 b
	on a.ParcelID = b.ParcelID
	AND a.[UniqueID ] <> b.[UniqueID ]
Where a.PropertyAddressNew is null;


Update a
SET PropertyAddressNew = ISNULL(a.PropertyAddressNew,b.PropertyAddressNew)
From Nashville_Housing_staging2 a
JOIN Nashville_Housing_staging2 b
	on a.ParcelID = b.ParcelID
	AND a.[UniqueID ] <> b.[UniqueID ]
Where a.PropertyAddressNew is null
SELECT *
FROM world_layoffs.layoffs_staging2;

-- everything looks good except apparently we have some "United States" and some "United States." with a period at the end. Let's standardize this.

SELECT * 
FROM world_layoffs.layoffs_staging2;
