# aws-project

# AWS Image Resizer CLI Implementation Guide

This guide provides step-by-step AWS CLI commands to implement the image resizing service.

## Step 1: Configure AWS CLI

First, ensure your AWS CLI is configured with your credentials

## Step 2: Create S3 Buckets

1. **Create Source Bucket**
```bash
aws s3api create-bucket \
  --bucket my-image-resizer-source \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1
```

2. **Create Destination Bucket**
```bash
aws s3api create-bucket \
  --bucket my-image-resizer-resized \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1
```

3. **Enable Versioning on Both Buckets**
```bash
aws s3api put-bucket-versioning \
  --bucket my-image-resizer-source \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-versioning \
  --bucket my-image-resizer-resized \
  --versioning-configuration Status=Enabled
```

## Step 3: Create IAM Role and Policy

1. **Create IAM Policy**
```bash
aws iam create-policy \
  --policy-name image-resizer-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:s3:::my-image-resizer-source/*",
                "arn:aws:s3:::my-image-resizer-resized/*",
                "arn:aws:logs:*:*:*"
            ]
        }
    ]
}'
```

2. **Create IAM Role**
```bash
aws iam create-role \
  --role-name image-resizer-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}'
```

3. **Attach Policy to Role**
```bash
aws iam attach-role-policy \
  --role-name image-resizer-lambda-role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/image-resizer-policy
```

## Step 4: Create Lambda Function

1. **Create Project Directory and Files**
```bash
mkdir image-resizer
cd image-resizer

# Create package.json
echo '{
  "name": "image-resizer",
  "version": "1.0.0",
  "description": "AWS Lambda image resizer",
  "main": "index.js",
  "dependencies": {
    "aws-sdk": "^2.1.0",
    "jimp": "^0.16.0"
  }
}' > package.json

# Install dependencies
npm install

# Create index.js
cat > index.js << 'EOL'
const AWS = require('aws-sdk');
const Jimp = require('jimp');

const s3 = new AWS.S3();

// Configuration
const DEST_BUCKET = process.env.DEST_BUCKET;
const MAX_WIDTH = 800;
const MAX_HEIGHT = 800;
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif'];

exports.handler = async (event) => {
    try {
        const record = event.Records[0];
        const sourceBucket = record.s3.bucket.name;
        const objectKey = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));

        console.log(`Processing image: ${objectKey} from bucket: ${sourceBucket}`);

        const params = {
            Bucket: sourceBucket,
            Key: objectKey
        };

        const inputData = await s3.getObject(params).promise();

        if (!ALLOWED_TYPES.includes(inputData.ContentType)) {
            throw new Error(`Unsupported file type: ${inputData.ContentType}`);
        }

        const image = await Jimp.read(inputData.Body);
        const originalWidth = image.getWidth();
        const originalHeight = image.getHeight();

        await image.scaleToFit(MAX_WIDTH, MAX_HEIGHT);

        const newWidth = image.getWidth();
        const newHeight = image.getHeight();

        const resizedImageBuffer = await image.getBufferAsync(Jimp.MIME_JPEG);

        const fileExtension = objectKey.split('.').pop();
        const fileNameWithoutExt = objectKey.substring(0, objectKey.lastIndexOf('.'));
        const newKey = `resized/${fileNameWithoutExt}_${newWidth}x${newHeight}.${fileExtension}`;

        const destParams = {
            Bucket: DEST_BUCKET,
            Key: newKey,
            Body: resizedImageBuffer,
            ContentType: 'image/jpeg'
        };

        await s3.putObject(destParams).promise();

        return {
            statusCode: 200,
            headers: {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            body: JSON.stringify({
                message: "Image processed successfully",
                source: objectKey,
                destination: newKey,
                originalDimensions: `${originalWidth}x${originalHeight}`,
                newDimensions: `${newWidth}x${newHeight}`
            })
        };

    } catch (error) {
        console.error('Error processing image:', error);
        return {
            statusCode: 400,
            headers: {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            body: JSON.stringify({
                message: "Error processing image",
                error: error.message,
                details: "Only image/jpeg, image/png, and image/gif are supported"
            })
        };
    }
};
EOL
```

2. **Create Deployment Package**
```bash
zip -r function.zip .
```

3. **Create Lambda Function**
```bash
aws lambda create-function \
  --function-name image-resizer-function \
  --runtime nodejs18.x \
  --handler index.handler \
  --role arn:aws:iam::YOUR_ACCOUNT_ID:role/image-resizer-lambda-role \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables="{DEST_BUCKET=my-image-resizer-resized}"
```

## Step 5: Configure S3 Event Trigger

1. **Create Notification Configuration**
```bash
cat > notification-config.json << 'EOL'
{
    "LambdaFunctionConfigurations": [
        {
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:YOUR_ACCOUNT_ID:function:image-resizer-function",
            "Events": ["s3:ObjectCreated:*"]
        }
    ]
}
EOL
```

2. **Apply Configuration**
```bash
aws s3api put-bucket-notification-configuration \
  --bucket my-image-resizer-source \
  --notification-configuration file://notification-config.json
```

3. **Add S3 Permission to Lambda**
```bash
aws lambda add-permission \
  --function-name image-resizer-function \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::my-image-resizer-source \
  --source-account YOUR_ACCOUNT_ID
```

## Step 6: Test the Implementation

1. **Upload Test Image**
```bash
# Create a test image (if you don't have one)
convert -size 1600x1200 xc:white test-image.jpg

# Upload to source bucket
aws s3 cp test-image.jpg s3://my-image-resizer-source/
```

2. **Monitor Lambda Execution**
```bash
# Get the latest log stream
aws logs describe-log-streams \
  --log-group-name /aws/lambda/image-resizer-function \
  --order-by LastEventTime \
  --descending \
  --limit 1

# Get log events
aws logs get-log-events \
  --log-group-name /aws/lambda/image-resizer-function \
  --log-stream-name YOUR_LOG_STREAM_NAME
```

3. **Check Resized Image**
```bash
# List objects in destination bucket
aws s3 ls s3://my-image-resizer-resized/resized/

# Download resized image
aws s3 cp s3://my-image-resizer-resized/resized/test-image_800x600.jpg ./resized-image.jpg
```

## Step 7: Set Up Monitoring

1. **Create CloudWatch Alarm**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ImageResizerErrors \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:ImageResizerAlerts
```

## Step 8: Clean Up (Optional)

If you need to remove the resources:
```bash
# Delete Lambda function
aws lambda delete-function --function-name image-resizer-function

# Delete S3 buckets
aws s3 rb s3://my-image-resizer-source --force
aws s3 rb s3://my-image-resizer-resized --force

# Delete IAM role and policy
aws iam detach-role-policy \
  --role-name image-resizer-lambda-role \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/image-resizer-policy

aws iam delete-role --role-name image-resizer-lambda-role
aws iam delete-policy --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/image-resizer-policy
```

## Important Notes:

1. Replace `YOUR_ACCOUNT_ID` with your actual AWS account ID in all commands
2. Replace `us-east-1` with your desired region if different
