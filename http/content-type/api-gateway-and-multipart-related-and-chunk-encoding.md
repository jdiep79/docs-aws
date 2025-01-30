### **Can AWS API Gateway Handle `multipart/related` with Chunked Encoding?**

🚨 **No, AWS API Gateway does not natively support `Transfer-Encoding: chunked` for incoming requests.** 🚨

If you try to send a **`multipart/related` request with `Transfer-Encoding: chunked`**, API Gateway **will reject it** because it expects a `Content-Length` header for request bodies. API Gateway does **not** process chunked encoding on incoming requests.

---

## **1️⃣ What Happens If You Send `multipart/related` with `Transfer-Encoding: chunked`?**

When you send a request like this:

```http
POST /upload HTTP/1.1
Host: your-api-id.execute-api.region.amazonaws.com
Content-Type: multipart/related; boundary=example-boundary
Transfer-Encoding: chunked

4
Wiki
6
pedia
0
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

### **API Gateway Response**

- API Gateway **rejects the request** with a `400 Bad Request` error.
- It expects a `Content-Length` header, which chunked encoding does **not** provide.
- There is **no way to enable chunked encoding for incoming requests** in API Gateway.

---

## **2️⃣ Workarounds for Sending `multipart/related` to API Gateway**

Since API Gateway does **not** support `Transfer-Encoding: chunked`, you have two options:

### **🔹 Option 1: Send a Full `multipart/related` Request with `Content-Length`**

✅ Works, but **requires knowing the total body size** before sending the request.

Instead of:

```http
Transfer-Encoding: chunked
```

Send:

```http
Content-Length: 3876
```

- This means you must calculate the total size of the `multipart/related` body before sending the request.

---

### **🔹 Option 2: Use S3 Presigned URLs Instead**

Since chunked encoding is often used for **large uploads**, a better solution is to:

1. **Generate an S3 Presigned URL** using API Gateway + Lambda.
2. **Upload the file directly to S3**.
3. **Send metadata separately**.

#### **How This Works**

1️⃣ **Client requests an upload URL**:

```http
POST /generate-upload-url
```

- API Gateway calls Lambda to generate a **Presigned S3 URL**.

2️⃣ **Lambda generates a Presigned URL**:

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

3️⃣ **Client uploads the file directly to S3 using the Presigned URL**:

```http
PUT {uploadUrl}
Content-Type: image/png

(binary image data)
```

- **No API Gateway involved in the upload**—S3 can handle large files **without chunking issues**.

4️⃣ **Client sends metadata separately**:

```http
POST /save-metadata
Content-Type: application/json

{
  "user": "john_doe",
  "fileUrl": "https://s3.amazonaws.com/my-bucket/uploads/profile-pic.png",
  "description": "Profile picture"
}
```

✅ **Why This Works Best**

- **No need to calculate `Content-Length` manually.**
- **No need for `multipart/related` parsing inside Lambda.**
- **Faster uploads, no API Gateway size limits.**

---

## **Final Answer**

🚨 **API Gateway does NOT support `Transfer-Encoding: chunked` on incoming requests.**  
✅ **You must send `Content-Length` instead** if using `multipart/related`.  
🔥 **Best practice:** Use **S3 Presigned URLs** for large file uploads.

Would you like an example for sending `multipart/related` without chunked encoding? 🚀
