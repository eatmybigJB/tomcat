```python
import csv
from itertools import islice
from typing import Iterable, Dict, Any, List, Optional

def write_users_to_two_csvs(
    users_iter: Iterable[Dict[str, Any]],
    tmp_csv_path: str,
    tmp_csv_path_full: str,
    attributes_full: List[str],
    attributes_limit: List[str],
    *,
    limit: Optional[int] = None,
    batch_size: int = 4000,
) -> None:
    """
    单次遍历 users_iter，同时写两份 CSV：
      - tmp_csv_path       只写 attributes_limit 列
      - tmp_csv_path_full  写 attributes_full 列
    不把生成器转为 list，不重复消费。
    """
    # 打开两个文件和对应 writer
    with open(tmp_csv_path, 'w', newline='', encoding='utf-8') as f_small, \
         open(tmp_csv_path_full, 'w', newline='', encoding='utf-8') as f_full:

        w_small = csv.DictWriter(f_small, fieldnames=attributes_limit, extrasaction='ignore')
        w_full  = csv.DictWriter(f_full,  fieldnames=attributes_full,  extrasaction='ignore')

        w_small.writeheader()
        w_full.writeheader()

        # 逐批从生成器取数据，避免一次性加载内存
        remaining = limit
        while True:
            take = batch_size if remaining is None else min(batch_size, remaining)
            batch = list(islice(users_iter, 0, take))
            if not batch:
                break

            # 写两份文件：按指定字段做投影，缺失补空串
            for u in batch:
                w_small.writerow({k: u.get(k, "") for k in attributes_limit})
                w_full.writerow({k: u.get(k, "") for k in attributes_full})

            if remaining is not None:
                remaining -= len(batch)
                if remaining <= 0:
                    break
write_users_to_two_csvs(
    users_gen,
    tmp_csv_path=tmp_csv_path,
    tmp_csv_path_full=tmp_csv_path_full,
    attributes_full=attributes_full,
    attributes_limit=attributes_limit,
    batch_size=4000  # 可按需调整
)
