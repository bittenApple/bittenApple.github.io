---
layout: post
title: "TIMESTAMP, DATETIME or INT for a Mysql Time Field"
description: ""
category: "Programming" 
comments: true
tags: [MySQL, TIMESTAMP, DATETIME]
---

I used to using INT type for MySQL time column, some days ago I found other members of team like to use TIMESTAMP or DATETIME type, we had a discussion to have an agreement on unifying the time field type when design database schema. This article is an attempt to better understand how to choose appropriate time type.

## Differences
* Default MySQL INT type takes 4 bytes (you could see int(10) when you use `show create table`, the number 10 in parentheses is MySQL display width, not storage or data range). The range of Signed INT type is from -2147483648 to 2147483647, which supports UTC time from '1970-01-01 00:00:01' to '2038-01-19 11:14:07'; the range of unsigned INT is 0 to 4294967295, which supports UTC time that from '1970-01-01 00:00:01' to '2106-02-07 14:28:15'.
* TIMESTAMP is 4 bytes (after MySQL 5.6.4, it needs fractional seconds storage), the supported range is '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC.
* DATETIME is 8 bytes, MySQL retrieves and displays DATETIME values in 'YYYY-MM-DD HH:MM:SS' format. The supported range is '1000-01-01 00:00:00' to '9999-12-31 23:59:59' and it supports fractional seconds.

## About Timstamp
Fist, we should know why timestamp (not the MySQL filed type) is useful, as it is timezone independent. The timestamp is the number of seconds elapsed since an absolute point in time (midnight of Jan 1 1970 in UTC time). If you render your time at html view by a backend language, maybe a person sees the time is 12:00 in timezone Asia/Shanghai, but it's a wrong time for people who visit you page at America.  
I find many Chinese developers are not very concerned about this issue. Probably because China follows UTC+08:00 as nationwide time standard, despite China spanning five geographical time zones. And many Chinese websites face to native users only.  
For web development, the correct way is always using timestamp, and calculating the displayed time by JavaScript, then users in different timezones would see the correct local time.
As for DateTime filed, if the database server timezone setting has to be changed, you have to also recalculate the data.

## Others
MySQL supports many date and time functions to manipulate temporal values, which are very handy if you use TIMESTAMP or DATETIME field.   
Not having actually benchmarked the speed when these types indexed, some posts say INT type is significantly faster than DATETIME. I think the main reason is storage length of INT is shorter than DATETIME, leading to smaller index which is easy to comparing.

## Conclusion
If you don't care about readability when using MySQL client or phpMyAdmin, unsigned INT as time field is a good choice. In fact, it's not that inconvenient to read when using FROM_UNIXTIME function which could translate int value to a formatted date value, and INT type is also very convenient to communicate with PHP.  
If you care about the readability and need to update the time data at phpMyAdmin, use TIMESTAMP. Don't worry about the 2038 time limit, there are still more than twenty years left after all. And I think MySQL TIMESTAMP field will support larger time range in the future.  
If you need larger date range, use DATETIME.
If you need fractional seconds, use TIMESTAMP or DATETIME.

## References
[https://stackoverflow.com/questions/409286/should-i-use-field-datetime-or-timestamp](https://stackoverflow.com/questions/409286/should-i-use-field-datetime-or-timestamp)
[http://dev.mysql.com/doc/refman/5.7/en/datetime.html](http://dev.mysql.com/doc/refman/5.7/en/datetime.html)  
[http://stackoverflow.com/questions/4594229/mysql-integer-vs-datetime-index](http://stackoverflow.com/questions/4594229/mysql-integer-vs-datetime-index)
