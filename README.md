```python
fields @timestamp, @logStream, @message
| filter @message like '"matchingId": "2df4dd77-7e51-4b29-9599-9473ebd5ce79"'
| sort @timestamp desc
| limit 10000
