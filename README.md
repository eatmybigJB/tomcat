```python
fields @timestamp, @logStream, @message
| filter @message like '"currentMatchingId": "'
| parse @message '"currentMatchingId": "*"' as currentMatchingId
| display @timestamp, @logStream, currentMatchingId, @message
| sort @timestamp desc
| limit 10000
