```python
output "s3_label_name" {
  value = module.s3_tf_state.default_label[0].name
}
