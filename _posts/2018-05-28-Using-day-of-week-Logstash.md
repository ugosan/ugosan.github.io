---
layout: post
title: Logstash - Augmenting events with day of week and day of month 
---

It is useful sometimes to have 


```ruby
    input {...}
    
    filter {
        mutate {
            add_field => {"[day_of_week]" => "%{+EEE}"}
            add_field => {"[day_of_month]" => "%{+d}"}
        }
    }
    
    output {...}
```



