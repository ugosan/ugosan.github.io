---
layout: post
title: Logstash Gsub tricks
excerpt: Logstash Gsub tricks
---

```
1. JSON parse error, original data now in message field {JSON}
2. Exception caught in json filter {JSON} :exception=>#<RuntimeError: Invalid FieldReference: `proc.aname[2]`>}
```

when you have `proc.aname[2]` and want to have `proc_aname2` - you can use regex groups to automatically change all occurrences of that string: 

```ruby
mutate {
    gsub => [ "message", "proc\.aname\[([0-9]+)\]", "proc_aname\1"]
}
```

Basically parenthesis `()` will make groups that can be later referenced by its number ( `\1` for the first group, `\2` for the second and so on.

<kbd><img src="/images/2022-09-27-10-22-27.png" ></kbd>

Used [Logshark](https://github.com/ugosan/logshark) for debugging