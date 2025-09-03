```python
fields @timestamp, @message, @logStream
| parse @message '"currentMatchingId": *' as currentMatchingId
| filter currentMatchingId != "null"
| sort @timestamp desc
| limit 10000
