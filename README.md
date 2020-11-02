# aws-cli-sso-with-sdk-support
Quick hack to make the AWS SDK work locally after logging in to the AWS CLI with AWS SSO

## How it works

If you use AWS SSO to manage access to your AWS account, you can use the [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) (via  `aws sso login`) to login and enable CLI commands to use your SSO credentials

However, the CLI SSO login process does not write your SSO credentials to `~/.aws/credentials` which is used by the AWS SDK. Therefore, local SDK scripts will not work. 

Fortunately, there's an easy workaround. When you run `aws sso login`, your SSO temporary IAM access keys are copied to `~/.aws/cli/cache`. We can extract your credentials from this file and insert it into `~/.aws/credentials` to make the SDK work. 

Note that SSO does not initially write your file to ~/.aws/cli/cache until you execute an AWS CLI command after login. That's why the very first line in the demo script is `aws sts get-caller-identity`... this will make SSO write the credentials to the cache that we can then read from.

Now, each time you log in via `aws sso login`, a new file is added to the cache directory. Therefore, we need to use the newest file to get the proper credentials.

The bash script below does the job. You can run this after first executing `aws sso login`. Note - in my simple example, we overwrite the credentials file and use the `[default]` profile. Depending on your needs, you may want to insert (or find and replace) the credentials into the existing file while maintaining other profiles you have saved. 

```sh
aws sts get-caller-identity
LATEST_FILE=$(ls -t ~/.aws/cli/cache/* | head -1)
FILE_CONTENT=$(cat $LATEST_FILE)
ACCESS_KEY=$(echo $FILE_CONTENT | jq -r '.Credentials.AccessKeyId')
SECRET_KEY=$(echo $FILE_CONTENT | jq -r '.Credentials.SecretAccessKey')
TOKEN=$(echo $FILE_CONTENT | jq -r '.Credentials.SessionToken')

cat >~/.aws/credentials <<EOL
[default]
aws_access_key_id = ${ACCESS_KEY}
aws_secret_access_key = ${SECRET_KEY}
aws_session_token = ${TOKEN}
EOL
```
