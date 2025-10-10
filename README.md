```python
sum by (topic) (
  increase(kafka_producer_producer_topic_metrics_record_send_total{job="HK-Juniper-kafka-connect"}[10m])
)
