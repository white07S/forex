## Forex analysis and High frequency time series data managment
# Real-time economic events analysis and bi directional prediction
### Architecture

![Architecture](https://github.com/white07S/forex/blob/main/assets/arch.png)

* [producer]
 	
   	The producer instantiates day session and gets the intraday market data from Alpha Vantage and IEX Cloud APIs, it also runs Scrapy Spiders to fetch economic indicators, COT Reports data and VIX. The producer will call the source API and extract data from web sources with the frequency specified by the user (interval) until the market is closed. The collected data subsequently creates a set of streams that are published to corresponding Kafka topics.

* [spark_consumer]

  	The distributed streaming Pyspark application that is responsible for following tasks:

    - subscribe to a stream of records in given Kafka topic and create a streaming Data Frame based on the pre-defined schema
    - fill missing values
    - perform real-time financial data feature extraction:

        - weighted average for bid's and ask's side orders      

        - Order Volume Imbalance

        - Micro-Price (according to Gatheral and Oomen)

        - Delta indicator

        - Bid-Ask Spread

        - calculate the bid and ask price relative to best values

        - day of the week

        - week of the month

        - start session (first 2 hours following market opening)

    - perform oneHotEncoding
    - join streaming data frames
    - write stream to MySQL/MariaDB
    - signal to Pytorch model readiness to make a prediction for current datapoint

	For testing we will run Spark locally with one worker thread (.master("local")).
	Other options to run Spark  (locally, on cluster) can be found here:
	<http://spark.apache.org/docs/latest/submitting-applications.html#master-urls>

	SPARK STRUCTURED STREAMING LIMITATIONS:

	In Spark Structured Streaming 2.4.4 several operations are not supported on Streaming DataFrames. The most significant constraint pertaining this application is that multiple streaming aggregations (a chain of aggregations on a streaming DataFrame) are not yet supported, thus the feature extraction process that requires multiple window aggregations will be moved from Spark to MariaDB.

	All unsupported operations are listed here <https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#unsupported-operations>


* [create_database]
	 Creates a 'stock_data' database that stores in the main table, processed by Spark application data, but also performs further feature extraction using SQL views. The following are the created additional features:

	- Volume Moving Averages
	- Price Moving Averages
	- Delta indicator Moving Averages
	- Bollinger Bands
	- Stochastic Oscillator
	- Average True Range

	Creates a VIEW with target variables, that are determined using following manner:

	|                Condition                    |  up1  |  up2  | down1 | down2 |
	| :------------------------------------------:|:-----:|:-----:|:-----:|:-----:|
	| 8th bar p8_close >= p0_close + (n1 * ATR)   |   1   |   0   |   0   |   0   |
	| 15th bar p15_close >= p0_close + (n2 * ATR) |   0   |   1   |   0   |   0   |
	| 8th bar p8_close <= p0_close - (n1 * ATR)   |   0   |   0   |   1   |   0   |
	| 15th bar p15_close <= p0_close - (n2 * ATR) |   0   |   0   |   0   |   1   |

	You can generate different target variables by modifying SQL <i>target_statement</i>


