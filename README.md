## What to do with aws-lambda-configuration (Problem to solve)  
- Maintain dynamic configuration. Using environment variable in lambda is in-flexible and hard to maintain.
- Read/Write/Delete configurations **across** lambda functions. Similar to what you did locally:  
```
var config = require('config');  
var id = config.get('id');  
```
  
## Why aws-lambda-configuration  
- Save you time for creating dynamoDB table, writing database I/O functions for every project.  
- Internal cache mechanism reduce redundant database access, hence lower dynamoDB capacity and lower cost.  
- A standardized way to access/manage/store configuration.  
- A trade-off solution between performance and running cost (propagation delay & access time & cheap lambda execution cost vs high dynamoDB capacity vs running cost of professional configuration system)  
  
## Cons  
- You need to accept a period of propagation delay after updating a config **IF** you use the cache mechanism to save dynamoDB access cost (default expire time 300s)  
  
## Preparation I (AWS Account & Key)  
You may skip this part if you already have and would like to keep using that access key/secret key.

1. Prepare an AWS account (Free Tier **SHOULD** cover this service's costs, ref: [AWS Free](https://aws.amazon.com/free/))  
2. \[Optional\] Create a machine user at [AWS IAM](https://console.aws.amazon.com/iam/home). Create User -> Tick Programmatic access -> Attach existing policies directly ->  
    2.a Choose the AdministratorAccess **or**  
    2.b Create Policy -> JSON -> Copy the Policy Sample at the end of this readme -> Review Policy -> Create Policy  
3. Prepare an set of credential (1 access key + 1 secret key), either from step 2 or create a key for your user. (ref: [AWS - Managing Access Keys for Your AWS Account](http://docs.aws.amazon.com/general/latest/gr/managing-aws-access-keys.html))  
4. Set up the aws-cli credential environment (ref: [AWS Configuration and Credential Files](http://docs.aws.amazon.com/cli/latest/userguide/cli-config-files.html)).    
5. \[Optional\] Test your credentials setting with aws-cli (ref: [AWS STS - get-caller-identity](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html))  
`aws sts get-caller-identity`  
You are expected to see the result:  
```
{  
    "Account": "123456789012",  
    "UserId": "AR#####:#####",  
    "Arn": "arn:aws:sts::123456789012:assumed-role/role-name/role-session-name"  
}  
```
  
## Preparation II (Core)  
You may skip the core if you sure you DO NOT want to use the cache mechanism. E.g., you just want a dynamoDB wrapper to access your configurations.  

1. Install the configuration core: [aws-lambda-configuration-core](https://github.com/tonyliu7870/aws-lambda-configuration-core)  
2. \[optional, for testing only\] Go to your [AWS DynamoDB Console](https://console.aws.amazon.com/dynamodb/home). Find your configuration table. Create a new item: 
```
{  
    "configName": "settings",  
    "data": {  
      "sampleConfig": "sampleValue"  
    }  
}  
```
  
## Preparation III (Encryption Key)  
You may skip this part if you DO NOT want to use encryption related functions.  

1. Go To [AWS IAM](https://console.aws.amazon.com/iam/home) -> Encryption keys -> Choose the region -> Create Key -> Input **Alias** (recommend: **lambda-configuration-key**) and Choose **KMS** for **Key Material Origin** -> Next Step x N -> Finish.  
  
2. Choose your created key -> Under Key Policy/Key Users -> Add ALL lambda execution roles that will use the encryption functions -> Save Changes  
  
## Configuration Standard  
aws-lambda-configuration use DynamoDB as storage. If you installed the core, there will be a dynamoDB table for you. If not, you need to prepare at least one table according to the following standard.  
- The table name (default: lambda-configurations) and item name (default: settings) is arbitrary and configurable.  
- The only requirement is the table MUST use a Primary partition key named **configName** (*String*).  
- Each config has the format:  
```
{
    "configName": string,
    "data": any(prefer object),
}
```
  
## Use aws-lambda-configuration  
There are few ways for you to interact with your configurations.  

1. Manually access via AWS Console  
    Go to [AWS DynamoDB Console](https://console.aws.amazon.com/dynamodb/home), find your table and directly access your config.  
  
2. Programmatic access via Core  
    aws-lambda-configuration-core comes with a lambda function (default: lambda-configuration). You can directly invoke the lambda function. Detail usage refer to [aws-lambda-configuration-core](https://github.com/tonyliu7870/aws-lambda-configuration-core).  
  
3. Programmatic access via library  
    Install the library/module to your serverless project.  
  
## Library List
JavaScript (TypeScript): [aws-lambda-configuration-js](https://github.com/tonyliu7870/aws-lambda-configuration-js)  
Python: In future  
  
## Best Practices
- Cache vs Core vs Direct:  
    - Use cache mechanism, get({ mode: 'cache' }), for config that are \[relatively stable & frequently get & allow short period of fault tolerant\], e.g. global server settings, cors settings, 3 parties credentials settings, etc.  
    - Use core, get({ mode: 'core' }), if you want to isolate permission, e.g. your lambda execution role only need `lambda:InvokeFunction` but not `dynamo:*`.  
    - Otherwise, use direct way, get().  
- Table Naming:  
    - Use tableName separates development stages, e.g. `lambda-configurations-dev`, `lambda-configurations-stage`, `lambda-configurations-prod`  
    - Use tableName separates projects, e.g. `projectA-configurations-dev`, `projectB-configurations-prod`  
    So that your project's settings will not be accidentally modified by some newer of this library using the same AWS account  
- Document Naming:  
    - Use standard documentName, e.g. `settings` for normal server settings (ids, name, domain, etc.) and `secrets` for sensitive settings (tokens, server credentials, keys, etc.)  
    - Use 1 document for 1 unit (user/device/account/etc), i.e. use documentName separates users' settings, e.g. a uuid/v4 string. Make sure you don't have a user called `settings` or `secrets`  
- Read/Write Configuration:  
    - If you need to read/write only 1 config, set `key: PATH_TO_SUB_OBJECT` to reduce transfer payload size.  
    - If you need to read/write multiple config, leave `key: undefined` to get/set the whole document to reduce multiple invoke times.   
    - Read **static** config outside the API handler, e.g. initializer, global variable, etc. to reduce repeated lambda invoke.  
- Security:  
    - Encrypt the sensitive data using library's functions: encrypt/encryptKEK.  
  
## Policy Sample (To be confirm)  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "dynamodb:*",
                "cloudwatch:*",
                "cloudformation:*",
                "iam:*",
                "lambda:*",
                "kms:*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```
  
## aws-lambda-configuration Performance
Memory Usage  
37MB + amount of cache  
  
Time (for getting a string config: "sampleValue")  
Cold Start + No cache: \~2700ms  
Warm + Without cache: 400ms\~1200ms  
With cache: 50ms\~200ms  
With cache & Very frequent access: 40ms\~80ms  
Direct access from library: 10ms\~70ms  
