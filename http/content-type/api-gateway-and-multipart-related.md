### **Can AWS API Gateway Handle `multipart/related` Requests?**

Yes, AWS **API Gateway** can receive `multipart/related` requests, but **it does not natively parse them**. Similar to `multipart/form-data`, API Gateway **forwards the raw request body** to AWS Lambda, meaning you'll need to **manually parse** the `multipart/related` content inside your Lambda function.

---

## **1️⃣ How `multipart/related` Works**

- Used when **sending multiple related parts in a single request**.
- Common for **JSON metadata + binary files**.
- Requires a **boundary** to separate different parts.

### **Example `multipart/related` HTTP Request**

```http
POST /upload HTTP/1.1
Host: your-api-id.execute-api.region.amazonaws.com
Content-Type: multipart/related; boundary=example-boundary

--example-boundary
Content-Type: application/json

{
  "name": "profile-pic",
  "description": "User profile picture"
}
--example-boundary
Content-Type: image/png
Content-Transfer-Encoding: binary

(binary image data)
--example-boundary--
```

---

## **2️⃣ API Gateway Behavior**

- API Gateway **does not parse** `multipart/related`; it **forwards** the request as-is.
- If the request **includes binary data**, API Gateway **base64-encodes** it before sending it to Lambda.
- You **must enable binary media types** in API Gateway for proper handling.

### **How to Enable Binary Support in API Gateway**

1. **Go to API Gateway Console → Your API → Settings**.
2. **Under "Binary Media Types", add**:
   ```
   multipart/related
   image/png  (or any binary types you're expecting)
   ```
3. **Deploy the API**.

---

## **3️⃣ Parsing `multipart/related` in AWS Lambda**

Since API Gateway **does not parse `multipart/related`**, your Lambda function must handle it.

### **Example: Parsing `multipart/related` in a Node.js AWS Lambda**

You can use **`busboy`** or **manual parsing**.

```typescript
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import * as Busboy from 'busboy';

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const contentType =
    event.headers['Content-Type'] || event.headers['content-type'];

  if (!contentType?.startsWith('multipart/related')) {
    return { statusCode: 400, body: 'Invalid Content-Type' };
  }

  const bodyBuffer = Buffer.from(
    event.body ?? '',
    event.isBase64Encoded ? 'base64' : 'utf-8'
  );
  const busboy = new Busboy({ headers: { 'content-type': contentType } });

  return new Promise((resolve) => {
    let metadata = '';
    let fileBuffer: Buffer | null = null;

    busboy.on('field', (fieldname, value) => {
      metadata = value;
    });

    busboy.on('file', (_, file) => {
      const chunks: Buffer[] = [];
      file.on('data', (chunk) => chunks.push(chunk));
      file.on('end', () => (fileBuffer = Buffer.concat(chunks)));
    });

    busboy.on('finish', () => {
      resolve({
        statusCode: 200,
        body: JSON.stringify({ metadata, fileSize: fileBuffer?.length }),
      });
    });

    busboy.end(bodyBuffer);
  });
};
```

### **How This Works**

- The function **extracts JSON metadata** and the **binary file** separately.
- It **decodes the base64** content if API Gateway encoded it.
- It **returns the parsed metadata** and the **file size** as a response.

---

## **4️⃣ Alternative: Using S3 Presigned URLs**

Handling `multipart/related` inside Lambda is **complex and not ideal for large files**.  
A better approach is to:

1. Use API Gateway + Lambda **to generate an S3 Presigned URL**.
2. Upload the file **directly to S3** from the frontend.
3. Store the **metadata in a database** (DynamoDB, RDS, etc.).

### **Example Presigned URL Flow**

1. **Client Requests Upload URL**:

   ```http
   POST /generate-upload-url
   ```

   - API Gateway calls Lambda to generate a **Presigned S3 URL**.

2. **Lambda Generates URL**:

   ```typescript
   import AWS from 'aws-sdk';

   const s3 = new AWS.S3();

   export const handler = async () => {
     const params = {
       Bucket: 'my-bucket',
       Key: `uploads/profile-pic.png`,
       Expires: 60, // 1 min expiry
       ContentType: 'image/png',
     };
     const uploadUrl = s3.getSignedUrl('putObject', params);

     return { statusCode: 200, body: JSON.stringify({ uploadUrl }) };
   };
   ```

3. **Client Uploads File Directly to S3**:

   ```http
   PUT {uploadUrl}
   Content-Type: image/png

   (binary image data)
   ```

4. **Client Sends Metadata Separately**:

   ```http
   POST /save-metadata
   Content-Type: application/json

   {
     "user": "john_doe",
     "fileUrl": "https://s3.amazonaws.com/my-bucket/uploads/profile-pic.png",
     "description": "Profile picture"
   }
   ```

✅ **Why This is Better**:

- **No need for API Gateway to handle large files.**
- **Faster uploads (direct to S3, no Lambda bottleneck).**
- **Metadata and file are stored separately for flexibility.**

---

## **Final Thoughts**

✔ **Yes, API Gateway supports `multipart/related`**, but **you must manually parse it in Lambda**.  
✔ **Enable binary media types** in API Gateway if sending files.  
✔ **Handling `multipart/related` in Lambda is complex**—consider using **S3 Presigned URLs** instead.

Let me know if you need more details! 🚀
