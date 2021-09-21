# Alert: Live Node Count Change

### Purpose of this Alert

Live node count change

<Formula doesn't make sense?>



------

### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | min cluster (liveness.livenodes) - min cluster  (liveness.livenodes) as of 5 minutes ago  < 0 for 90 minutes |
| WARNING  | min cluster (liveness.livenodes) - min cluster  (liveness.livenodes) as of 2 minutes ago  < 0 for 90 minutes |



### Alert Response

The actual response varies depending on the alert tier, i.e. the severity of potential consequences.

- Check ....

- 

