```python
# default_label 是 s3_tf_state 里的子模块
output "label_name" {
  value = module.default_label.name
}

output "s3_label_name" {
  value = module.s3_tf_state.label_name
}
