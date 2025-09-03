```python
fields @timestamp, @message, @logStream, @log
| parse @message '"currentMatchingId": *"*" ' as currentMatchingId
| filter ispresent(currentMatchingId)
| sort @timestamp desc
| limit 10000
