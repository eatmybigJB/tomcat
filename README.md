```python
# svocid valid and not equal old svocid
current_matching_id = update_info['after'].get('currentMatchingId')
new_matching_id = update_info['after'].get('new_matchingId_id')
old_svoc_id = cognito_user_info.get('custom:svoc_id')

if current_matching_id and current_matching_id != old_svoc_id:
    update_item['custom:svoc_id'] = current_matching_id
elif new_matching_id and new_matching_id != old_svoc_id:
    update_item['custom:svoc_id'] = new_matching_id

return update_item
