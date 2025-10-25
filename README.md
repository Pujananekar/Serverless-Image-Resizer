# 🖼️ Serverless Image Resizer

An automated, serverless image processing pipeline built with AWS Lambda and S3 that automatically resizes uploaded images into multiple sizes (thumbnail, medium, and large).

![AWS](https://img.shields.io/badge/AWS-Lambda-orange)
![Python](https://img.shields.io/badge/Python-3.12-blue)
![S3](https://img.shields.io/badge/S3-Storage-green)

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Setup Instructions](#setup-instructions)
- [How It Works](#how-it-works)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)


## 🎯 Overview

This project demonstrates a serverless, event-driven architecture for automatic image processing. When users upload images to an S3 bucket, AWS Lambda automatically triggers and creates three optimized versions: thumbnail (150x150), medium (500x500), and large (1000x1000), which are then stored in a separate S3 bucket.

**Use Cases:**
- E-commerce product image optimization
- Social media platforms
- Real estate listing websites
- Content management systems
- Photo sharing applications

## ✨ Features

- ✅ **Automatic Processing**: Images are processed instantly upon upload
- ✅ **Multiple Sizes**: Generates 3 versions (thumbnail, medium, large)
- ✅ **Serverless Architecture**: No server management required
- ✅ **Cost-Effective**: Pay only for execution time
- ✅ **Scalable**: Handles 1 to 10,000+ images seamlessly
- ✅ **Format Preservation**: Maintains original image format (JPEG, PNG, etc.)
- ✅ **Aspect Ratio**: Preserves image proportions during resizing
- ✅ **High Quality**: Uses LANCZOS resampling for best quality

### Architecture Diagram
![Architecture](https://via.placeholder.com/800x400?text=Architecture+Diagram)


**Flow:**
1. User uploads image to `original-images` bucket
2. S3 triggers Lambda function via event notification
3. Lambda downloads original image
4. Lambda creates 3 resized versions using Pillow
5. Lambda uploads resized images to `resized-images` bucket
6. Process completes in ~300-500ms per image

## 🛠️ Technologies Used

| Technology | Purpose |
|------------|---------|
| **AWS Lambda** | Serverless compute for image processing |
| **Amazon S3** | Object storage for original and resized images |
| **Python 3.12** | Lambda runtime environment |
| **Pillow (PIL)** | Image processing library |
| **AWS IAM** | Permissions and access management |
| **CloudWatch** | Logging and monitoring |
| **Boto3** | AWS SDK for Python |



## 🚀 Setup Instructions

### Step 1: Create S3 Buckets

1. Go to AWS S3 Console
2. Create two buckets:
   - `my-original-images-[yourname]` (for uploads)

### S3 Buckets
![S3 Buckets](https://via.placeholder.com/800x400?text=S3+Buckets)

   - `my-resized-images-[yourname]` (for processed images)
3. Choose the same region for both (e.g., `us-east-1`)

### Step 2: Create IAM Role

1. Go to IAM Console → Roles → Create role
2. Trusted entity: **AWS service** → **Lambda**
3. Attach permissions:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonS3FullAccess`
4. Name: `LambdaImageResizerRole`

### Step 3: Create Lambda Function

1. Go to Lambda Console → Create function
2. Configuration:
   - Name: `ImageResizer`
   - Runtime: `Python 3.12`
   - Role: Use existing → `LambdaImageResizerRole`
3. Click "Create function"

### Lambda Function
![Lambda](https://via.placeholder.com/800x400?text=Lambda+Function)

### Step 4: Add Pillow Layer

1. In Lambda function, scroll to **Layers** section
2. Click "Add a layer"
3. Choose "Specify an ARN"
4. Enter ARN:
```
   arn:aws:lambda:us-east-1:770693421928:layer:Klayers-p312-pillow:1
```
   *(Change region code if not using us-east-1)*
5. Click "Add"

### Step 5: Add Lambda Code

Copy and paste the following code into `lambda_function.py`:
```python
import json
import boto3
import urllib.parse
from PIL import Image
import io

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    
    print("Lambda function started!")
    print(f"Event received: {json.dumps(event)}")
    
    # Get bucket and key from event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'])
    
    print(f"Processing: {key} from {bucket}")
    
    # Target bucket name - CHANGE THIS!
    target_bucket = 'my-resized-images-yourname'
    
    # Define image sizes
    sizes = {
        'thumbnail': (150, 150),
        'medium': (500, 500),
        'large': (1000, 1000)
    }
    
    try:
        # Download original image
        print("Downloading original image...")
        response = s3_client.get_object(Bucket=bucket, Key=key)
        image_content = response['Body'].read()
        original_image = Image.open(io.BytesIO(image_content))
        
        print(f"Image format: {original_image.format}, Size: {original_image.size}")
        
        # Process each size
        for size_name, dimensions in sizes.items():
            print(f"Creating {size_name}...")
            
            resized_image = original_image.copy()
            resized_image.thumbnail(dimensions, Image.Resampling.LANCZOS)
            
            buffer = io.BytesIO()
            img_format = original_image.format if original_image.format else 'JPEG'
            resized_image.save(buffer, format=img_format)
            buffer.seek(0)
            
            new_key = f"{size_name}/{key}"
            print(f"Uploading to: {target_bucket}/{new_key}")
            
            s3_client.put_object(
                Bucket=target_bucket,
                Key=new_key,
                Body=buffer,
                ContentType=response['ContentType']
            )
            
            print(f"✅ Successfully created: {size_name}")
        
        print("All sizes created successfully!")
        
        return {
            'statusCode': 200,
            'body': json.dumps('Success!')
        }
        
    except Exception as e:
        print(f"❌ ERROR: {str(e)}")
        import traceback
        print(traceback.format_exc())
        raise e
```

**Important:** Change `target_bucket` value to your actual resized bucket name!

### Step 6: Configure Lambda Settings

1. Go to **Configuration** → **General configuration** → Edit
2. Set:
   - Timeout: `1 min 0 sec`
   - Memory: `512 MB`
3. Click "Save"

### Step 7: Add S3 Trigger

1. In Lambda function, click "Add trigger"
2. Configuration:
   - Source: **S3**
   - Bucket: Select your original images bucket
   - Event type: **All object create events**
   - Acknowledge recursive invocation warning
3. Click "Add"

### Step 8: Configure Bucket Access (Optional)

To view images via browser URL:

1. Go to S3 → Resized bucket → Permissions
2. Edit "Block public access" → Uncheck all
3. Add bucket policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-resized-images-yourname/*"
        }
    ]
}
```

## 🧪 Testing

1. Upload a test image to the original bucket
2. Wait 10-30 seconds
3. Check resized bucket for 3 folders:
   - `thumbnail/` - 150x150 images
   - `medium/` - 500x500 images
   - `large/` - 1000x1000 images

   ### Resized Images
![Results](https://via.placeholder.com/800x400?text=Resized+Images)

**Test with different formats:**
- JPEG images ✓
- PNG images ✓
- Different sizes (100KB to 10MB) ✓


## 🐛 Troubleshooting

### Issue: Lambda not triggering

**Solution:**
- Check S3 event notification in bucket properties
- Verify Lambda has S3 trigger configured
- Check CloudWatch logs for errors

### Issue: "No module named PIL"

**Solution:**
- Verify Pillow layer is attached
- Check Python runtime version matches layer (3.12)
- Try different layer ARN for your region

### Issue: "Access Denied"

**Solution:**
- Add `AmazonS3FullAccess` policy to Lambda role
- Check bucket permissions
- Verify bucket names in code

### Issue: "Timeout"

**Solution:**
- Increase Lambda timeout to 1+ minutes
- Increase memory allocation
- Check image file size

### Issue: Images not appearing in browser

**Solution:**
- Make bucket public (see Step 8)
- Use presigned URLs instead
- Check bucket policy syntax

## 👤 Author

**Your Name**
- GitHub: [@yourusername](https://github.com/yourusername)
- LinkedIn: [Your Name](https://linkedin.com/in/yourprofile)
- Email: your.email@example.com

