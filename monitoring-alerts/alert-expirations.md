# Alert: Enterprise License Expiration

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

---------------------------------





# Alert: Security Certificate Expiration

### Purpose of this Alert

Avoid security certificate expiration.

------

### Monitoring Metric

```
security.certificate.expiration.ca
security.certificate.expiration.client-ca
security.certificate.expiration.ui-ca
security.certificate.expiration.node
security.certificate.expiration.node-client
security.certificate.expiration.ui
```

Expiration timestamp in seconds since Unix epoch for the certificate. 0 means no certificate or error.

Set the alert for each type of certificate.



### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| WARNING  | Less than 1814400 seconds (3 weeks) until enterprise license expiry |
| CRITICAL | Less than 259200 seconds (3 days) until enterprise license expiry |

### Alert Response

Rotate the expiring certificates.
