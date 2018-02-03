#Capturing Solr Metrics To a Log File

Solr 6.4+ has the ability to capture a lot of metrics. The[Metrics Reporting](http://lucene.apache.org/solr/guide/metrics-reporting.html)page gives a comprehensive introduction  


#Goal
I want to capture how my updates are performing, number of updates etc. and log them to a file every 60 seconds. 


#Solution
Firstly we need to define a separate logger which captures these metrics. We modify the `<solr-install>/server/resources/log4j.properties` file and add the following lines to define a new logger

```properties
# The logger is called updateMetricLogger. We will reference this logger to report metrics to a file
log4j.logger.updateMetricLogger = INFO, updateMetricFile
log4j.additivity.updateMetricLogger = false

#- size rotation with log cleanup.
log4j.appender.updateMetricFile=org.apache.log4j.RollingFileAppender
log4j.appender.updateMetricFile.MaxFileSize=4MB
log4j.appender.updateMetricFile.MaxBackupIndex=9

#- File to log to and log format
log4j.appender.updateMetricFile.File=${solr.log}/updateMetric.log
log4j.appender.updateMetricFile.layout=org.apache.log4j.EnhancedPatternLayout
log4j.appender.updateMetricFile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss.SSS} %-5p %m%n
```


Lastly we need to define the logging metric reporter which reports the metrics that we care about ( in this case update throughput metrics ) to a file.  

We add the following section to the `<solr-install>/server/solr/solr.xml` file 

```xml
<solr>
...
<metrics>
  <reporter name="log_update_stats" group="core" class="org.apache.solr.metrics.reporters.SolrSlf4jReporter">
    <int name="period">60</int>
    <str name="filter">UPDATE./update.requestTimes</str>
    <str name="logger">updateMetricLogger</str>
  </reporter>
</metrics>
...
</solr>
``` 