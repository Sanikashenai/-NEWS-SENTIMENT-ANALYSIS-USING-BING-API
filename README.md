#  NEWS SENTIMENT ANALYSIS USING BING API
This is end to end microsoft fabric project that analyze of sentiments of news

https://github.com/user-attachments/assets/48119434-2fe7-4f33-b8d2-e3be0db4de1c


STEP 1: convert csv to json using google sheets

![import](https://github.com/user-attachments/assets/6cf0cc03-0b15-4e28-ab10-d188bd820894)

after importing ,go to help and search app script edit code according to imported csv file

![image](https://github.com/user-attachments/assets/2e8310f0-4f22-4f69-861a-ae0290ad2381)

Json file is readyyy!!!

![image](https://github.com/user-attachments/assets/9ad45cdb-22a0-407e-9204-b4fccaf6db5a)

STEP2:Ingest data using Bing API
create resource in microsoft azure and search bing search & select Bing search API(Free) which returns feature of Bing News Search
ingested json file through rest api into bing API resource using Data Factory Pipeline 

![ingest](https://github.com/user-attachments/assets/b8d59d2b-808d-44b9-a67d-c739c7eef6fe)

STEP 3: Data Tranformation using 




