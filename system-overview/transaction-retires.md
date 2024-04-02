# Transactions: Retires



### About Transaction Retires

A database transaction that is forced to restart will either be retried automatically by CockroachDB on the server side, or may receive a [retry error](https://www.cockroachlabs.com/docs/v21.2/error-handling-and-troubleshooting.html#transaction-retry-errors) `40001` to the client and then the transaction must be retried on the client side by the application logic.



### Automatic Server Side Retires

An automatic server side retry is relatively inexpensive. The statement is retried within the existing transaction, which has had its timestamp changed.

A server side retry avoids an extra network roundtrip to the client.



### Client Side Retires

Client-side retries have higher performance overhead than the automatic server side retries. CockroachDB will make the best effort to retry automatically, however it can only be done for the first statement of a transaction and only if no result has been returned to the client yet (the opportunity for automatic retry is lost if the client received any result). If an application uses multi-statement transactions that have long running selects with large scans and/or produce large result sets - it would make client retires very likely, with an associated performance overhead.



### Useful Resources

 [Client Side Intervention](https://www.cockroachlabs.com/docs/v21.2/transactions.html#client-side-intervention)

