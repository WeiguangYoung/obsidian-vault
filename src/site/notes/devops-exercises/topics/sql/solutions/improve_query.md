---
{"dg-publish":true,"permalink":"/devops-exercises/topics/sql/solutions/improve_query/","dg-note-properties":{}}
---


## Comparisons vs. Functions - Solution

```
SELECT count(*)
FROM shawarma_purchases
WHERE
  purchased_at >= '2017-01-01' AND
  purchased_at <= '2017-31-12'
```
