```python
import csv

CSV1 = "csv1.csv"
CSV2 = "csv2.csv"
OUTPUT = "matched.csv"


def normalize_email(email: str) -> str:
    """去掉前后空格 + 忽略大小写"""
    if email is None:
        return ""
    return email.strip().lower()


def main():
    # 读取 csv2，构造 email → 行 的映射
    csv2_map = {}
    with open(CSV2, "r", newline="", encoding="utf-8") as f2:
        reader = csv.DictReader(f2)
        for row in reader:
            email2 = normalize_email(row.get("email", ""))
            if email2:
                csv2_map[email2] = row

    # 读取 csv1，逐行匹配 csv2
    output_rows = []
    with open(CSV1, "r", newline="", encoding="utf-8") as f1:
        reader1 = csv.DictReader(f1)

        for row1 in reader1:
            raw_email = row1.get("email", "")
            email_key = normalize_email(raw_email)

            if email_key in csv2_map:
                matched_row = {"email": raw_email}  # 保留原始 email
                matched_row.update(csv2_map[email_key])  # 合并 csv2 的信息
                output_rows.append(matched_row)

    if not output_rows:
        print("没有匹配到任何行")
        return

    # 写到新的 CSV
    # 输出列：email（来自 csv1） + csv2 的所有列
    fieldnames = list(output_rows[0].keys())

    with open(OUTPUT, "w", newline="", encoding="utf-8") as fo:
        writer = csv.DictWriter(fo, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(output_rows)

    print(f"匹配完成，共输出 {len(output_rows)} 行到 {OUTPUT}")


if __name__ == "__main__":
    main()
