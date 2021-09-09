
 **< UNDER CONSTRUCTION >**

### CockroachDB License Management

When an Enterprise license expires, there will not be any adverse impact on the cluster and nor will it result in any outages. However, when a trial license expires, certain enterprise features will stop working.

Licenses can be tracked with the prometheus metric seconds\_until\_license\_expiry that reports on the number of seconds until the enterprise license on the cluster expires. If the metric is a negative number, this means that the license expired in the past. If there is no license found, the metric will report as 0.

Please read the [steps for setting and verifying a license](https://www.cockroachlabs.com/docs/stable/licensing-faqs.html#set-a-license) for further information on setting or changing an expired license.



