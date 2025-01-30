Yes, you **can send an HTTP request to AWS API Gateway using `multipart/form-data`**, but there are a few considerations depending on how your API Gateway is set up.

---

## **1️⃣ How `multipart/form-data` Works**

- **Used for file uploads**, where each part has its own `Content-Type`.
- Requires a `boundary` to separate parts.
- Each field (text or file) is sent as a separate part.

### **Example HTTP Request (`multipart/form-data`)**

```http
POST /upload HTTP/1.1
Host: your-api-id.execute-api.region.amazonaws.com
Content-Type: multipart/form-data; boundary=boundary123

--boundary123
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{ "user": "john_doe", "description": "Profile picture" }
--boundary123
Content-Disposition: form-data; name="file"; filename="image.png"
Content-Type: image/png

(binary image data)
--boundary123--
```

---

## **2️⃣ API Gateway Considerations**

### **🔹 Direct Integration with AWS Lambda**

- **API Gateway does not natively parse `multipart/form-data`**.
- If you're integrating with **Lambda**, the entire request is received as **base64-encoded binary data**.
- You must enable **binary support** in API Gateway.
- The Lambda function will need to parse the `multipart/form-data` request manually.

### **🔹 Enabling Binary Support**

If you’re sending **binary data (like an image)**, you need to configure API Gateway:

1. **Add `multipart/form-data` to the Binary Media Types**:

   - Go to **API Gateway → Your API → Settings**.
   - Under **Binary Media Types**, add:
     ```
     multipart/form-data
     image/png (or whatever type you expect)
     ```
   - Deploy the API.

2. **Enable Base64 Decoding in Lambda**:
   - AWS API Gateway sends binary data as a **base64 string** to Lambda.
   - Your Lambda function must decode it.

---

## **3️⃣ Parsing `multipart/form-data` in AWS Lambda**

You can use `busboy` or `multiparty` to parse `multipart/form-data` in a **Node.js Lambda**.

### **Example (Node.js + Lambda)**

```typescript
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import * as Busboy from 'busboy';

export const handler = async (
  event: APIGatewayProxyEvent
): Promise<APIGatewayProxyResult> => {
  const contentType =
    event.headers['Content-Type'] || event.headers['content-type'];

  if (!contentType?.startsWith('multipart/form-data')) {
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
      if (fieldname === 'metadata') metadata = value;
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

---

## **4️⃣ Alternative: Use an S3 Presigned URL**

Instead of handling `multipart/form-data` in Lambda, you can:

1. Generate an **S3 Presigned URL** from Lambda.
2. Let the frontend **upload the file directly to S3**.
3. Store metadata separately in DynamoDB or another database.

---

## **Final Thoughts**

✅ **Yes, API Gateway supports `multipart/form-data`** but requires manual parsing in Lambda.  
✅ **You need to enable binary media types** if uploading files.  
✅ **Consider using S3 Presigned URLs** to avoid handling file uploads in Lambda.

Would you like an example for using S3 Presigned URLs? 🚀
