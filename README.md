```python
with open(config_path, "rb") as f:
    config = tomllib.load(f)

server = config["server"]["host"]
share = config["server"]["share"]
username = config["auth"]["username"]
password = config["auth"]["password"]
remote_path = config["file"]["remote_path"]
