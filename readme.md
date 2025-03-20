# Image Resizer Implementation Guide

This guide provides specific implementation details with actual values and configurations for the AWS Lambda image resizing service.

## Project Configuration

### 1. S3 Bucket Names
```bash
SOURCE_BUCKET="my-image-resizer-source"
DESTINATION_BUCKET="my-image-resizer-resized"
REGION="us-east-1"
```

### 2. IAM Role and Policy

**Policy Name**: `image-resizer-policy`
```json
{
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
}
```

**Role Name**: `image-resizer-lambda-role`

### 3. Lambda Function Configuration

**Function Name**: `image-resizer-function`
- Runtime: Node.js 18.x
- Memory: 256 MB
- Timeout: 30 seconds
- Handler: index.handler

### 4. Project Structure
```
image-resizer/
├── index.js
├── package.json
├── node_modules/
└── function.zip
```

### 5. Implementation Steps

1. **Create Project Directory and Initialize**
```bash
mkdir image-resizer
cd image-resizer
npm init -y
```

2. **Install Dependencies**
```bash
npm install aws-sdk jimp
```

3. **Create Lambda Function Code**
Create `index.js`:
```javascript
const AWS = require('aws-sdk');
const Jimp = require('jimp');

const s3 = new AWS.S3();

// Configuration
const DEST_BUCKET = process.env.DEST_BUCKET;
const MAX_WIDTH = 800;  // Increased size for better quality
const MAX_HEIGHT = 800;
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif'];

exports.handler = async (event) => {
    try {
        // Get the source bucket and object key from the event
        const record = event.Records[0];
        const sourceBucket = record.s3.bucket.name;
        const objectKey = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));

        console.log(`Processing image: ${objectKey} from bucket: ${sourceBucket}`);

        // Get the image from S3
        const params = {
            Bucket: sourceBucket,
            Key: objectKey
        };

        const inputData = await s3.getObject(params).promise();

        // Validate content type
        if (!ALLOWED_TYPES.includes(inputData.ContentType)) {
            throw new Error(`Unsupported file type: ${inputData.ContentType}`);
        }

        // Process the image
        const image = await Jimp.read(inputData.Body);
        
        // Get original dimensions
        const originalWidth = image.getWidth();
        const originalHeight = image.getHeight();

        // Resize image maintaining aspect ratio
        await image.scaleToFit(MAX_WIDTH, MAX_HEIGHT);

        // Get new dimensions
        const newWidth = image.getWidth();
        const newHeight = image.getHeight();

        // Convert to buffer
        const resizedImageBuffer = await image.getBufferAsync(Jimp.MIME_JPEG);

        // Generate new filename with dimensions
        const fileExtension = objectKey.split('.').pop();
        const fileNameWithoutExt = objectKey.substring(0, objectKey.lastIndexOf('.'));
        const newKey = `resized/${fileNameWithoutExt}_${newWidth}x${newHeight}.${fileExtension}`;

        // Upload resized image
        const destParams = {
            Bucket: DEST_BUCKET,
            Key: newKey,
            Body: resizedImageBuffer,
            ContentType: 'image/jpeg'
        };

        await s3.putObject(destParams).promise();

        console.log(`Successfully resized and uploaded ${objectKey} to ${DEST_BUCKET}`);
        
        return {
            statusCode: 200,
            body: JSON.stringify({
                message: 'Image processed successfully',
                source: objectKey,
                destination: newKey,
                originalDimensions: `${originalWidth}x${originalHeight}`,
                newDimensions: `${newWidth}x${newHeight}`
            })
        };

    } catch (error) {
        console.error('Error processing image:', error);
        throw error;
    }
};
```

4. **Create Deployment Package**
```bash
zip -r function.zip .
```

5. **Create Lambda Function**
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

6. **Configure S3 Event Notification**
Create `notification-config.json`:
```json
{
    "LambdaFunctionConfigurations": [
        {
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:YOUR_ACCOUNT_ID:function:image-resizer-function",
            "Events": ["s3:ObjectCreated:*"]
        }
    ]
}
```

7. **Apply S3 Configuration**
```bash
aws s3api put-bucket-notification-configuration \
  --bucket my-image-resizer-source \
  --notification-configuration file://notification-config.json
```

### 6. Testing the Implementation

1. **Upload Test Image**
```bash
aws s3 cp test-image.jpg s3://my-image-resizer-source/
```

2. **Monitor Execution**
- Check CloudWatch Logs in the AWS Console
- Look for log group: `/aws/lambda/image-resizer-function`
- Verify the resized image in the destination bucket

### 7. Expected Results

When you upload an image to the source bucket:
1. The Lambda function will be triggered
2. The image will be resized to fit within 800x800 pixels while maintaining aspect ratio
3. The resized image will be saved in the destination bucket with the format:
   `resized/original-name_newWidthxnewHeight.jpg`

### 8. Cost Estimation

**Lambda Costs**:
- Memory: 256 MB
- Average execution time: ~1-2 seconds
- Estimated cost per 1000 images: $0.00001667 per GB-second

**S3 Costs**:
- Storage: Standard storage class
- Estimated cost per GB: $0.023 per month
- Data transfer: $0.09 per GB

### 9. Monitoring Setup

1. **CloudWatch Alarms**
```json
{
    "AlarmName": "ImageResizerErrors",
    "MetricName": "Errors",
    "Namespace": "AWS/Lambda",
    "Statistic": "Sum",
    "Period": 300,
    "EvaluationPeriods": 1,
    "Threshold": 1,
    "ComparisonOperator": "GreaterThanThreshold",
    "AlarmActions": ["arn:aws:sns:us-east-1:YOUR_ACCOUNT_ID:ImageResizerAlerts"]
}
```

2. **CloudWatch Dashboard**
- Lambda execution metrics
- S3 bucket metrics
- Error rates
- Processing times

### 10. Maintenance Tasks

1. **Weekly Tasks**
- Check CloudWatch logs for errors
- Monitor Lambda execution times
- Review S3 storage usage

2. **Monthly Tasks**
- Update dependencies
- Review IAM permissions
- Analyze cost reports
- Clean up old logs

### 11. Troubleshooting Steps

1. **Common Issues**
- Lambda timeout: Increase timeout to 60 seconds
- Memory issues: Increase memory to 512 MB
- Permission errors: Review IAM role permissions
- Invalid file types: Check ALLOWED_TYPES array

2. **Log Analysis**
```bash
aws logs get-log-events \
  --log-group-name /aws/lambda/image-resizer-function \
  --log-stream-name $(aws logs describe-log-streams \
    --log-group-name /aws/lambda/image-resizer-function \
    --order-by LastEventTime \
    --descending \
    --limit 1 \
    --query 'logStreams[0].logStreamName' \
    --output text)
```
