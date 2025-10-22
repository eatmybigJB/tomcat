```python
from typing import Iterable, Dict, Any, List, Optional
from itertools import islice
from concurrent.futures import ThreadPoolExecutor
import csv

def write_users_to_csv(
    users_iter: Iterable[Dict[str, Any]],
    filename: str,
    *,
    fieldnames: Optional[List[str]] = None,   # 新增：用来控制列以及顺序
    limit: Optional[int] = None,
    batch_size: int = 100,
    max_workers: int = 4,
) -> None:
    # 预取（也确保生成器只被消费一次）
    users = list(islice(users_iter, limit))
    if not users:
        print("No users to write.")
        return

    # 如果未显式传入字段，则动态收集所有键（无固定顺序）
    if fieldnames is None:
        keys = set()
        for u in users:
            keys.update(u.keys())
        fieldnames = list(keys)

    # 批处理函数：投影到指定字段，缺失补空
    def process_batch(batch):
        out = []
        for user in batch:
            row = {k: user.get(k, "") for k in fieldnames}
            out.append(row)
        return out

    processed_rows: List[Dict[str, Any]] = []
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = []
        for i in range(0, len(users), batch_size):
            batch = users[i:i + batch_size]
            futures.append(executor.submit(process_batch, batch))
        for f in futures:
            processed_rows.extend(f.result())

    # 统一写盘
    with open(filename, mode='w', newline='', encoding='utf-8') as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction='ignore')
        writer.writeheader()
        writer.writerows(processed_rows)

# 假设你已有一个列表 users_list（或任意可迭代的 dict）
cols = ["Username", "preferred_username", "UserStatus", "custom:svoc_id"]

write_users_to_csv(
    iter(users_list),
    tmp_csv_path_full,
    fieldnames=cols,          # 明确列与顺序
    batch_size=4000,
    max_workers=8
)
