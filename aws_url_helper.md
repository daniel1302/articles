Last time, when I developed some integration with AWS, I got in trouble when I wanted to generate a direct URL to the Cloudwatch metrics. I could not find any working solution, so I decided to write my implementation.

I expect to get a URL to AWS Cloudwatch metrics for given metrics data.

### Get direct URL for AWS Cloudwatch metrics

What data do I have?

- Statistic,
- Dimensions,
- MetricName,
- Namespace,
- Period

You can find source code for the helper I implemented on my GitHub: [AwsMetricUrlHelper.py](https://github.com/daniel1302/aws_helpers/blob/master/AwsMetricUrlHelper.py)

### Example of usage:

```python
metric_data = """
[
    {
        "MetricName": "prod-ecommerce-front-http-response-status-5xx",
        "Namespace": "LogMetrics",
        "StatisticType": "Statistic",
        "Statistic": "SUM",
        "Unit": null,
        "Dimensions": [],
        "Period": 300,
        "EvaluationPeriods": 1,
        "ComparisonOperator": "GreaterThanOrEqualToThreshold",
        "Threshold": 1.0,
        "TreatMissingData": "- TreatMissingData: missing",
        "EvaluateLowSampleCountPercentile": ""
    }
]
"""

metric_data2 = """
[
    {
        "MetricName": "RequestCount",
        "Namespace": "AWS/ApplicationELB",
        "StatisticType": "Statistic",
        "Statistic": "SUM",
        "Unit": null,
        "Dimensions": {
            "TargetGroup": "targetgroup/prod-application-api-alb-tg/XXXXXX33c4073540",
            "LoadBalancer": "app/prod-application-api-alb/XXXXXXb12cae3d294",
            "AvailabilityZone": "us-east-1a"
        },
        "Period": 300,
        "EvaluationPeriods": 1,
        "ComparisonOperator": "GreaterThanOrEqualToThreshold",
        "Threshold": 1.0,
        "TreatMissingData": "- TreatMissingData: missing",
        "EvaluateLowSampleCountPercentile": ""
    }
]
"""

url_helper = AwsMetricUrlHelper('us-east-1')
print(url_helper.url(json.loads(metric_data), 60, 6))
print(url_helper.url(json.loads(metric_data2), 60, 6))
```

### Output

```
https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#metricsV2:graph=~(view~'timeSeries~period~60~stacked~false~region~'us-east-1~start~'2019-11-28T11*3A01*3A47~end~'2019-11-28T17*3A01*3A47~metrics~(~(~'LogMetrics~'prod-ecommerce-front-http-response-status-5xx~(stat~'Sum))))
https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#metricsV2:graph=~(view~'timeSeries~period~60~stacked~false~region~'us-east-1~start~'2019-11-28T11*3A01*3A47~end~'2019-11-28T17*3A01*3A47~metrics~(~(~'AWS/ApplicationELB~'RequestCount~'TargetGroup~'targetgroup/prod-application-api-alb-tg/XXXXXX33c4073540~'LoadBalancer~'app/prod-application-api-alb/XXXXXXb12cae3d294~'AvailabilityZone~'us-east-1a~(stat~'Sum))))
```
