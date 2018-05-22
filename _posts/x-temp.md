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

According to Delaware's Open Data portal: "Purchasing cards are used by employees within the Organization to purchase goods/services that are needed for business. These cards should also be used for registering travelers for conferences."

## Downloading the dataset
You can choose **Export > CSV** from the portal or if you are on Mac or Linux you can download it directly using wget:
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

Enters Logstash
```
wget https://artifacts.elastic.co/downloads/logstash/logstash-6.2.4.tar.gz
```

## Pipelines
```
input {}

filter {}

output {}
```

```yaml
input {
    file {
        path => "path/to/State_Employee_Credit_Card_Transactions.csv"
        start_position => "beginning"
    }
}

filter {
    csv {
        separator => ","
        columns => ["fiscal_year","fiscal_period","department","division","merchant","category","date","amount"]
    }

    date {
        match => [ "date", "MM/dd/yyyy" ]
    }

    mutate {
        remove_field => [ "message", "date", "host", "path" ]
    }

    mutate {
        gsub => [
            "amount", "[\\$]", ""
        ]
    }
}

output {
    stdout {
        codec => rubydebug
    }
}
```

## Fingerprinting Transactions
