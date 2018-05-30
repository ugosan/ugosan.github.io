---
layout: post
title: Analyzing Credit Card Transactions with the Elastic Stack. Part 1 - Ingesting
---

In this blog post I'm going to detail all the steps needed to load data from a CSV file into the Elastic Stack, so we can explore it, make visualizations, augment the original data and use more advanced techniques such as machine learning in order to find some possible patterns in the data. 

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


Our folder structure for this project would look like this:

```
/elastic-csv
    pipeline.conf    
    logstash-6.2.4/
    State_Employee_Credit_Card_Transactions.csv
```

We are going to use Logstash for ingesting the data. Let's download it and put it in the same folder. https://artifacts.elastic.co/downloads/logstash/logstash-6.2.4.tar.gz

_Note: Logstash needs a Java runtime to be installed, OpenJDK is fine_

## Pipeline configuration (`pipeline.conf`)
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
        path => "path/to/State_Employee_Credit_Card_Transactions.csv"
        start_position => "beginning"
        sincedb_path => "NUL"
    }
}
```

_note: the `path` must be absolute_


This input will pass along every line of the file through the filters, starting from the `beggining`. Logstash also keeps track of the last processed line in the csv, writing a log file specified in `sincedb_path`. We are probably going to be testing and debugging the pipeline, so want to set it to `NUL` to start afresh every time. 

### The `csv` filter
Once we have specificed the `input{}` there are a couple of filters we will use. Let's start with the `csv` filter: we should tell the names of the columns that will become field names of our documents.

```ruby
    csv {
        separator => ","
        columns => ["FISCAL_YEAR","FISCAL_PERIOD","DEPT_NAME","DIV_NAME","MERCHANT","CAT_DESCR","TRANS_DT","MERCHANDISE_AMT"]
        skip_header => true
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
        columns => ["FISCAL_YEAR","FISCAL_PERIOD","DEPT_NAME","DIV_NAME","MERCHANT","CAT_DESCR","TRANS_DT","MERCHANDISE_AMT"]
        skip_header => true
    }

}

output {
    stdout {}
}
```

Regarding the `output`, let's leave it as `stdout` so we can check everything is right with the date, this output will just print the documents to the console.



We can go ahead and test the pipeline executing logstash from the command line:

```bash
./logstash-6.2.4/bin/logstash -f pipeline.conf --debug
```

Logstash will start reading the file and passing every line through the pipeline. This is how a document looks like on the output:
```ruby
{
           "DIV_NAME" => "STOCKLEY CENTER",
        "FISCAL_YEAR" => "2013",
           "TRANS_DT" => "06/27/2012",
           "MERCHANT" => "PCI*SAMMONS PRESTON",
          "CAT_DESCR" => "DENTAL-LAB-MED-OPHTHALMIC HOSP EQUIP SUPPLIES",
               "host" => "tempo.local",
         "@timestamp" => 2018-05-30T20:49:57.798Z,
      "FISCAL_PERIOD" => "1",
          "DEPT_NAME" => "DEPT OF HEALTH AND SOCIAL SV",
        "MERCHANDISE_AMT" => "$379.86",
}
```


# Parsing date and amount correctly

So the above document has two problems:

* Problem #1 - date is incorrect

The `@timestamp` field on the document shows the **current** timestamp, being the time and date in which the document itself was created, we need it to be the same as `transaction_date` which is `06/28/2012`

* Problem #2 - amount is a string

The `amount` field on the above document is expressed as `"amount" => "$19.00"`, we need to get rid of the `$` otherwise it would not be treated as a number.


## `date` filter
The `date` filter takes a text field and parses it according to a pattern, and uses the resulting date as the document's timestamp field. In our case we have `transaction_date` expressed like "05/20/2016" which would be `"MM/dd/yyyy"`

```ruby
date {
    match => [ "transaction_date", "MM/dd/yyyy" ]
}
```

## `mutate` filter
Lets also remove the `$` character from the amount, so we can work with it as a number, instead of a string.

```ruby
    mutate {
        gsub => [
            "amount", "[\\$]", ""
        ]
    }
```

Lets give the fields more readable names, using the `rename` mutate:

```ruby

    mutate {
        rename => { "FISCAL_YEAR" => "fiscal_year" }
        rename => { "FISCAL_PERIOD" => "fiscal_period"}
        rename => { "DEPT_NAME" => "department"}
        rename => { "DIV_NAME" => "division" }
        rename => { "MERCHANT" => "merchant" }
        rename => { "CAT_DESCR" => "category" }
        rename => { "TRANS_DT" => "transaction_date"}
        rename => { "MERCHANDISE_AMT" => "amount"}
    }

```


## Final pipeline configuration

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


    mutate {
        rename => { "FISCAL_YEAR" => "fiscal_year" }
        rename => { "FISCAL_PERIOD" => "fiscal_period"}
        rename => { "DEPT_NAME" => "department"}
        rename => { "DIV_NAME" => "division" }
        rename => { "MERCHANT" => "merchant" }
        rename => { "CAT_DESCR" => "category" }
        rename => { "TRANS_DT" => "transaction_date"}
        rename => { "MERCHANDISE_AMT" => "amount"}
    }

    #using the transaction_date in the event as the date 
    date {
        match => [ "transaction_date", "MM/dd/yyyy" ]
    }

    #removing the $ character from the amount field
    mutate {
        gsub => [
            "amount", "[\\$]", ""
        ]
    }

    #adding day of week, day of month 
    #and removing fields we dont need
    mutate {
        add_field => {"day_of_week" => "%{+EEEE}"}
        add_field => {"day_of_month" => "%{+d}"}
        add_field => {"month" => "%{+MMMM}"}
        remove_field => [ "message", "transaction_date", "host", "path" ]
    }
}

output {
    stdout {}
}
```

Now this pipeline's output shows a much better parsed documents, the `amount` field is correctly recognized as a number, the `@timestamp` field correctly reflects the transaction's date.

![Logstash output](/images/delaware/ss3.jpg)

```ruby
{
    "fiscal_period" => "1",
      "fiscal_year" => "2013",
       "@timestamp" => 2012-06-26T05:00:00.000Z,
       "department" => "DEPT OF TRANSPORTATION",
     "day_of_month" => "26",
         "category" => "INDUSTRIAL SUPPLIES NOT ELSEWHERE CLASSIFIED",
            "month" => "June",
      "day_of_week" => "Tuesday",
         "merchant" => "WW GRAINGER",
           "amount" => 22.2,
         "@version" => "1",
         "division" => "MAINTENANCE DISTRICTS"
}
```


# Sending data to Elastic Cloud

There are multiple ways to have this data stored so we can visualize it. 

Lets use Elastic Cloud, you can create a 14-day trial account at https://cloud.elastic.co/ and create an Elasticsearch cluster in seconds.

Give it a name to your deployment and select the smallest size (1GB) and 1 availability zone. 
![Elastic Cloud](/images/delaware/ss4.jpg)

Click **Create Deployment** and your cluster should be created. Elastic Cloud will show you the password for the `elastic` user, which is the one we will use to receive the data from our Logstash. 


![Elastic Cloud](/images/delaware/ss5.jpg)

After the cluster is created, open it up and copy the Elasticsearch endpoint


![Elastic Endpoints](/images/delaware/ss6.jpg)

## Back to our `pipeline.conf`

We have now created our cluster, lets add `elasticsearch` as an output in our pipeline configuration, setting `demos-delaware` as the index we are going to write to and using `elastic` as the user:

```ruby
output {
    stdout {}
    
    elasticsearch {
        hosts => "https://38d7e6df573a4b9d83642761ae5cd095.europe-west1.gcp.cloud.es.io:9243"
        index => "demos-delaware"
        user => "elastic"
        password => "ELFBX8KhrgUlxbmqA2hjrsqj"
    }
}
```


Let's execute the pipeline again from the command line and in a couple of minutes we should be able to see the data in our Elasticsearch:

```bash
./logstash-6.2.4/bin/logstash -f pipeline.conf --debug
```

## Visualizing with Kibana
Open up the Kibana endpoint using the link we have displayed in [Endpoints](elastic-endpoints). Go to **Management > Index Patterns** and put `demos-delaware` as the index pattern.

![Kibana](/images/delaware/ss7.jpg)

Select `@timestamp` as the time field in the next screen and click **Create index pattern**.

![Kibana](/images/delaware/ss8.jpg)


Now go to the **Discover** tab and change the time filter to the last 8 years or so and an Auto-refresh of 30 seconds:
![Kibana](/images/delaware/ss9.jpg)

We can see the documents being loaded in realtime, at this point we can already start making some visualizations.

![Kibana](/images/delaware/ss10.jpg)


