```python
terraform state list \
| awk -F. '
{
  if ($1=="module") {
    printf "%-15s %-35s %s\n", $2, $3"."$4, $5
  } else {
    printf "%-15s %-35s %s\n", "root", $1"."$2, $3
  }
}'



