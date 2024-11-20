![image](https://github.com/user-attachments/assets/a41b9cfe-6d66-4951-bb3a-8a69e132096e)#  NEWS SENTIMENT ANALYSIS USING BING API
This is end to end microsoft fabric project that analyze of sentiments of news

https://github.com/user-attachments/assets/48119434-2fe7-4f33-b8d2-e3be0db4de1c


# STEP 1: convert csv to json using google sheets

![import](https://github.com/user-attachments/assets/6cf0cc03-0b15-4e28-ab10-d188bd820894)

after importing ,go to help and search app script edit code according to imported csv file

![image](https://github.com/user-attachments/assets/2e8310f0-4f22-4f69-861a-ae0290ad2381)

Json file is readyyy!!!

![image](https://github.com/user-attachments/assets/9ad45cdb-22a0-407e-9204-b4fccaf6db5a)

#STEP2:Ingest data using Bing API
create resource in microsoft azure and search bing search & select Bing search API(Free) which returns feature of Bing News Search
ingested json file through rest api into bing API resource using Data Factory Pipeline 

![ingest](https://github.com/user-attachments/assets/b8d59d2b-808d-44b9-a67d-c739c7eef6fe)

#STEP 3: Data Tranformation using Synapse Notebooks 

df = spark.read.option("multiline", "true").json("Files/bing-latest-news.json")

df now is a Spark DataFrame containing JSON data from "Files/bing-latest-news.json".
display(df)

![image](https://github.com/user-attachments/assets/75350b5b-4029-47d9-bbd6-725c917960f6)
df = df.select("value")
display(df)

![image](https://github.com/user-attachments/assets/1dfb5f2b-7d30-4bfc-a507-00a443f842c1)
from pyspark.sql.functions import explode
df_exploded = df.select(explode(df["value"]).alias("json_object"))
display(df_exploded)

![image](https://github.com/user-attachments/assets/79c07309-46f5-4cc2-a498-dcfb2831d05e)
json_list = df_exploded.toJSON().collect()
print(json_list[25])

![image](https://github.com/user-attachments/assets/8f0bd6c8-aedb-4fe3-a80c-e25fef340d02)
import json
news_json = json.loads(json_list[25])
print(news_json)

![image](https://github.com/user-attachments/assets/219437ce-5699-4196-b790-05a0e1fa2640)
title = []

description = []

category = []

url = []

image = []

provider = []

datePublished =[]

Process each JSON object in the list
for json_str in json_list:
    try:
    
         Parse the JSON string into a distionary
        
        article = json.loads(json_str)
        
        if article["json_object"].get("category") and article["json_object"].get("image", {}).get("thumbnail", {}).get("contentUrl"):
        
            Extract information from the dictionary
            title.append(article["json_object"]["name"])
            
            description.append(article["json_object"]["description"])
            
            category.append(article["json_object"]["category"])
            
            url.append(article["json_object"]["url"])
            
            image.append(article["json_object"]["image"]["thumbnail"]["contentUrl"])
            
            provider.append(article["json_object"]["provider"][0]['name'])
            
            datePublished.append(article["json_object"]["datePublished"])
            
    except Exception as e:
    
        print(f"Error processing JSON object: {e}")  
        
  title
  
![image](https://github.com/user-attachments/assets/15576f39-19a6-48c6-8b01-d1bd8d7923d4)

from pyspark.sql.types import StructType, StructField, StringType
data=list(zip(title,description,category,url,image,provider,datePublished))

schema=StructType([

    StructField("title", StringType(), True),
    
    StructField("description", StringType(), True),
    
    StructField("category", StringType(), True),
    
    StructField("url", StringType(), True),
    
    StructField("image", StringType(), True),
    
    StructField("provider", StringType(), True),
    
    StructField("datePublished", StringType(), True)

])

df_cleaned=spark.createDataFrame(data, schema=schema)

display(df_cleaned)

![image](https://github.com/user-attachments/assets/846ccae1-c956-49e5-a78c-2c7082b522ff)

from pyspark.sql.functions import to_date,date_format
df_cleaned_final=df_cleaned.withColumn("datePublished",date_format(to_date("datePublished"),"dd-mm-yyyy"))
display(df_cleaned_final)
![image](https://github.com/user-attachments/assets/4605a94c-e51e-4461-8aaf-9f709f591e06)

from pyspark.sql.utils import AnalysisException

try:

    table_name = 'bing_lake_db.tbl_latest_news'

    df_cleaned_final.write.format("delta").saveAsTable(table_name)

except AnalysisException:

    print("Table Already exists")

    df_cleaned_final.createOrReplaceTempView("vw_df_cleaned_final")

    spark.sql(f"""  MERGE INTO {table_name} target_table
                    USING vw_df_cleaned_final source_view
                     
                     ON source_view.url = target_table.url
                     
                     WHEN MATCHED AND
                     source_view.title <> target_table.title OR
                     source_view.description <> target_table.description OR
                     source_view.category <> target_table.category OR
                     source_view.image <> target_table.image OR
                     source_view.provider <> target_table.provider OR
                     source_view.datePublished <> target_table.datePublished
                     
                     THEN UPDATE SET *
                     
                     WHEN NOT MATCHED THEN INSERT *
                     
                 """)
    %%sql 

select count(*) from bing_lake_db.tbl_latest_news
![image](https://github.com/user-attachments/assets/a63f0681-aaee-475c-b297-7c47480b00f5)

#Step 4:Real time sentiment analysis 

df = spark.sql("SELECT * FROM bing_lake_db.tbl_latest_news LIMIT 1000")
display(df)
![image](https://github.com/user-attachments/assets/57db0b69-0005-47c3-b414-643b06132622)
import synapse.ml.core
from synapse.ml.services import AnalyzeText
#configuring input and output column and importing model
model = (AnalyzeText()
        .setTextCol("description")
        .setKind("SentimentAnalysis")
        .setOutputCol("response")
        .setErrorCol("error"))
result = model.transform(df)
display(result)

![image](https://github.com/user-attachments/assets/e0f0a1e2-df69-4300-b168-a5c76d2671d0)
from pyspark.sql.functions import col
sentiment_df = result.withColumn("sentiment", col("response.documents.sentiment"))
display(sentiment_df)

![image](https://github.com/user-attachments/assets/f84849eb-c76a-431a-8454-a80989a87014)
sentiment_df_final = sentiment_df.drop("error","response")
display(sentiment_df_final)

![image](https://github.com/user-attachments/assets/6aaec8db-cfe1-48c2-9ba3-35452be1f394)

from pyspark.sql.utils import AnalysisException
try:
    table_name = 'bing_lake_db.sentiment_Analysis'
    sentiment_df_final.write.format("delta").saveAsTable(table_name)
except AnalysisException:
    print("Table Already exists")
    sentiment_df_final.createOrReplaceTempView("vw_sentiment_df_final")
    spark.sql(f"""  MERGE INTO {table_name} target_table
                    USING vw_sentiment_df_final source_view
                     ON source_view.url = target_table.url
                     WHEN MATCHED AND
                     source_view.title <> target_table.title OR
                     source_view.description <> target_table.description OR
                     source_view.category <> target_table.category OR
                     source_view.image <> target_table.image OR
                     source_view.provider <> target_table.provider OR
                     source_view.datePublished <> target_table.datePublished
                     THEN UPDATE SET *
                     WHEN NOT MATCHED THEN INSERT *
                 """)
                 
df=spark.sql("SELECT * FROM bing_lake_db.sentiment_analysis")
display(df)

![image](https://github.com/user-attachments/assets/3a8fc668-373a-4fe6-9132-c580b11c17c7)

df.printSchema()

![image](https://github.com/user-attachments/assets/89907507-7def-4ccc-bbe0-8f1def564b4c)

df.write.format('delta').mode("overwrite").option("overwriteSchema","True").saveAsTable(table_name)

df = spark.sql("SELECT * FROM bing_lake_db.sentiment_analysis LIMIT 1000")
display(df)

![image](https://github.com/user-attachments/assets/fc81a60d-0bbb-4fa8-a7d2-667ef33218e6)
![image](https://github.com/user-attachments/assets/b26ce934-7a3a-4f46-a373-37fd87b8b0be)

















