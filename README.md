# securityhub-alert配置指南
在所有ESS服务中只有Securityhub支持多regions聚合到一个region,在配置告警时,使用聚合后的region最为简便.
# 手动发送告警模式
-----------------------------------------------------------------------
## 参数设置
region为securityhub指定的聚合region
```
region='us-west-2'
rulename='manualalert'
email='**@**.com'
buttonname='send2email'
actionid='send2email'
```
## CLI 命令复制粘贴
```
snsarn=$(aws sns create-topic   --name  $rulename  --region=$region  --output text --query 'TopicArn')
aws sns subscribe --topic-arn $snsarn --protocol email --notification-endpoint  $email --region=$region
buttonarn=$(aws securityhub create-action-target \
    --name $buttonname\
    --description "send alert to email" \
    --id $actionid --region=$region  --output text --query 'ActionTargetArn')
aws events put-rule \
--name $rulename \
--event-pattern "{\"source\":[\"aws.securityhub\"], \
\"detail-type\": [\"Security Hub Findings - Custom Action\"], \
  \"resources\": [\"$buttonarn\"]}"  --region=$region
aws events put-targets --rule $rulename  --targets "Id"="1","Arn"=$snsarn --region=$region
```


## 打开eventbridge rule,复制以下内容至Target-Input transformer
### Input path
```
{
  "title": "$.detail.findings[0].Title",
  "Description": "$.detail.findings[0].Description",
  "account":"$.account",
  "region":"$.region"
  
}
```
### Template

```
"安全团队, there is an alert title : <title> in region:<region>"
"in account number:<account>"
"内容为:<Description>"
"请处理,谢谢!"
```
# 自动发送告警模式
-----------------------------------------------------------------------
## 参数设置
```
region='聚合后的region名:us-east-1'
rulename='autoalert3'
email='**@**.com'
```
## CLI 命令复制粘贴,此sample是只对High和Critical两类告警,可以自行修改
```
snsarn=$(aws sns create-topic   --name  $rulename  --region=$region  --output text --query 'TopicArn')
aws sns subscribe --topic-arn $snsarn --protocol email --notification-endpoint  $email --region=$region
aws events put-rule \
--name $rulename \
--event-pattern \
"{ \
  \"source\": [\"aws.securityhub\"],
  \"detail-type\": [\"Security Hub Findings - Imported\"],
  \"detail\": {
    \"findings\": {
      \"RecordState\": [\"ACTIVE\"],
      \"Severity\": {
        \"Label\": [\"HIGH\", \"CRITICAL\"]
      },
      \"Workflow\": {
        \"Status\": [\"NEW\"]
      }
    }
  } \
 }"  --region=$region
aws events put-targets --rule $rulename  --targets "Id"="1","Arn"=$snsarn --region=$region
```
## 打开eventbridge rule,配置邮件格式与手动发送告警模式相同
