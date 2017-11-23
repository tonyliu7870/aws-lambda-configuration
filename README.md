# What to do with aws-lambda-configuration  
- Maintain dynamic configuration. Using environment variable in lambda is in-flexible and hard to maintain.
- Read/Write/Delete configurations **across** lambda functions. Similar to what you did locally:  
```
var config = require('config');  
var id = config.get('id');  
```
  
# Why aws-lambda-configuration  
- Save you time to create dynamoDB table, write database I/O functions every project.  
- Internal cache mechanism improves access time 2-10 times faster 
  
# Preparation  
1. Prepare an AWS account (Free Tier **SHOULD** cover this service's costs, ref: [AWS Free](https://aws.amazon.com/free/))  
2. Prepare an set of credential (1 access key + 1 secret key)  
3. Set up the aws-cli credential environment (ref: [AWS Configuration and Credential Files](http://docs.aws.amazon.com/cli/latest/userguide/cli-config-files.html)). Just the credential, **NOT required** the aws-cli tools.  
4. \[Optional\] Test your credentials setting with aws-cli (ref: [AWS STS - get-caller-identity](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html))
`aws sts get-caller-identity`  
You are expected to see the result:  
```
{  
    "Account": "123456789012",  
    "UserId": "AR#####:#####",  
    "Arn": "arn:aws:sts::123456789012:assumed-role/role-name/role-session-name"  
}  
```
5. Intall the configuration core: [aws-lambda-configuration-core](https://github.com/tonyliu7870/aws-lambda-configuration-core)  
6. Go to your [AWS DynamoDB Console](https://console.aws.amazon.com/dynamodb/home). Find your configuration table. Create a new item: 
```
{  
    "configName": "settings",  
    "data": {  
      "sampleConfig": "sampleValue"  
    }  
}  
```
7. Install the library/module to your serverless project.  
  
# Library List
JavaScript (TypeScript): [aws-lambda-configuration-js](https://github.com/tonyliu7870/aws-lambda-configuration-js)  
Python: In future  
  
# AWS statistics
Memory  
AWS Lambda basic memory usage: ~34MB (empty code, default aws-sdk library)  
  
Time  
AWS Lambda invoke time: 50ms~100ms (included both call & return)  
AWS Lambda cold start time: <50ms  
AWS DynamoDB simple query: ~1200ms (single document, directly get by key)  
  
# aws-lambda-configuration Performance
Memory Usage
37MB + amount of cache  
  
Time (for a string config: "sampleValue")  
Cold Start + No cache: ~2700ms  
Warm + Without cache: 400ms~600ms  
With cache: 50ms~200ms  
  
# Best Practices
- Table Naming:  
    - Use tableName separates development stages, e.g. `lambda-configurations-dev`, `lambda-configurations-stage`, `lambda-configurations-prod`  
    - Use tableName separates projects, e.g. `projectA-configurations-dev`, `projectB-configurations-prod`  
    So that your project's settings will not be accidentally modified by some newer of this library using the same AWS company account  
- Document Naming:  
    - Use standard documentName, e.g. `settings` for normal server settings (ids, name, domain, etc.) and `secrets` for sensitive settings (tokens, server credentials, keys, etc.)  
    - Use 1 document for 1 unit (user/device/account/etc), i.e. use documentName separates users' settings, e.g. a uuid/v4 string. Make sure you don't have a user called `settings` or `secrets`  
- Read/Write Configuration:  
    - If you need to read/write only 1 config, set `key: PATH_TO_SUB_OBJECT` to reduce transfer payload size.  
    - If you need to read/write multiple config, leave `key: undefined` to get/set the whole document to reduce multiple invoke times.  
    - Use `noCache: true` to get config that you may modify without using the core (e.g. you are keeping trial & error to develop the program).  
    - Read **static** config outside the API handler, e.g. initializer, global variable, etc. to reduce unnecessary lambda invoke.  
- Security:  
    - Encrypt the sensitive data yourself.  
