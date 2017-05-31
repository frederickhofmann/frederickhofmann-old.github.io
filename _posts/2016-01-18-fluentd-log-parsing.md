---
layout: post
title: Better Log Parsing with Fluentd
subtitle: Description of a couple of approaches to designing your fluentd configuration.
category: dev
tags: [howto, devops, logging]
author: Doru Mihai
author_email: doru.mihai@haufe-lexware.com
header-img: "images/new/Exportiert_33.jpg"
---

When you will start to deploy your log shippers to more and more systems you will encounter the issue of adapting your solution to be able to parse whatever log format and source each system is using. Luckily, fluentd has a lot of plugins and you can approach a problem of parsing a log file in different ways.


The main reason you may want to parse a log file and not just pass along the contents is that when you have multi-line log messages that you would want to transfer as a single element rather than split up in an incoherent sequence.


Another reason would be log files that contain multiple log formats that you would want to parse into a common data structure for easy processing.
Below I will enumerate a couple of strategies that can be applied for parsing logs.

And last but not least, there is the case that you have multiple log sources (perhaps each using a different technology) and you want to parse them and aggregate all information to a common data structure for coherent analysis and visualization of the data.

## One Regex to rule them all
The simplest approach is to just parse all messages using the common denominator. This will lead to a very black-box type approach to your messages deferring any parsing efforts to a later time or to another component further downstream.


In the case of a typical log file a configuration can be something like this (but not necessarily):

~~~
<source>
  type tail
  path /var/log/test.log
  read_from_head true
  tag test.unprocessed
  format multiline
  format_firstline /\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}/
  #we go with the most generic pattern where we know a message will have
  #a timestamp in front of it, the rest is just stored in the field 'message'
  format1 /(?<time>\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}) (?<message>(.|\s)*)/
</source>
~~~
{: .language-xml}
You will notice we still do a bit of parsing, the minimal level would be to just have a multiline format to split the log contents into separate messages and then to push the contents on.

The reason we do not just put everything into a single field with a greedy regex pattern is to have the correct timestamp pushed showing the time of the log and not the time when the log message was read by the log shipper, along with the rest of the message.
If more pieces are common to all messages, it can be included in the regex for separate extraction, if it is of interest of course.

## Divide & Conquer
As the name would suggest, this approach suggests that you should try to create an internal routing that would allow you to precisely target log messages based on their content later on downstream.
An example of this is shown in the configuration below:

~~~
#Sample input:
#2015-10-15 08:19:05,190 [testThread] INFO  testClass      - Queue: update.testEntity; method: updateTestEntity; Object: testEntity; Key: 154696614; MessageID: ID:test1-37782-1444827636952-1:1:2:25:1; CorrelationID: f583ed1c-5352-4916-8252-47298732516e; started processing
#2015-10-15 06:44:01,727 [ ajp-apr-127.0.0.1-8009-exec-2] LogInterceptor                 INFO  user-agent: check_http/v2.1.1 (monitoring-plugins 2.1.1)
#connection: close
#host: test.testing.com
#content-length: 0
#X-Forwarded-For: 8.8.8.8
#2015-10-15 08:21:04,716 [ ttt-grp-127.0.0.1-8119-test-11] LogInterceptor                 INFO  HTTP/1.1 200 OK

<source>
   type tail
   path /test/test.log
   tag log.unprocessed
   read_from_head true

   format multiline
   format_firstline /\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}/
   format1 /(?<time>\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}) (?<message>(.|\s)*)/
</source>

<match log.unprocessed.**>
    type rewrite_tag_filter
    rewriterule1 message \bCorrelationID\b correlation
    rewriterule2 message .* clear
</match>
<match clear>
    type null
</match>

<filter correlation>
    type parser
    key_name message
    format / * (.*method:) (?<method>[^;]*) *(.*Object:) (?<object>[^;]*) *(.*Key:) (?<objectkey>[^;]*) *(.*MessageID:) (?<messageID>[^;]*) *(.*CorrelationID:) (?<correlationID>[^;]*).*/
    reserve_data yes
</filter>

<match correlation>
    type stdout
</match>
~~~
{: .language-ruby}

This approach is useful when we have multiline log messages within our logfile and the messages themselves have different formats for the content. Still, the important thing to note is that all log messages are prefixed by a standard timestamp, this is key to succesfully splitting messages correctly.


The break-down of the approach with the configuration shown is that all entries in the log are first parsed into individual events to be processed. The key separator here is the timestamp and it is marked by the *format_firstline* key/value pair as a regex pattern.
Fluentd will continue to read logfile lines and keep them in a buffer until a line is reached that starts with text that matches the regex pattern specified in the *format_firstline* field. After detecting a new log message, the one already in the buffer is packaged and sent to the parser defined by the regex pattern stored in the format<n> fields.


Looking at the example, all our log messages (single or multiline) will take the form:
~~~
{ "time":"2015-10-15 08:21:04,716", "message":"[ ttt-grp-127.0.0.1-8119-test-11] LogInterceptor                 INFO  HTTP/1.1 200 OK" }
~~~
{: .language-json}
Being tagged with log.unprocessed all the messages will be caught by the *rewrite_tag_filter* match tag and it is at this point that we can pinpoint what type of contents each message has and we can re-tag them for individual processing.

This module is key to the whole mechanism as the *rewrite_tag_filter* takes the role of a router. You can use this module to redirect messages to different processing modules or even outputs depending on the rules you define in it.

## Shooting fish in a barrel
You can use *fluent-plugin-multi-format-parser* to try to match each line read from the log file with a specific regex pattern (format).
This approach probably comes with performance drawbacks because fluentd will try to match using each regex pattern sequentially until one matches.
An example of this approach can be seen below:

~~~
<source>
  type tail
  path /var/log/aka/test.log
  read_from_head true
  keep_time_key true
  tag akai.unprocessed
  format multi_format
    # 2015-10-15 08:19:05,190 [externalPerson]] INFO  externalPersonToSapSystem      - Queue: aka.update.externalPerson; method: ; Object: externalPerson; Key: ; MessageID: ID:test1-37782-1444827636952-1:1:2:25:1; CorrelationID: f583ed1c-5352-4916-8252-47698732506e; received
    <pattern>
      format /(?<time>\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}) \[(?<thread>.*)\] (?<loglevel>[A-Z]*) * (.*method:) (?<method>[^;]*) *(.*Object:) (?<object>[^;]*) *(.*Key:) (?<objectkey>[^;]*) *(.*MessageID:) (?<messageID>[^;]*) *(.*CorrelationID:) (?<correlationID>[^;]*); (?<status>.*)/
    </pattern>
    # 2015-10-13 12:30:18,475 [ajp-apr-127.0.0.1-8009-exec-14] LogInterceptor                 INFO  Content-Type: text/xml; charset=UTF-8
    # Authorization: Basic UFJPRE9NT1xyZXN0VGVzdFVzZXI6e3tjc2Vydi5wYXNzd29yZH19
    # breadcrumbId: ID-haufe-prodomo-stage-51837-1444690926044-1-1731
    # checkoutId: 0
    # Content-Encoding: gzip
    # CS-Cache-Minutes: 0
    # CS-Cache-Time: 2015-10-13 12:30:13
    # CS-Client-IP: 172.16.2.51
    # CS-Inner-Duration: 207.6 ms
    # CS-Outer-Duration: 413.1 ms
    # CS-Project: PRODOMO
    # CS-UserID: 190844
    # CS-UserName: restTestUser
    # Expires: Thu, 19 Nov 1981 08:52:00 GMT
    # Server: Apache
    # SSL_CLIENT_S_DN_OU: Aka-Testuser
    # User-Agent: check_http/v2.1.1 (monitoring-plugins 2.1.1)
    # Vary: Accept-Encoding
    # workplace: 0
    # X-Forwarded-For: 213.164.69.219
    # X-Powered-By: PHP/5.3.21 ZendServer/5.0
    # Content-Length: 2883
    # Connection: close
    <pattern>
      format multiline
      format_firstline /\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}/
      format1 /(?<time>\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}) \[(?<thread>.*)\] (?<class>\w*) * (?<loglevel>[A-Z]*) (?<message>.*)/
    </pattern>

    #Greedy default format
    <pattern>
        format /(?<time>\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2},\d{3}) (?<message>(.|\s)*)/
    </pattern>
</source>
~~~
{: .language-ruby}
When choosing this path there are multiple issues you need to be aware of:
* The pattern matching is done sequentially and the first pattern that matches the message is used to parse it and the message is passed along
* You need to make sure the most specific patterns are higher in the list and the more generic ones lower
* Make sure to create a very generic pattern to use as a default at the end of the list of patterns.
* Performance will probably decrease due to the trial&error approach to finding the matching regex


The biggest issue with this approach is that it is very very hard to handle multi-line log messages if there are significantly different log syntaxes in the log.

__Warning:__ Be aware that the multiline parser continues to store log messages in a buffer until it matches another firstline token and when it does it then it packages and emits the multiline log it collected.
This approach is useful when you have good control and know-how about the format of your log source.

## Order & Chaos
Introducing Grok!

Slowly but surely getting all your different syntaxes, for which you will have to define different regular expressions, will make your config file look very messy, filled with regex-es that are longer and longer, and just relying on the multiple format lines to split it up doesn't bring that much readability nor does it help with maintainability. Reusability is something that we cannot even discuss in the case of pure regex formatters.


Grok allows you to define a library of regexes that can be reused and referenced via identifiers. It is structured as a list of key-value pairs and can also contain named capture groups.
An example of such a library can be seen below. (Note this is just a snippet and does not contain all the minor expressions that are referenced from within the ones enumerated below)

~~~
###
#  AKA-I
###
# Queue: aka.update.externalPerson; method: updateExternalPerson; Object: externalPerson; Key: ; MessageID: ID:test1-37782-1444827636952-1:1:2:25:1; CorrelationID: f583ed1c-5352-4916-8252-47698732506e; received
AKA_AKAI_CORRELATIONLOG %{AKA_AKAI_QUEUELABEL} %{AKA_AKAI_QUEUE:queue}; %{AKA_AKAI_METHODLABEL} %{AKA_AKAI_METHOD:method}; %{AKA_AKAI_OBJECTLABEL} %{AKA_AKAI_OBJECT:object}; %{AKA_AKAI_KEYLABEL} %{AKA_AKAI_KEY:key}; %{AKA_AKAI_MESSAGEIDLABEL} %{AKA_AKAI_MESSAGEID:messageId}; %{AKA_AKAI_CORRELATIONLABEL} %{AKA_AKAI_CORRELATIONID:correlationId}; %{WORD:message}
#CorrelationId log message from AKAI
#2015-10-15 08:19:05,190 [externalPerson]] INFO  externalPersonToSapSystem      - Queue: aka.update.externalPerson; method: updateExternalPerson; Object: externalPerson; Key: ; MessageID: ID:test1-37782-1444827636952-1:1:2:25:1; CorrelationID: f583ed1c-5352-4916-8252-47698732506e; received
AKA_AKAI_CORRELATIONID %{AKAIDATESTAMP:time} %{AKA_THREAD:threadName} %{LOGLEVEL:logLevel} * %{JAVACLASS:className} %{AKA_AKAI_CORRELATIONLOG}
#Multiline generic log pattern
# For detecting that a new log message has been read we will use AKAIDATESTAMP as the pattern and then match with a greedy pattern
AKA_AKAI_GREEDY %{AKAIDATESTAMP:time} %{AKA_THREAD:threadName} %{JAVACLASS:class} * %{LOGLEVEL:loglevel} %{AKA_GREEDYMULTILINE:message}
# Variation since some log messages have loglevel before classname or vice-versa
AKA_AKAI_GREEDY2 %{AKAIDATESTAMP:time} %{AKA_THREAD:threadName} * %{LOGLEVEL:loglevel} %{JAVACLASS:class} %{AKA_GREEDYMULTILINE:message}
###
# ARGO Specific
###
AKA_ARGO_DATESTAMP %{MONTHDAY}-%{MONTH}-%{YEAR} %{TIME}
#17-Nov-2015 07:53:38.786 INFO [www.lexware-akademie.de-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /var/www/html/www.lexware-akademie.de/ROOT has finished in 44,796 ms
AKA_ARGO_LOG %{AKA_ARGO_DATESTAMP:time} %{LOGLEVEL:logLevel} %{AKA_THREAD:threadName} %{JAVACLASS:className} %{AKA_GREEDYMULTILINE:message}
#2015-11-17 07:53:51.606 RVW  INFO  AbstractApplicationContext:515  - Refreshing Root WebApplicationContext: startup date [Tue Nov 17 07:53:51 CET 2015]; root of context hierarchy
AKA_ARGO_LOG2 %{AKAIDATESTAMP2:time} %{WORD:argoComponent} *%{LOGLEVEL:logLevel} * %{AKA_CLASSLINENUMBER:className} %{AKA_GREEDYMULTILINE:message}
#[GC (Allocation Failure) [ParNew: 39296K->4351K(39296K), 0.0070888 secs] 147064K->114083K(172160K), 0.0071458 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
#[CMS-concurrent-abortable-preclean: 0.088/0.233 secs] [Times: user=0.37 sys=0.02, real=0.23 secs]
AKA_ARGO_SOURCE (GC|CMS)
AKA_ARGO_GC \[%{AKA_ARGO_SOURCE:source} %{AKA_GREEDYMULTILINE:message}
~~~
{: .language-bash}

To use Grok you will need to install the *fluent-plugin-grok-parser* and then you can use grok patterns with any of the other techniques previously described with regex: Multiline, Multi-format.

# Go Get'em!

Now you should have a pretty good idea of how you can approach different log formats and how you can structure your config file using a couple of plugins from the hundreds of plugins available.
