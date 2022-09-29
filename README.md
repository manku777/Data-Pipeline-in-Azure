# Data-Pipeline-in-Azure

# OBJECTIVE
  * Firstly, we want to create a data platform from which our data science team can run machine learning models to predict the spread of the virus and find other insights from the data.
 * Secondly, we want to create a data platform from which our data analysts can easily report on the covid-19 trends using a reporting tool.
 ![image](https://user-images.githubusercontent.com/87073324/192124679-065b5681-82a9-450e-857b-cb67342e1ddf.png)
![image](https://user-images.githubusercontent.com/87073324/192124694-a68d5a44-bffd-4446-ba60-33f76ca8e64b.png)
![image](https://user-images.githubusercontent.com/87073324/192124761-0e946900-00c5-4141-bc45-73a077444a7a.png)

# DATA SET
 * case_death, hospital_admission, testing, country_reponse, population(blob storage), DIM(lookup file)

# ARCHITECTURE
  ![image](https://user-images.githubusercontent.com/87073324/192125005-d58bd000-3dd7-4c2f-b3aa-37496009292c.png)


 * Create STORAGE ACCOUNTS (covidreportingsa), Azure Data Lake Gen2 STORAGE ACCOUNT (covidreportingdl) and Azure SQL Database. Create all three of them in the same resource group. Downloaded Azure Data studio and connnect with Azure SQL database for query(provide server name from SQL database)

# DATA INGESTION FROM AZURE BLOB(POPULATION BY AGE)
  ![image](https://user-images.githubusercontent.com/87073324/192128970-d6b04033-145d-42d6-a57a-05e057355557.png)
  ![image](https://user-images.githubusercontent.com/87073324/192129415-2b5212e3-5772-4f52-9bdd-53bcfcea27fd.png)

  
  * Population by age file has columns like age range, country and years from 2008 to 2019. 'PC_Y0_14,DE', here 0-14 is the age range and DE is the country 'Denmark'. Values under years column is in percentage i.e. percentage of population.
  Note: I have uploaded population.csv file in 'covidreportingsa' storage account, under 'population' container
  * Source : 
    Storage account : covidreportingsa
    Container : Population
    File : population_by_age.tsv.gz
  * Destination:
    Storage account : covidreportingdl
    Container : raw
    File : population_by_age.tsv
  
  * STEPS FOR DIFFERENT ACTIVITIES:
    * Open azure data factory -> Create LINKED SERVICE(we have to provide storage account name). Similary create a linked service for Azure data lake Gen2 and select storage account.
    * After creating LINKED SERVICE, I created the DATASETS for both azure blob storage(provide path of source file) and azure data lake Gen2(provide path for sink data). While creating we need to select the format of our source data set and sink data set. While creating I selected the linked service for both blob and ADLG2, this how connection between linked service and source & sink established. File format is csv for both source and sink
    * Now create a pipeline with COPY activity. While creating, provide the source and sink data set, which 
    
    * CONTROL FLOW ACTIVITIES - VALIDATION ACTIVITY  
    - Created a VALIDATION ACTIVITY which will check if there is any changes in the source data set file 'population_by_age.tsv.gz' for 2 mins, if its succeed, then COPY  ACTIVITY will be performed.
    - Created GET METADATA ACTIVITY and IF CONDITION ACTIVITY which will check if the number of columns, size are same as previous, if TRUE it will call COPY activity and if FALSE it will call WEB ACTIVITY which will send an email
    - Created DELETE acitivty, which will run after COPY activity and will delete the source file

      VALIDATION ACTIVITY(check if file exists) -> GET METADATA -> IF CONDITION ACTIVITY(if true COPY ACTIVITY and then DELETE ACTIVITY), else WEB ACTIVITY(email)
      
  * At last, I created a EVENT TRIGGER on 'covidreportingsa' storage account and 'population' container. We also need to provide Blob path name. Once we have created trigger, we need to attach this trigger to our pipeline(all activities above are under the pipeline)


# DATA INGESTION FROM HTTP(ECDC DATA)
  ![image](https://user-images.githubusercontent.com/87073324/192145181-465cc004-0c1b-4f57-a763-adce663f0dfb.png)

DATA SET: cases & death, hospital admission

 * Data ingestion requirements: New cases and death by country, hospital admission and ICU cases, Testing numbers, country response to Covid 19. CSV files used are cases & deaths 
 * Create HTTP LINKED SERVICE and provide the BASE URL(till .com) and a new source data set(HTTP) with format CSV file & relative URL. We'll also create the sink data set only(provide the linked service name of AZDLG2 and the path where csv file will be store). We're not creating sink LINKED SERVICE bcz it is already created 
earlier.
* Create a pipeline with COPY activity(cases & death csv) and provide the source data set and sink data set.
* Since we're extracting multiple files from ECDC, so using parameter and variable concept. Base url is same for both file and only relative URL is different. Similarly, in sink, container and folder name is same, only the name of the file under the folder is different
 
  Step1 : Create parameters in pipeline (sourceRelativeURL and sinkFileName) and default value should be blank
  Step2: Open the copy activity and provide the paremeters for 'relativeURL'(source) and 'fileName'(sink)
  
  Now when we will debug, it will ask for sourceRelativeURL and sinkFileName. We'll use schedule trigger to automate this. But this was still not feasible bcz we are creating four trigger for four files. 
* In order to improve this, I used LOOK UP and FOR EACH activity
  Step 1: Create the json file and provide the sourceBaseURL, sourceRelativeURL & sinkFileName. Upload this file by creating the 'config' folder under container in covidreportingsa
  Step 2: Create a dataset (Azure blob storage) with file format(JSON). 
  Step 3: Create Lookup -> ForEach(invoke COPY activity in it). We will then create a trigger and attach to the pipeline
  
  Note: JSON file contains sourceBaseURL, sourceRelativeURL and sinkFileName


# Data Flows - Cases & Deaths Data Transformation
  ![image](https://user-images.githubusercontent.com/87073324/192166010-26b2755d-0b41-4a48-a193-e5e042e558a1.png)

  Here we'll convert the raw covid and death csv file to transformed csv file. We'll add new column country code with 2 digit
  Note: In each of the transformation, we need source and sink. Also, I have create lookup csv file which contains 2 and 3 digit country code and uploaed in data lake(inside the looup folder)
  
  * Source Transformation -  Create a source and built a new data set in data flow by selecting azure blob storage which will take the cases_death csv file
  * Filter Transformation - Filter data only for the Europe since it contains data of other continent as well. In addition, few Europe continent has no country_code, so need to filter that as well. In VISUAL EXPRESSION BUILDER write expression -> continent == 'Europe' && not(isNull(country_code))
  * Select Transformation - It's use to filter the required columns only and remove/rename the extra columns. Deleted the extra columns like 
	continent, rate_14_day and date and added rule based mapping for date column in visual expression builder
	MATCHING: name == 'date'
	OUTPUT: ('reported' + '_date')
 * Pivot Transformation - Here I took cases death csv file and grouped by country, country code, population, source, reported data and PIVOTED the DAILY COUNT and indicator(confirmed, death etc). Renamed date to reported_date
 * Lookup tranformation: Lookup transformation is basically left join, so will join the result of our left side(PIVOT TRANSFORMATION) data with the country lookup source ON country column, it will have duplicate columns. Will select the appropriate columns from the SELECT column
 * Select transformation: We will use select transformation and remove the duplicate columns 
 * Sink transformation: Sink is like storing the data in the output file. So first we'll create a folder 'processed' in our data lake storage. After that, I will create a dataset(in data flow), select ADLS Gen2, select format as csv, linked service as for ADLS Gen2 and provide the file path of the 'processed' folder.

At last we will create a pipeline for executing the data flow and see the results coming out it. In our 'processed' folder, our csv file is splitted into around 200 csv file on distributed node BY DEFAULT but we stored everything in one csv file ONLY. 


# Data Flows - Hospital admission csv file
![image](https://user-images.githubusercontent.com/87073324/192541044-3537c02e-ff42-4f17-9d99-448abc169fef.png)
In the hospital_admission csv file the we have columns like country, indicator(daily hopital occupancy & ICU occupancy, weekly new hospital occupancy and ICU admission), date, year_weekly, value, source. One single file is having two different level of granularity(daily and weekly), its not a good approach, so we'll split the file into two different file. So will keep daily with daily file and weekly with weekly file and KEEP THIS AS COLUMN. Also, we will have new columns reported_week_start_date and reported_week_end_date

 * SOURCE TRANSFORMATION: Created a new data flow, selected ADLS Gen2, format as csv, linked service which we created for ADLS Gen2 and provided the path of hospital_admission csv file
 * SELECT TRANSFORMATION: Remove URL, rename date to reported_date and year_week to reported_year_week
 * LOOKUP TRANFORMATION: Performed lookup in the DIM file for retreving the two digit country code, three digit country code and population. LOOK UP will also bring unncessary columns like continent etc which we don't need. So will SELECT transformation after that to remove it.
 * SELECT TRANSFORMATION: Select the required columns and remove the duplicates
 * CONDITIONAL SPLIT: Through this we'll split the daily and weekly records. Provide the condition in the visual expression builder, will use INDICATOR column for this. Output will be two streams.
 
 Then we need to introduce two new columns reported_start_date and reported_end_date, we can do this by the help of dim_date file that we have seen previously. In our hospital_admission file the week year_week is in the form  '2020-W06' but in our dim_date file, it is not in the correct format. So in order to achive that in dim_date column we have two columns year(2020) and week_of_year(6), we will make it in the form of '2020-W06', source that we can join this with the hospital_admission file. We will do this by derive transformation. So we wil create another source and will select the file as dim_date.csv  
DIM_DATE file is under look up container in ADLS Gen 2.

* DERIVE TRANSFORMATIONS: It is used to either modify or transform date itself within the stream. With the help of this will transform the date in the format of 2020-W01. We will create new column ecdc_year_week('2020-W06') here. In derive transformation we can create new column or choose exisitng one. We created a column(ecdc_year_week) and will create a expression  -> year + '-w' + lpad(week_of_year,2,'0')
* AGGREGATE TRANSFORMATION: Here we will transform our data from the derive transformation to create one record per week, which will include the week start date and week end date. Also, new column dervied week column we create above. In aggregate transformation  we have GROUP BY and AGGREGATES. 
Here first we will group by ecdc_year_week and in Aggregation we will create new columns(week start date and week end date) and will find the min and max date of the ecdc_year_week. (We will remove the remove derive transformatiom from the steam later)
* JOIN TRANSFORMATION: We will do LEFT JOIN on Weekly split stream(reported year week) with the AGGREGATE TRANSFORMATION stream(ecdc_year_week) on the common column
* PIVOT TRANSFORMATION: GROUPED BY the weekly and daily stream by country, 2_digit country code, 3_digit country code, population, reported_date, source. PIVOT key will be the value of INDICATOR column. Both weekly and daily stream will have separate PIVOT transformation. For daily, PIVOT KEY will be Daily hospital occupancy and Daily ICU occupancy. For weekly, PIVOT KEY will be weekly hospital occupancy and weekly ICU occupancy. PIVOT column will be SUM of Daily hospital occupancy and Daily ICU occupancy(for daily stream) and weekly hospital occupancy and weekly ICU occupancy(for weekly stream)
* SORT TRANSFORMATION: Sorted weekly and daily date in DESCENDING ORDER and country in ASCENDING order
* SINK TRANSFORMATION: Created sink separately for daily and weekly by creating data set, selecting ADLS Gen2, format as csv. After that we'll create a ETL pipeline which will run the source data, perform the transformation and will load the data in the sink.
NOW in the processed folder in ADLS Gen2 we have three files, cases_death, hospital_admission_daily, hospital_admission_weekly


# Important Concept
 * Blob storage is used to store unstructed data like text, binary data, media, pdf etc
 * 'Parameters' are external values passed into pipelines, datasets or linked services. The value cannot be changed inside a pipeline
 * 'Variables' are internal values set inside a pipeline. The value can be changed inside the pipeline using set variable or append variable activity
 * When we create a dataset we need to provide the source system like blob storage, redhshift, S3 etc
 * Data factory submits data flows to Azure Databricks cluster and they run as spark jobs 

# Challenges
 * In the population.csv file, some percentage values are ':', '15.2e', '18.p'. Here 'p' stands for provisional, 'e' stands for estimate etc
 * Since I was extracted multiple csv files from ECDC website, creating different pipeline, data sets etc was not the feasible solution so I used Parameters & variables. Parameterized the Relative URL in source dataset and the file name in sink dataset 
 * Cases and death file has country's data other than Europe and has 3 digit country code. However, testing file has 2 digit country code
