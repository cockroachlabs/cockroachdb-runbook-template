seconds_until_enterprise_license_expiry	gauge	

# Alert: Enterprise License Expiry

### Purpose of this Alert

Avoid enterprise license expiration.

------

### Monitoring Metric

```
seconds.until.enterprise.license.expiry
```

Seconds until enterprise license expiry. If no license is present, this metric is 0.



### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| WARNING  | Less than 1814400 seconds (3 weeks) until enterprise license expiry |
| CRITICAL | Less than 259200 seconds (3 days) until enterprise license expiry |

### Alert Response

Renew the enterprise license.
