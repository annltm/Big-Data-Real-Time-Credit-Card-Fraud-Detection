# Big-Data-Real-Time-Credit-Card-Fraud-Detection
1. Objectives: 
This custom project involved the implementation of a real-time framework for credit card fraud predictions.
The specific objectives were as follows: 
Conduct standard ML pipeline development activities, including exploratory data analysis, feature engineering, and model design and evaluation as taught in the program. 
Explore the techniques for streamed transaction processing and ML predictions using Spark Streaming and Kafka. 

2. Analysis
   
Data Source: 
The source of data for this project was this synthetic (simulated) credit card fraud detection dataset published on Kaggle: 
https://www.kaggle.com/datasets/kartik2112/fraud-detection/data

It would have been extremely difficult to find real fraud data. However, this dataset was considered by many as quite realistic and rich in patterns. As such, it would be adequate to support the data analysis and ML pipeline development phase of the project. 
The dataset came in the form of a CSV file containing more than 1.2 million records. As far as preparation goes, we designed a schema to convert date and numerical features to the proper data types for analysis. The schema was also a key component for the real-time prediction component of the project, allowing the streaming dataframe to ingest and transform events the same way we did for the initial load and ML training. 

Exploratory Data Analysis:
Our preliminary scan of the dataset showed that there were no missing values or duplicate values. The features of interest to us had a relatively normal distribution with the exception of city population and the amounts involved in transactions. We have therefore applied logarithmic transformations to these elements as part of the pipeline. 
Unsurprisingly, the target class, “is_fraud”, was heavily imbalanced. Less than 1% of transactions are fraudulent (Figure 1). We had anticipated that this would be the case. As such, at the onset of the project, we made plans to assign weights to the target class. 

Fraud by Hour of the Day, by Hour of Day:
Notably, there is a significant increase in fraud incidents during late-night hours, between 10:00 PM and 12:00 AM. Another surge in fraudulent activities is observed in the early morning, especially between 1:00 AM and 3:00 AM. These patterns suggest that fraudsters might be exploiting the late-night and early morning hours, possibly capitalizing on the likelihood of reduced scrutiny from cardholders and financial institutions alike. 

Fraudulent Activity by Shopping Category: 
Online shopping and miscellaneous internet-based transactions exhibit the highest rates of fraud, everyday transaction categories, such as groceries and retail shopping, both online and at physical points of sale, also attract a considerable number of fraudulent activities, potentially due to their routine nature and lower individual transaction scrutiny. Conversely, sectors associated with lifestyle, essentials, and niche services, like travel, dining, and health, demonstrate lower fraud rates, possibly reflecting more robust security measures or less targeting by fraudsters due to higher risk and lower volume. 

Fraud by Vendor: 
The top five list of merchants with the highest fraud percentages — Kozey-Boehm, Herman LLC, Treutel-Dickens, Kerluke-Abshire, and Brown PLC — suggests that these vendors may be more susceptible to fraudulent transactions or targeted by fraudsters. These findings could be a signal to payment processors and financial institutions to apply more stringent fraud checks for transactions involving these vendors. 

Fraud Ratio by Age Group: 
Is fraud more prevalent in certain age groups than others? The analysis below suggests that seniors aged 80+ are more exposed to fraudulent transactions than other groups. This made age a potentially viable feature for the machine learning pipeline. 

Fraud by Age Group and Gender: 
We took a further look into age groups and split by gender to identify potential patterns in fraudulent activities. The analysis revealed that males in the age group 50 exhibited the highest incidence of fraudulent transactions whereas fraudulent activity was highest for females in the age group 30. Surprisingly, this trend indicates the lowest number of fraudulent activities among seniors, contradicting our initial observation. It is important to note that these analyses focus on the sheer number of fraudulent transactions and not the ratio of fraudulent transactions to total transactions. Both analyses yielded intriguing insights. 

Fraud by Transaction:
Lastly we delved into fraudulent activity based on transactional amount and discovered that fraud activities were most prevalent for transactions under $1500. Percentage of fraud was highest for transactions below $100 and peaked again between $200-$400 before tapering down. Transactions between $800-$1100 also exhibited a higher percentage of fraud. The concentration of fraudulent activities in transactions below $1500 suggests that fraudsters may be more inclined to target smaller-value transactions (such as those under $100). This could be because smaller transactions are less likely to trigger suspicion and may be overlooked. 

Feature Engineering:
Though some of the individual features alone result in a high fraud transaction fraud rate, we would often see a high false positive rate if a rule is created using only one feature. To optimize the performance of individual rules, it is often a good idea to pair features together. 
In our case, a strong feature that we have identified is the hour of the day. It is very evident that around midnight, there is a high volume of fraud. However, using time of the day alone would cause our engine to over-fire. 

1) grocery_pos, misc_net and shopping_net, along with hour_of_day being almost midnight generate a fraud rate of over 10%, which is acceptable for a fraud engine. We can use these two features to create the first rule. 
2) Pairing the hour_of_day with amount also gives us a very strong sense of rule creation around these two features. 
3) Last but not least, we can combine the above 2 rules together to alert transactions by categories, time and amount.
   
3. Overall Design:

We designed and tested a functional end-to-end machine learning pipeline. After evaluation of different algorithms and tuning, the best model was retained and used to support the real-time prediction process. The first streaming prototype was based on CSV file inputs. The second prototype was based on Apache Kafka. Shown below is a high level overview of our training and real-time streamed event processing pipelines. 
The ML pipeline was designed in such a way as to be able to ingest raw inputs and perform all necessary transformations as distinct stages in the pipeline. This was also a requirement to enable structured streaming capabilities. One major challenge in that area was to incorporate into the pipeline the various data transformations and extra features needed to establish spending patterns. The Spark MLlib package provided a number of common transformations (e.g. VectorAssembler, StringIndexer, OneHotEncoder, etc..). However, to achieve our goal, we had to define our own transformers. 
As an additional preprocessing step before training, we balanced the dataset by computing a weight column based on fraud ratio and then configuring the models to use that column to apply the appropriate weight for each record. 

Model Training, Tuning and Evaluation:

We trained and compared the performance of three models based on the RandomForestClassifier, GradientBoostingTreesClassifier and Feedforward Neural Network algorithms. As the dataset was quite large (1.2M records, 500MB storage), we ran cross-validation and hyperparameter tuning outside the shared workspace on the most powerful machines the group had. We then integrated the best parameters in a single tuned model pipeline. 

As the target class (is_fraud) is heavily imbalanced, we felt that Precision-Recall metrics would be more meaningful than the AUC-ROC analysis. The P-R curve we produced out of this exercise highlights the compromise that would have to be reached between precision and recall should a different probability threshold be selected. 
Feature importances extracted from the tuned model were consistent with our EDA findings. Unusual transaction amounts are the strongest differentiators for fraud detection, followed by specific merchant categories (e.g. gas stations and grocery stores). 

As a final step, we proceeded with finding the best balance between precision and recall by testing different probability thresholds. Given that the F1 metric takes into account precision and recall, we choose F1 as the evaluation metric. As the table below shows, a threshold of 90% yielded the best score. As a result, we programmed the best model to use that threshold for the real-time pipeline. 
Probability Threshold	0.5	0.6	0.7	0.8	0.9
F1 Score	0.423	0.470	0.536	0.601	0.673

Real Time Prediction Pipeline: 
We implemented the real-time pipeline with the Spark Structured Streaming framework. Streaming dataframes work in the same way as static dataframes. However there are differences that must be taken into consideration. In a streaming context, data is ingested in small batches. This makes the use of aggregates in the prediction pipeline a little more complex. 
For instance, we incorporated a feature containing the number of transactions accumulated over a 24-hour period. In a static dataframe, for training purposes, adding this feature was only a matter of running an aggregation query against the whole dataframe. In a streamed dataframe, where data comes in as micro-batches, a separate dataframe must be created to keep track of daily transaction counts. This “aggregate” dataframe can then be joined with the original streamed dataframe to add the new feature column. The diagram below illustrates the process. 
Our first streaming prototype was constructed with CSV files as the format for the transaction events. This provided us with a quick way to generate transactions from the testing dataframe and test our real-time pipeline. Once we confirmed that the pipelined worked properly, we proceeded with Kafka as our streaming source. Setup of Zookeeper and Kafka was a fairly involved process. However, much to our relief, changing the streaming source was straightforward and only required changes to the configuration parameters of the streamed dataframe. The screenshot below shows the pipeline in action and our first real-time detection of a fraudulent transaction. 

4. Conclusions:
This project proved to be as challenging as it was rewarding. From a data analysis and fraud detection standpoint, the main conclusions are as follows: 
 1) The analysis suggests that most fraudulent transactions involve larger than usual amounts and occur late in the evening. 
2) Credit card fraud detection is very much a matter of compromise. As we observed through numerous runs involving different parameters, maximizing fraudulent transaction detection can lead to a higher rate of false positives, which would translate into calls from annoyed customers. On the other hand, increasing the probability threshold will lead to a higher count of false negatives, resulting in increased fraud handling costs for the credit agency. We tuned our model to strike the best balance between the two. However a credit card agency may opt to choose a different strategy. 
Regarding the technical ML pipeline aspects,the main takeaways are as follows: 
● Out of the classification algorithms we evaluated, GradientBoostClassification yielded the best performance (using the AUC-PR metric) for this particular use case. 
● In spite of similarities between static and streamed dataframe APIs, an ML pipeline must be adjusted to the micro-batch approach used for all operations. This was particularly important for aggregate features. Enrichment of the streamed dataframes with aggregates must be done by creating separate aggregate queries, which can then be joined with the main inbound dataframe. 

