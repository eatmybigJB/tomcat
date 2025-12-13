```python
import json
import csv
from typing import List, Dict, Any


PLAN_FILE = "plan_output.json"
OUTPUT_FILE = "plan_resources_with_arn.csv"


def collect_resources(module: Dict[str, Any], results: List[Dict[str, str]]):
    """
    递归遍历 root_module / child_modules
    """
    for res in module.get("resources", []):
        # 只要 Terraform 管理的资源
        if res.get("mode") != "managed":
            continue

        resource_type = res.get("type")
        values = res.get("values", {}) or {}

        arn = values.get("arn")

        # plan 阶段不是所有资源都有 arn
        if arn:
            results.append({
                "resource_type": resource_type,
                "arn": arn
            })

    for child in module.get("child_modules", []):
        collect_resources(child, results)


def main():
    with open(PLAN_FILE, "r", encoding="utf-8") as f:
        plan = json.load(f)

    results: List[Dict[str, str]] = []

    root_module = plan.get("planned_values", {}).get("root_module")
    if root_module:
        collect_resources(root_module, results)

    # 去重（同一个 ARN 只保留一次）
    unique = {(r["resource_type"], r["arn"]) for r in results}

    with open(OUTPUT_FILE, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow(["resource_type", "arn"])
        for resource_type, arn in sorted(unique):
            writer.writerow([resource_type, arn])

    print(f"✅ Extracted {len(unique)} resources with ARN")
    print(f"📄 Output written to {OUTPUT_FILE}")


if __name__ == "__main__":
    main()



