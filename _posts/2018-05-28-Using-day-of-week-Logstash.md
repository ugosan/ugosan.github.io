---
layout: post
title: Logstash - Augmenting events with day of week and day of month 
---

It is useful sometimes to have day of week and day of month in fields that are separate from the `@timestamp` so we can make aggregations or even machine learning jobs to find a potential correlation between your events and weekdays.

In Logstash you can add the following to your pipeline:

```ruby
    input {...}
    
    filter {

        date {...} #your timestamp

        mutate {
            add_field => {"[day_of_week]" => "%{+EEE}"}
            add_field => {"[day_of_month]" => "%{+d}"}
        }
    }
    
    output {...}
```

The result is that Logstash will extract the values from the current `@timestamp` using [the same syntax used in the `date` filter](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html#plugins-filters-date-match). 

![State of Delaware's Open Data Portal (https://data.delaware.gov/)](/_posts/2018-05-28/day-of-week-logstash.jpg)


We could also have the week day in full, using `%{EEEE}`. You can see the whole syntax [here](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html#plugins-filters-date-match).
