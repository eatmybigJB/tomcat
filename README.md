```python
sum by (topic) (
  increase(kafka_connect_source_task_metrics_source_record_write_total{connector="你的连接器名"}[10m])
)

