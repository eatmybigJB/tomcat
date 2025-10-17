```python
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowInvokeFromThisVPCE",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:ap-east-1:381492094486:ljhrt4uewe/*",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "vpce-0d563070568d4ffd42"
        }
      }
    }
  ]
}
