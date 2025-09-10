```python
{
  "Id": "cognito-cmk-policy",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow service access (Identity Store) - optional",
      "Effect": "Allow",
      "Principal": { "Service": [ "identitystore.amazonaws.com" ] },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKeyWithoutPlainText"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow Amazon Cognito service access (User Pool)",
      "Effect": "Allow",
      "Principal": { "Service": [ "cognito-idp.amazonaws.com" ] },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKeyWithoutPlainText"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:aws-crypto-ec:aws:cognito-idp:userpool:arn": "arn:aws:cognito-idp:<region>:<account-id>:userpool/<user-pool-id>",
          "aws:SourceAccount": "<account-id>"
        },
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:cognito-idp:<region>:<account-id>:userpool/<user-pool-id>"
        }
      }
    },
    {
      "Sid": "Allow Amazon Cognito service DescribeKey access",
      "Effect": "Allow",
      "Principal": { "Service": [ "cognito-idp.amazonaws.com" ] },
      "Action": [ "kms:DescribeKey" ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:aws-crypto-ec:aws:cognito-idp:userpool:arn": "arn:aws:cognito-idp:<region>:<account-id>:userpool/<user-pool-id>"
        }
      }
    },
    {
      "Sid": "Allow service DescribeKey access (Identity Store) - optional",
      "Effect": "Allow",
      "Principal": { "Service": [ "identitystore.amazonaws.com" ] },
      "Action": [ "kms:DescribeKey" ],
      "Resource": "*"
    },

    // ========  替换 CloudWatch Logs 的两段，换成 Firehose 版本  ========

    {
      "Sid": "Allow Amazon Cognito to encrypt user logs for export (to Firehose)",
      "Effect": "Allow",
      "Principal": { "Service": [ "cognito-idp.amazonaws.com" ] },
      "Action": [ "kms:GenerateDataKey", "kms:Encrypt" ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:SourceArn": "arn:aws:firehose:<region>:<account-id>:deliverystream/<firehose-name>"
        }
      }
    },
    {
      "Sid": "Allow Firehose to decrypt user logs and deliver",
      "Effect": "Allow",
      "Principal": { "Service": [ "firehose.amazonaws.com" ] },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "<account-id>"
        },
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:firehose:<region>:<account-id>:deliverystream/<firehose-name>"
        }
      }
    }
  ]
}
