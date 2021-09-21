# Alert: Node Non-Incrementing Uptime

### Purpose of this Alert

Node uptime is not incrementing. Check logs for possible panic loop.

<describe>



------

#### Alert Rule

| Tier     | Definition                                                   |
| -------- | ------------------------------------------------------------ |
| CRITICAL | 60 minutes - min (system uptime) during last  > 0 for 5 minutes |
| WARNING  | 60 minutes - min (system uptime) during last  > 0 for 2 minutes |



#### Alert Response

The actual response varies depending on the alert tier, i.e. the severity of potential consequences.

- Check ....

- 

