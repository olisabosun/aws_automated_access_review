#  This provides explanation line by line of the deploy.sh script used for deployment.

###!/bin/bash
### This is called a "shebang" - it tells the system to run this script using bash shell

set -e
### This tells bash to exit immediately if any command fails (returns non-zero exit code)
### Without this, the script would continue even if something goes wrong

### Configuration
### These are variables that store default values for the script
STACK_NAME="aws-access-review"
### STACK_NAME: The name of the CloudFormation stack that will be created in AWS

REGION="us-east-1"  ### Default region
### REGION: The AWS region where resources will be deployed (Virginia data center)

SCHEDULE="rate(30 days)"  ### Default: run every 30 days
### SCHEDULE: How often the Lambda function should run automatically
### "rate(30 days)" means it runs once every 30 days

EMAIL=""
### EMAIL: Will store the email address where reports will be sent
### Empty now because user must provide it via command line

AWS_PROFILE=""  ### AWS profile to use
### AWS_PROFILE: Which AWS credentials profile to use (from ~/.aws/credentials)
### Empty means it will use the default profile unless user specifies otherwise

### Parse command line arguments
### This section reads the parameters you pass when running the script
while [[ $### -gt 0 ]]; do
### Start a loop that continues while there are arguments left to process
### $### means "number of arguments remaining"
### -gt means "greater than"
### So this says: "while number of arguments is greater than 0"

  case $1 in
  ### case statement checks the first argument ($1) against different patterns
  ### It's like a switch statement in other languages
  
    --stack-name)
    ### If the argument is "--stack-name"
      STACK_NAME="$2"
      ### Set STACK_NAME variable to the next argument ($2)
      ### Example: --stack-name my-stack → STACK_NAME becomes "my-stack"
      shift 2
      ### "shift 2" removes the first 2 arguments from the list
      ### This moves past both "--stack-name" and its value
      ;;
      ### ;; marks the end of this case option
      
    --region)
    ### If the argument is "--region"
      REGION="$2"
      ### Set REGION to the next argument
      ### Example: --region us-west-2 → REGION becomes "us-west-2"
      shift 2
      ### Remove both "--region" and its value from arguments
      ;;
      
    --schedule)
    ### If the argument is "--schedule"
      SCHEDULE="$2"
      ### Set SCHEDULE to the next argument
      ### Example: --schedule "rate(7 days)" → runs weekly
      shift 2
      ;;
      
    --email)
    ### If the argument is "--email"
      EMAIL="$2"
      ### Set EMAIL to the next argument
      ### Example: --email user@example.com
      shift 2
      ;;
      
    --profile)
    ### If the argument is "--profile"
      AWS_PROFILE="$2"
      ### Set AWS_PROFILE to the next argument
      ### Example: --profile production → uses "production" AWS credentials
      shift 2
      ;;
      
    *)
    ### * is a wildcard - matches anything that doesn't match above patterns
    ### This catches invalid/unknown arguments
      echo "Unknown option: $1"
      ### Print error message showing the unknown option
      exit 1
      ### Exit the script with error code 1 (non-zero means failure)
      ;;
  esac
  ### "esac" is "case" spelled backwards - ends the case statement
done
### End of the while loop

### Check if email is provided
if [ -z "$EMAIL" ]; then
### [ -z "$EMAIL" ] checks if EMAIL variable is empty (zero length)
### The if statement runs the following code block if condition is true

  echo "Error: Email is required. Please provide it with --email parameter."
  ### Print error message to the terminal
  exit 1
  ### Exit script with error code 1 (failure)
fi
### End of if statement

### Set AWS profile if specified
AWS_CMD_PROFILE=""
### Initialize empty variable to hold profile parameter for AWS CLI commands

if [ -n "$AWS_PROFILE" ]; then
### [ -n "$AWS_PROFILE" ] checks if AWS_PROFILE is NOT empty (has content)

  AWS_CMD_PROFILE="--profile $AWS_PROFILE"
  ### Build the profile parameter string that will be added to AWS commands
  ### Example: if AWS_PROFILE="dev", this becomes "--profile dev"
  
  echo "Using AWS profile: $AWS_PROFILE"
  ### Print confirmation message showing which profile is being used
fi

### Verify AWS credentials are valid
echo "Verifying AWS credentials..."
### Print status message to let user know what's happening

if ! aws sts get-caller-identity $AWS_CMD_PROFILE --region "$REGION" &>/dev/null; then
### This is a complex line - let's break it down:
### - aws sts get-caller-identity: AWS command that returns info about current user
### - $AWS_CMD_PROFILE: Inserts the profile parameter (if any)
### - --region "$REGION": Specifies which AWS region to use
### - &>/dev/null: Redirects all output (both stdout and stderr) to /dev/null (trash)
###   This makes the command silent - we only care if it succeeds or fails
### - ! at the start: Negates the result - true becomes false, false becomes true
### So this says: "if the AWS identity check FAILS..."

  echo "Error: Unable to validate AWS credentials. Check your AWS configuration or profile."
  ### Print error message if credentials are invalid
  exit 1
  ### Exit with failure code
fi

echo "AWS credentials verified."
### If we reach here, credentials are valid - print success message

### Prepare deployment files
echo "Preparing deployment files..."
### Print status message

rm -rf deployment
### Remove the "deployment" directory and all its contents if it exists
### -r means recursive (delete directory and everything inside)
### -f means force (don't ask for confirmation, don't error if it doesn't exist)

mkdir -p deployment
### Create a new "deployment" directory
### -p means "create parent directories if needed" and "don't error if it exists"

### Copy all Lambda files from src to deployment
cp -r src/lambda/* deployment/
### Copy all files from src/lambda/ directory to deployment/ directory
### -r means recursive (copy directories and their contents)
### The * wildcard means "everything inside src/lambda/"

### Package Lambda function
echo "Creating Lambda deployment package..."
### Print status message

cd deployment
### Change directory to "deployment" folder
### All subsequent commands will run from inside this directory

zip -r ../lambda_function.zip .
### Create a zip file containing all files in current directory
### -r means recursive (include subdirectories)
### ../lambda_function.zip means save the zip file one level up (parent directory)
### . means "current directory" (everything in deployment/)

cd ..
### Go back up one directory level to where we started

### Create/Update the CloudFormation stack
echo "Deploying CloudFormation stack '$STACK_NAME' to region '$REGION'..."
### Print status message showing which stack and region

aws cloudformation deploy \
### AWS CLI command to deploy a CloudFormation stack
### The \ at the end means "command continues on next line"

  --template-file templates/access-review-real.yaml \
  ### Path to the CloudFormation template file (YAML format)
  ### This file defines all the AWS resources to create
  
  --stack-name "$STACK_NAME" \
  ### Name for the CloudFormation stack (uses the variable we set earlier)
  
  --parameter-overrides \
  ### Indicates we're providing parameters to override template defaults
  
    RecipientEmail="$EMAIL" \
    ### Pass the EMAIL variable as the RecipientEmail parameter to CloudFormation
    
    ScheduleExpression="$SCHEDULE" \
    ### Pass the SCHEDULE variable as the ScheduleExpression parameter
    
  --capabilities CAPABILITY_IAM \
  ### Acknowledge that CloudFormation is allowed to create IAM resources
  ### Required when template creates IAM roles, policies, or users
  
  --region "$REGION" \
  ### Specify which AWS region to deploy to
  
  $AWS_CMD_PROFILE
  ### Add the profile parameter if one was specified (otherwise this is empty)

### Get the bucket name from the stack outputs
echo "Getting S3 bucket name from CloudFormation stack..."
### Print status message

BUCKET_NAME=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" --region "$REGION" $AWS_CMD_PROFILE --query "Stacks[0].Outputs[?OutputKey=='AccessReviewS3Bucket'].OutputValue" --output text)
### This complex line retrieves information from the CloudFormation stack
### Let's break it down:
### - $(...) means "run this command and store its output in the variable"
### - aws cloudformation describe-stacks: Get information about CloudFormation stacks
### - --stack-name "$STACK_NAME": Which stack to query
### - --region "$REGION": Which region the stack is in
### - $AWS_CMD_PROFILE: Add profile parameter if specified
### - --query "...": JMESPath query to extract specific data:
###   - Stacks[0]: Get the first (and only) stack from results
###   - Outputs[?OutputKey=='AccessReviewS3Bucket']: Find the output named 'AccessReviewS3Bucket'
###   - .OutputValue: Get the value of that output
### - --output text: Return the result as plain text (not JSON)
### Result: BUCKET_NAME variable now contains the S3 bucket name

echo "Getting Lambda function ARN from CloudFormation stack..."
### Print status message

LAMBDA_ARN=$(aws cloudformation describe-stacks --stack-name "$STACK_NAME" --region "$REGION" $AWS_CMD_PROFILE --query "Stacks[0].Outputs[?OutputKey=='AccessReviewLambdaArn'].OutputValue" --output text)
### Similar to above, but retrieves the Lambda function ARN (Amazon Resource Name)
### ARN is a unique identifier for AWS resources
### Format: arn:aws:lambda:region:account-id:function:function-name

### Update Lambda function code
echo "Updating Lambda function code..."
### Print status message

aws lambda update-function-code \
### AWS CLI command to update Lambda function's code
  --function-name "$LAMBDA_ARN" \
  ### Specify which Lambda function to update (using the ARN we just retrieved)
  
  --zip-file fileb://lambda_function.zip \
  ### Upload the zip file we created earlier
  ### fileb:// means "read this as binary file"
  
  --region "$REGION" \
  ### Specify the region
  
  $AWS_CMD_PROFILE
  ### Add profile parameter if specified

echo "Deployment completed successfully!"
### Print success message

echo "Lambda function: $LAMBDA_ARN"
### Display the Lambda function's ARN

echo "S3 bucket for reports: $BUCKET_NAME"
### Display the S3 bucket name where reports will be stored

echo "Recipient email: $EMAIL"
### Display the email address that will receive reports

echo "Schedule: $SCHEDULE"
### Display how often the Lambda will run

echo ""
### Print a blank line for spacing

echo "IMPORTANT: If this is a first-time deployment, you will need to verify your email address."
### Reminder message about email verification

echo "Check your inbox for a verification email from AWS SES and click the verification link."
### AWS SES (Simple Email Service) requires email verification before sending

echo ""
### Another blank line

echo "You can run a report immediately with: ./scripts/run_report.sh --stack-name $STACK_NAME --region $REGION $([[ -n \"$AWS_PROFILE\" ]] && echo \"--profile $AWS_PROFILE\")"
### Print instructions on how to run a report manually
### The complex part at the end: $([[ -n \"$AWS_PROFILE\" ]] && echo \"--profile $AWS_PROFILE\")
### - [[ -n \"$AWS_PROFILE\" ]]: Check if AWS_PROFILE is not empty
### - && echo \"--profile $AWS_PROFILE\": If true, output the profile parameter
### This conditionally adds the profile parameter only if one was specified
