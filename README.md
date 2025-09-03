```python
fields @timestamp, @logStream, @message
| filter @message like '"currentMatchingId":'
| parse @message '"currentMatchingId": *' as currentMatchingId
| filter currentMatchingId != null and currentMatchingId != "null"
| display @timestamp, @logStream, currentMatchingId, @message
| sort @timestamp desc
| limit 10000
