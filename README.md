```python
fields @timestamp, @message
| filter @message like '"currentMatchingId": "'
| parse @message '"currentMatchingId": "*"' as currentMatchingId
| stats
    count() as occurrences,
    min(@timestamp) as firstSeen,
    max(@timestamp) as lastSeen
  by currentMatchingId
| sort occurrences desc, lastSeen desc
| limit 10000
