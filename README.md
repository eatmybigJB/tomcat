```python
import csv
import re
from datetime import datetime
import os

# 当前脚本目录
script_dir = os.path.dirname(os.path.realpath(__file__))

# 修改这里的文件路径
a_csv = os.path.join(script_dir, "a.csv")        # 文件 A：yyyy-mm-dd
b_csv = os.path.join(script_dir, "b.csv")        # 文件 B：m/d/yyyy
c_csv = os.path.join(script_dir, "c.csv")        # 输出文件

def parse_date_a(s: str):
    try:
        return datetime.strptime(s.strip(), "%Y-%m-%d").date()
    except Exception:
        return None

def parse_date_b(s: str):
    s = (s or "").strip()
    m = re.search(r'(\d{1,2})/(\d{1,2})/(\d{4})', s)
    if not m:
        return None
    mm, dd, yyyy = m.groups()
    try:
        return datetime(int(yyyy), int(mm), int(dd)).date()
    except Exception:
        return None

# 读取 A
a_map = {}
with open(a_csv, "r", encoding="utf-8-sig", newline="") as fa:
    reader = csv.DictReader(fa)
    for row in reader:
        nric = (row.get("nric") or "").strip()
        d = parse_date_a(row.get("birthdate") or "")
        a_map[nric] = d

# 写输出
with open(b_csv, "r", encoding="utf-8-sig", newline="") as fb, \
     open(c_csv, "w", encoding="utf-8", newline="") as fc:

    reader = csv.DictReader(fb)
    writer = csv.DictWriter(fc, fieldnames=["nric", "birthdate", "correct as a"])
    writer.writeheader()

    for row in reader:
        nric = (row.get("nric") or "").strip()
        birth_b = (row.get("birthdate") or "").strip()
        date_a = a_map.get(nric)
        date_b = parse_date_b(birth_b)

        result = "yes" if date_a and date_b and date_a == date_b else "no"
        writer.writerow({
            "nric": nric,
            "birthdate": birth_b,
            "correct as a": result
        })

print(f"比较完成，结果已写入: {c_csv}")
