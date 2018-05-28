---
layout: post
title: Analyzing Credit Card Transactions with the Elastic Stack. Part 1 - Ingesting
---

In this blog post I'm going to detail all the steps needed to load data from a CSV file into the Elastic Stack, so we can explore it, make visualizations, augment the original data and use more advanced techniques such as machine learning in order to 

* **Part I - Ingesting the data** (You are here)
* Part II - Graph and other visualizations
* Part III - Fingerprinting transactions
* Part IV - Machine Learning

# The dataset

We are going to use an open dataset from State of Delaware called **State Employee Credit Card Transactions**. 

![State of Delaware's Open Data Portal (https://data.delaware.gov/)](/images/delaware/portal.jpg)

According to Delaware's Open Data portal this data set contains the : "Purchasing cards are used by employees within the Organization to purchase goods/services that are needed for business. These cards should also be used for registering travelers for conferences."

## Downloading the dataset
You can choose **Export > CSV** from the portal or if you are on Mac/Linux you can download it directly using wget:
```
wget https://data.delaware.gov/api/views/nurt-5rqw/rows.csv
```




We can see that this dataset has the following headers: `FISCAL_YEAR`, `FISCAL_PERIOD`, `DEPT_NAME`, `DIV_NAME`, `MERCHANT`, `CAT_DESCR`, `TRANS_DT`, `MERCHANDISE_AMT` and about 1 million rows, one for every transaction ranging from 2013 to 2018.

```
FISCAL_YEAR,FISCAL_PERIOD,DEPT_NAME,DIV_NAME,MERCHANT,CAT_DESCR,TRANS_DT,MERCHANDISE_AMT
2013,1,SVS FR CHILDREN YOUTH FAMILIES,INTERVENTIONTREATMENT,SODEXHO 30051437,EATING PLACES RESTAURANTS,04/26/2012,$5.28
2013,1,SVS FR CHILDREN YOUTH FAMILIES,MANAGED CARE ORGANIZATION,IMAGISTICSINV 204164237,BUSINESS SERVICES-NOT ELSEWHERE CLASSIFIED,05/08/2012,$648.00
2013,1,INDIAN RIVER SCHOOL DISTRICT,HOWARD T ENNIS SCHOOL,FRAUD CHARGEBACK,INTERNAL TRANSACTION,05/16/2012,$321.57
2013,1,EXECUTIVE,FACILITIES MANAGEMENT,C/B GROUP REVERSAL,INTERNAL TRANSACTION,05/18/2012,$52.40
2013,1,EXECUTIVE,FACILITIES MANAGEMENT,C/B GROUP REVERSAL,INTERNAL TRANSACTION,05/18/2012,$99.22
2013,1,DEPT OF SAFETY AND HOMELAND,ST BUREAU OF IDENTIFICATION,CHARGEBACK CREDIT,INTERNAL TRANSACTION,05/27/2012,$-15.75
2013,1,DEPT OF SAFETY AND HOMELAND,DEMA,BYRDS CATERING SERVICE,CATERERS,06/01/2012,$1620.00
2013,1,DEPT OF HEALTH AND SOCIAL SV,MANAGEMENT SERVICES,BRICCO,EATING PLACES RESTAURANTS,06/13/2012,$56.01
2013,1,DEPT OF LABOR,OSHABLS,COURTESY WRITE OFF,INTERNAL TRANSACTION,06/18/2012,$-3.14
...
```

# Ingesting a CSV with Logstash

We are going to use Logstash for ingesting the data. Let's download it and put it on our folder. https://artifacts.elastic.co/downloads/logstash/logstash-6.2.4.tar.gz

_Logstash needs a Java runtime to be installed, OpenJDK_

## Pipeline configuration
```
input {}

filter {}

output {}
```


### The `file` input

The `file` input stream events from files, normally by tailing them in a manner similar to `tail -0F` but optionally reading them from the beginning, which is exactly what we are going to do with this file.

```ruby
input {
    file {
        path => "path/to/csv-files/State_Employee_Credit_Card_Transactions.csv"
        start_position => "beginning"
        sincedb_path => "NUL"
    }
}
```

This input will pass along every line of the file through the filters, starting from the `beggining`. Logstash also keeps track of the last processed line in the csv, writing a log file specified in `sincedb_path`. We are probably going to be testing and debugging the pipeline, so want to set it to `NUL` to start afresh every time. 

### The `csv` filter
Once we have specificed the `input{}` there are a couple of filters we will use. Let's start with the `csv` filter: we should tell the names of the columns that will become field names of our documents.

```ruby
csv {
    separator => ","
    columns => ["fiscal_year","fiscal_period","department","division","merchant","category","transaction_date","amount"]
}
```



### Testing our pipeline
For now, this is the pipeline we have:

```ruby
input {
    file {
        path => "path/to/csv-files/State_Employee_Credit_Card_Transactions.csv"
        start_position => "beginning"
        sincedb_path => "NUL"
    }
}

filter {
    csv {
        separator => ","
        columns => ["fiscal_year","fiscal_period","department","division","merchant","category","transaction_date","amount"]
    }

}

output {
    stdout {}
}
```

Regarding the `output`, let's leave it as `stdout` so we can check everything is right with the date, this output will just print the documents to the console.




Our folder structure should look something like this:

```
/elastic-csv
    pipeline.conf
    logstash-6.2.4/
    csv-files/
        State_Employee_Credit_Card_Transactions.csv
```

We can go ahead and test the pipeline executing logstash from the command line:

```bash
./logstash-6.2.4/bin/logstash -f pipeline.conf --debug
```

Logstash will start reading the file and passing every line through the pipeline

![Logstash output](/images/delaware/ss1.jpg)

# Parsing date and amount correctly

The `date` filter takes a text field and parses it according to a pattern, and uses the resulting date as the document's timestamp field. In our case we have `transaction_date` expressed like "05/20/2016" which would be `"MM/dd/yyyy"`

```ruby
date {
    match => [ "transaction_date", "MM/dd/yyyy" ]
}
```


```ruby
input {
    file {
        path => "path/to/csv-files/State_Employee_Credit_Card_Transactions.csv"
        start_position => "beginning"
        sincedb_path => "NUL"
    }
}

filter {
    csv {
        separator => ","
        columns => ["fiscal_year","fiscal_period","department","division","merchant","category","transaction_date","amount"]
    }

    #using the date in the event as the date 
    date {
        match => [ "transaction_date", "MM/dd/yyyy" ]
    }

    #removing fields we dont need anymore
    mutate {
        remove_field => [ "message", "transaction_date", "host", "path" ]
    }

    #removing the $ character from the amount field
    mutate {
        gsub => [
            "amount", "[\\$]", ""
        ]
    }
}

output {
    stdout {}
}
```