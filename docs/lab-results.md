\# SQL Programmability Lab Results



\## Baseline



Before making any changes, the direct revenue query returned:



| Operator | Revenue date | Amount | Payments |

|---|---|---:|---:|

| OP-BUS | 2026-04-29 | 36.00 | 1 |

| OP-METRO | 2026-04-29 | 36.00 | 1 |



\## SQL objects



The SQL definitions from the starter files were copied to SQL files and applied to the database.



The lab uses a SQL function, a materialized view and a trigger maintained summary table.



\## Case 1 - Captured payment insert



I inserted a new captured payment of 36 DKK for TICKET-1, which belongs to OP-METRO.



| Approach | Amount | Payments |

|---|---:|---:|

| Direct query | 72.00 | 2 |

| Function | 72.00 | 2 |

| Materialized view | 36.00 | 1 |

| Trigger summary | 36.00 | 1 |



The direct query and function both show 72.00 because they read from the current data.



The materialized view still shows 36.00 because it has not been refreshed, so the data is stale.



The trigger summary also shows 36.00. This is because the trigger only saw the new payment. The old payment was already in the database before the trigger was created.



\## Case 2 - Failed payment insert



I inserted a failed payment of 50 DKK for OP-METRO.



| Approach | Amount | Payments |

|---|---:|---:|

| Direct query | 72.00 | 2 |

| Function | 72.00 | 2 |

| Materialized view | 36.00 | 1 |

| Trigger summary | 36.00 | 1 |



The results did not change because failed payments should not count as captured revenue.



The trigger also did not add the payment to the summary because its status was Failed.



\## Case 3 - Failed to Captured



I changed the 50 DKK payment from Failed to Captured.



| Approach | Amount | Payments |

|---|---:|---:|

| Direct query | 122.00 | 3 |

| Function | 122.00 | 3 |

| Materialized view | 36.00 | 1 |

| Trigger summary | 36.00 | 1 |



The direct query and function now show 122.00 because the payment is Captured.



The materialized view still shows the old data because it has not been refreshed.



The trigger summary does not change because the trigger only runs on inserts and not updates.



\## Case 4 - Captured to Refunded



I changed the 36 DKK payment from Captured to Refunded.



| Approach | Amount | Payments |

|---|---:|---:|

| Direct query | 86.00 | 2 |

| Function | 86.00 | 2 |

| Materialized view | 36.00 | 1 |

| Trigger summary | 36.00 | 1 |



The direct query and function now show 86.00 because the refunded payment no longer counts as captured revenue.



The materialized view still shows the old data because it has not been refreshed.



The trigger summary does not change because the trigger does not run on updates.



\## Case 5 - Delete payment



I deleted the 50 DKK payment that was changed to Captured earlier.



| Approach | Amount | Payments |

|---|---:|---:|

| Direct query | 36.00 | 1 |

| Function | 36.00 | 1 |

| Materialized view | 36.00 | 1 |

| Trigger summary | 36.00 | 1 |



All four now show the same result, but the trigger summary and materialized view were not actually updated by the delete. They just happen to have the same result as the current data.



\## Case 6 - Duplicate payment



I inserted a payment with an external payment reference that already existed in the database.



The payment was still accepted.



| Approach | Amount | Payments |

|---|---:|---:|

| Direct query | 72.00 | 2 |

| Function | 72.00 | 2 |

| Materialized view | 36.00 | 1 |

| Trigger summary | 72.00 | 2 |



The direct query and function count the duplicate because it is stored as a Captured payment.



The materialized view still has the old data because it has not been refreshed.



The trigger also counts the duplicate because it is a new Captured payment.



So the same external payment can be counted more than once.





\## Materialized view refresh



After the test cases I refreshed the materialized view.



Before the refresh OP-METRO showed 36.00 and 1 payment.



After the refresh it showed 72.00 and 2 payments.



This shows that the materialized view does not update by itself. It needs to be refreshed to get the current data.





\## Side-effect trace



When a payment is inserted into the payments table, PostgreSQL first checks the primary key on the payment id.



After the insert, the payments\_daily\_revenue\_after\_insert trigger runs.



If the payment has status Captured, the trigger finds the operator from the ticket, trip and route.



It then inserts or updates a row in daily\_revenue\_by\_operator. This table has a primary key on operator and date, and a foreign key to the operators table.



The payment and the trigger update are part of the same transaction. If something fails, the changes are not committed.



The direct query and function read from the payments table, so they get the new data.



The trigger summary is updated when a Captured payment is inserted.



The materialized view is not updated until it is refreshed.



\## Responsibility matrix



| Approach | Correctness | Freshness | Write cost | Read cost | Hidden side effects | Rebuild |

|---|---|---|---|---|---|---|

| Direct query | Good | Current data | Low | Higher | No | Not needed |

| Function | Good | Current data | Low | Higher | No | Not needed |

| Materialized view | Good after refresh | Can be stale | Low | Low | No | Refresh from payments |

| Trigger summary | Can become incorrect | Updated on insert | Higher | Low | Yes | Rebuild from payments |



The payments table is the authority for the revenue data.



The direct query and function always calculate the result from payments.



The materialized view stores the result and needs a refresh to get the current data.



The trigger summary is updated when a payment is inserted, but our trigger does not handle updates or deletes. This can make the summary different from payments.



\## Issue register



\### Duplicate payments



The database allows more than one payment with the same external payment reference.



In case 6 the duplicate payment was accepted and counted as captured revenue.



This can cause the revenue to be counted more than once.





\## Decision record



I would use the materialized view for this case.



The report is read often and it is okay if the data is delayed.



The payments table stays as the authority for the revenue data.



The materialized view can be refreshed on a schedule to get the latest data.



If payments are corrected or deleted, the materialized view can be rebuilt from the payments data.



This also avoids the problems we saw with the trigger when payments were updated or deleted.

