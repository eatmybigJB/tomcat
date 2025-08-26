```python
configs = {
    "userpool": tomllib.load(open(userpool_config_path, "rb")),
    "smb": tomllib.load(open(smb_config_path, "rb")),
}

user_pool_id  = configs["userpool"]["Cognito"]["user_pool_id"]
server        = configs["smb"]["server"]["host"]
share         = configs["smb"]["server"]["share"]
username      = configs["smb"]["auth"]["username"]
password      = configs["smb"]["auth"]["password"]
remote_folder = configs["smb"]["file"]["remote_folder"]
remote_path   = f"{remote_folder}/test.txt"
