```python
{
  "Sid": "AllowCognitoDataPlaneForThisUserPool",
  "Effect": "Allow",
  "Principal": { "Service": "cognito-idp.amazonaws.com" },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "cognito-idp.ap-east-1.amazonaws.com",
      "kms:CallerAccount": "730335524365"
    },
    "StringLike": {
      "kms:EncryptionContext:aws-crypto-ec:aws:cognito-idp:userpool-arn": "arn:aws:cognito-idp:ap-east-1:730335524365:userpool/*"
      // 若只允许一个特定池，改成具体 ARN：arn:aws:cognito-idp:ap-east-1:730335524365:userpool/ap-east-1_xxxxxxx
    }
  }
}
