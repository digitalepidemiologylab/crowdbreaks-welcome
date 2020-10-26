# Crowdbreaks

Hi! You're probably reading this either because you are working on the Crowdbreaks project or you are trying to understand more about what Crowdbreaks is.

The goal of this README is to provide an overview of what there is and where the project is headed. 

## Goal
For many health-related issues human behaviour is of central importance for Public Health to design appropriate policies. Health behaviors are partially influenced by people's opinion which has been traditionally assessed in surveys. Social media can be used to complement traditional surveys and serve as a low-cost, global, and real-time addition to the toolset of Public Health surveillance.

Crowdbreaks focuses on *public* social media data (currently from Twitter) to track such health behaviors. A common issue when building Machine Learning classifiers on social media data is model drift, as e.g. observed in Google Flu Trends. Crowdbreaks is specifically built to overcome this issue by re-annotating newly collected data and re-training algorithms automatically. For a (maybe slighly outdated, but more comprehensive) explanation of this idea, you may want to read [the Crowdbreaks paper](https://www.frontiersin.org/articles/10.3389/fpubh.2019.00081/full).

## Platform overview
The Crowdbreaks platform consists of multiple parts, each part has its own repository on GitHub. It should be seen as a toolbox to overcome many distinct technical challenges in analyzing large amounts of data in real-time. In general, all code related to Crowdbreaks is open source under MIT license.

### Website
The Crowdbreaks website is accessible under [https://www.crowdbreaks.org/](https://www.crowdbreaks.org/). 

The website has three main purposes:
1. **Collect and store annotation data.** 
Annotation data (also called "labelled data") is when raw data gets tagged with a class (or label) by a human. Annotation data is central to all supervised Machine Learning and is key to most projects running on Crowdbreaks. Annotation can be done by any user (registered or not) on the website (this process is also called "crowdsourcing"). Alternatively, annotation data can also be collected through Amazon Turk, which is a paid service. Lastly, annotation can be done by registered users (e.g. experts) who registered on the Crowdbreaks website. These three modes of annotation are called `public`, `mturk`, and `local`.
2. **Visualizations**: Part of the idea of Crowdbreaks is to provide results of the analysis back to the public. One way of doing this is via visualizations of these trends. In the future the platform might also provide educational content or expose public APIs which would allow real-time trends from Crowdbreaks to be integrated into other Public Health tools.
3. **Project management**: The website as a project management interface, allowing to 1) create and manage projects 2) control the data collection (see section on data streaming) 3) easily use Mturk 4) monitor the status of the system

**Tech**: The website is built with Ruby/Rails with a React.js front-end (visualizations are in d3.js). Annotation data is stored in a Postgres database. The website is hosted on Heroku.

### Data streaming
For data collection, Crowdbreaks leverages streaming endpoints within Twitter Developer API. The infrastructure is set up using Amazon Web Services (AWS).

There is a Python [streamer](https://github.com/crowdbreaks/crowdbreaks-streamer-2) app that runs on an AWS Fargate cluster and uses a [POST statuses/filter](https://developer.twitter.com/en/docs/twitter-api/v1/tweets/filter-realtime/api-reference/post-statuses-filter) (API v1.1) request to connect to a filtered stream of relevant tweets. The relevant tweets are filtered based on keywords and languages that are provided for each project within Crowdbreaks.

The whole data pipeline is set up using AWS. The streamer app itself runs on a Fargate cluster. After aquiring the tweets, it sends them over to their corresponding Kinesis Firehose Delivery Streams (one per project), which saves each project's tweets with a separate key-prefix ("folder") to a bucket in Simple Cloud Storage (S3). Each new batch of tweets being saved to S3 triggers an event that invokes a Lambda function, which preprocesses the tweets in the batch, makes predictions and sends the preprocessed data over to a project's Elasticsearch index.

This way, Crowdbreaks is able to collect and keep Twitter data in a flexible and scalable fashion.

### Analysis tools
Although Crowdbreaks is a real-time analysis tool (which is run in the Cloud, or more specifically on AWS), a great effort has been put in building tools to analyze Crowdbreaks data locally (e.g. on a cluster).

#### Preprocess
[Preprocess](https://github.com/crowdbreaks/preprocess) is a CLI (command line interface) tool, which allows to:
* **init**: Initialize  for a specific project
* **sync**: Download (synchronize) raw data (tweets) and annotation data for that project
* **parse**: Parse the raw data into a Python-usable format (Pandas DataFrames)
* **sample**: Select data for annotation on the Crowdbreaks website (usually around 10'000 tweeets)
* **batch**: Create a batch of the sample to be uploaded to the Crowdbreaks website for annotation (Generally batches are <4000 tweets)
* **clean_labels**: A single tweet is usually labelled by multiple users. In order to get a consensus between multiple users the annotation data is both cleaned from outliers and merged. The result is a consensus label for every tweet, which serves as training data for a Machine Learning algorithm

Additionally, the library provides efficient helper functions (e.g. to load the data). Usually this library is therefore integrated as a git submodule within a project-specific GitHub repository.

**Tech**: The tool is written in Python. For easier data cleaning we are using the Pandas library. Raw data is either in CSV format (annotation data) or in gzipped JSON-lines format (raw Twitter data). The parsed data (DataFrame) is in a binary format called parquet. 

#### txtcls
[txtcls](https://github.com/crowdbreaks/text-classification) is a CLI tool that automates text classifiction model training, testing and deployment for speed and reproducibility.

#### local-geocode
[Local geocode](https://github.com/mar-muel/local-geocode) is a library to perform reverse geo-coding. It is specificially tuned to work on the user location field of tweets. The library parses a string such as "New York City" and returns the geo coordinates (longitude and latitude) if there is match. It is a library which is used in preprocess (and soon in the streamer as well) to enrich geo-information of tweets.

**Tech**: The library is written in Python and uses data from [geonames.org](https://www.geonames.org/).

#### COVID-Twitter-BERT
[COVID-Twitter-BERT](https://github.com/digitalepidemiologylab/covid-twitter-bert) is a library which allows to perform what is called domain-specific pretraining (DSP) of transformer models. DSP is a process in which a base model (such as BERT) is trained on domain specific data (such as tweets) in order to increase its performance over the base model in downstream tasks (such as classification). The process was specifically used on a large dataset of tweets about COVID-19 and is described in [this](https://arxiv.org/abs/2005.07503) paper. 

**Tech**: The library is written in Python and using the Tensorflow 2 library. It is optimized for training on TPUs and requires a Google Cloud bucket.

## Past & ongoing Crowdbreaks projects
### Vaccine sentiment tracking
Vaccine sentiment tracking is an ongoing project and serves as a classic used case for Crowdbreaks. The lab has previous worked on this topic and collected annotation data outside of Crowdbreaks (e.g. in [this](https://www.pnas.org/content/114/52/13762?collection=&utm_source=TrendMD&utm_medium=cpc&utm_campaign=Proc_Natl_Acad_Sci_U_S_A_TrendMD_0) project). The data was also used in a project to understand correlations between vaccine sentiment and vacine uptake in England. The study can be found [here](https://www.sciencedirect.com/science/article/pii/S0264410X20307374).

### Vaccination sentiment in Brazil
This is a collaboration with PAHO (Pan American Health Organization), who is specifically interested in the influence of social media on vaccine uptake in Brazil.

### Assessing public opinion on CRISPR/Cas9
In this collaboration with ETH Zurich we used Twitter data related to the gene editing technology CRISPR/Cas9. Feel free to read more [here](https://www.jmir.org/2020/8/e17830).

### COVID-19 disease outbreak
The COVID-19 outbreak led to a massive flood of data which led to multiple projects:
* **Trending topics and tweets**: Early on in the pandemic we used Crowdbreaks to filter incoming data and generated a visualization of real-time tweets and trending topics. This was later discontinued for cost reasons.
* **Attention to experts**: A critical question is the role of scientific experts on Twitter. A study related to this can be found [here](https://arxiv.org/abs/2008.08364).
* Multiple other projects emerged from this data with work in progress.
