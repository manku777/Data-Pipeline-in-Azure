# Data-Pipeline-in-Azure

# OBJECTIVE
  * Firstly, we want to create a data platform from which our data science team can run machine learning models to predict the spread of the virus and find other insights from the data.
 * Secondly, we want to create a data platform from which our data analysts can easily report on the covid-19 trends using a reporting tool.
 ![image](https://user-images.githubusercontent.com/87073324/192124679-065b5681-82a9-450e-857b-cb67342e1ddf.png)
![image](https://user-images.githubusercontent.com/87073324/192124694-a68d5a44-bffd-4446-ba60-33f76ca8e64b.png)
![image](https://user-images.githubusercontent.com/87073324/192124761-0e946900-00c5-4141-bc45-73a077444a7a.png)

# DATA SET
 * case_death, hospital_admission, testing, country_reponse, population(blob storage)

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
