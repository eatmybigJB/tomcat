```python
fields @timestamp, @logStream, @message
| parse @message '"operationType":"*"' as operationType
| parse @message '"matchingId":"*"'  as matchingId
| filter operationType = "delete" and matchingId = "ab23eb9f-a528-473f-a6f5-2d7402744ee7"  -- 改成你的ID
| sort @timestamp desc
| limit 10000
