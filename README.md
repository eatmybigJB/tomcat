```python
terraform plan --var-file=var/singbox.tfvars -out=tfplan.out
terraform show -no-color tfplan.out > plan_output.txt
