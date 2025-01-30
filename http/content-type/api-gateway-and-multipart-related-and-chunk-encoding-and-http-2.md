### **Does AWS API Gateway Support `multipart/related` with HTTP/2?**

🚨 **No, AWS API Gateway does NOT support `multipart/related` with HTTP/2 multiplexed streaming.** 🚨

Even though HTTP/2 allows streaming **without** `Transfer-Encoding: chunked`, API Gateway **still requires `Content-Length`** for requests with a body. **This means you cannot stream a `multipart/related` request without knowing the total size beforehand.**

---

## **1️⃣ Understanding HTTP/2 and Streaming**

### **🔹 How HTTP/2 Streams Data**

- Unlike HTTP/1.1, HTTP/2 **does not use `Transfer-Encoding: chunked`** for streaming.
- Instead, **it splits data into multiple frames** (streams) and sends them asynchronously.
- HTTP/2 **can send `multipart/related`** but API Gateway still **expects `Content-Length`**.

### **🔹 Does API Gateway Support HTTP/2 Streaming?**

- **For requests:** ❌ **No**, API Gateway **does not support streaming HTTP/2 request bodies**.
- **For responses:** ✅ **Yes**, API Gateway can stream responses **if the backend supports it**.

---

## **2️⃣ What Happens If You Send `multipart/related` via HTTP/2?**

If you send a request like this:

```http
POST /upload HTTP/2
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

### **API Gateway Response**

- **If `Content-Length` is missing:** 🚨 `411 Length Required`
- **If streaming data instead of sending it all at once:** 🚨 `400 Bad Request`

This happens because API Gateway does **not** support streaming request bodies, even over HTTP/2.

---

## **3️⃣ Workarounds for Sending Large `multipart/related` Requests**

Since API Gateway **requires `Content-Length`**, here’s how to handle large uploads properly:

### **🔹 Option 1: Precompute `Content-Length` and Send `multipart/related` Normally**

✅ Works, but **you must calculate the request body size in advance.**

- **Before sending the request, measure the total size** (JSON + file + boundaries).
- **Set the `Content-Length` header**.
- **Send the full body in one go**.

### **🔹 Option 2: Use S3 Presigned URLs (Recommended)**

If you need to send large files:

1. **Request a Presigned S3 URL from API Gateway.**
2. **Upload the file directly to S3 (no API Gateway involved).**
3. **Send metadata separately to API Gateway.**

### **🔹 Option 3: Use AWS App Runner or ALB Instead**

- **App Runner** and **Application Load Balancer (ALB)** support **streaming HTTP/2 requests**.
- You can deploy a **custom service** that handles `multipart/related` streaming.

---

## **4️⃣ Final Answer**

❌ **AWS API Gateway does NOT support HTTP/2 request streaming** (even though HTTP/2 allows it).  
✅ **You must send `Content-Length` with `multipart/related`.**  
🔥 **Best practice:** Use **S3 Presigned URLs** for large file uploads.

Let me know if you need an example! 🚀

### **Yes! If you want to use HTTP/2 with `multipart/related`, you must:**

✅ **Send a `Content-Length` header** (you cannot stream the request).  
✅ **Avoid `Transfer-Encoding: chunked`**, since HTTP/2 does not use it.  
✅ **Send the entire `multipart/related` body in one go** (precomputed size).

---

## **1️⃣ Correct Way to Send `multipart/related` over HTTP/2**

Here’s an example of a **valid HTTP/2 request** with `multipart/related`:

```http
POST /upload HTTP/2
Host: your-api-id.execute-api.region.amazonaws.com
Content-Type: multipart/related; boundary=example-boundary
Content-Length: 3876

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

### **Key Rules**

✔ **Include `Content-Length`** (precompute the total size).  
✔ **No `Transfer-Encoding: chunked`** (HTTP/2 does not use it).  
✔ **Send the full request body in a single transmission**.

---

## **2️⃣ How to Calculate `Content-Length`**

Since API Gateway **requires `Content-Length`**, you need to calculate the total size of:

- **The JSON metadata part**
- **The binary file part**
- **The boundary strings (`--example-boundary`)**

### **Example in Node.js (Manually Constructing the Request)**

```typescript
import fs from 'fs';
import http2 from 'http2';

const client = http2.connect(
  'https://your-api-id.execute-api.region.amazonaws.com'
);

const jsonPart = JSON.stringify({
  name: 'profile-pic',
  description: 'User profile picture',
});

const imagePath = 'path/to/image.png';
const imageBuffer = fs.readFileSync(imagePath);
const boundary = 'example-boundary';

// Construct the multipart body
const body =
  `--${boundary}\r\nContent-Type: application/json\r\n\r\n${jsonPart}\r\n` +
  `--${boundary}\r\nContent-Type: image/png\r\nContent-Transfer-Encoding: binary\r\n\r\n` +
  imageBuffer +
  `\r\n--${boundary}--\r\n`;

const contentLength = Buffer.byteLength(body);

const req = client.request({
  ':method': 'POST',
  ':path': '/upload',
  'content-type': `multipart/related; boundary=${boundary}`,
  'content-length': contentLength,
});

req.write(body);
req.end();

req.on('response', (headers) => {
  console.log('Response Headers:', headers);
});
```

---

## **3️⃣ Why This Works**

✅ **HTTP/2 supports multiplexing**, but **API Gateway still requires `Content-Length`**.  
✅ **This ensures the request body is processed correctly.**  
✅ **No chunked encoding or streaming required.**

---

## **4️⃣ Alternative: Use S3 Presigned URLs for Large Files**

If you are sending **large files**, you **should NOT** use `multipart/related` with API Gateway.  
Instead:

1. **Use API Gateway to generate an S3 Presigned URL**.
2. **Upload the file directly to S3**.
3. **Send metadata separately via API Gateway**.

Would you like an example for this approach? 🚀
