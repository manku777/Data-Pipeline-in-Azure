# Data-Pipeline-in-Azure

# OBJECTIVE
  * Firstly, we want to create a data platform from which our data science team can run machine learning models to predict the spread of the virus and find other insights from the data.
 * Secondly, we want to create a data platform from which our data analysts can easily report on the covid-19 trends using a reporting tool.
 ![image](https://user-images.githubusercontent.com/87073324/192124679-065b5681-82a9-450e-857b-cb67342e1ddf.png)
![image](https://user-images.githubusercontent.com/87073324/192124694-a68d5a44-bffd-4446-ba60-33f76ca8e64b.png)
![image](https://user-images.githubusercontent.com/87073324/192124761-0e946900-00c5-4141-bc45-73a077444a7a.png)

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
  
  * STEPS FOR COPY ACTIVITY:
    * Open azure data factory -> Create linked service(we have to provide storage account name). Similary create a linked service for Azure data lake Gen2 and select storage account.
    * After creating LINKED SERVICE, I created the datasets for both azure blob storage(provide path of source file) and azure data lake Gen2(provide path for sink data). While creating we need to select the format of our source data set and sink data set. While creating I selected the linked service for both blob and ADLG2, this how connection between linked service and source & sink established. File format is csv for both source and sink
    * Now create a pipeline with COPY activity. While creating, provide the source and sink data set, which awqqw 













# Important Concept
 * Blob storage is used to store unstructed data like text, binary data, mediam pdf etc

# Challenges
 * In the population.csv file, some percentage values are ':', '15.2e', '18.p'. Here 'p' stands for provisional, 'e' stands for estimate etc
