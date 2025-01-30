The `multipart/related` header is a **MIME (Multipurpose Internet Mail Extensions) content type** used in HTTP when sending multiple related parts in a single request, typically in **HTTP POST** or **PUT** requests. It ensures that different parts of a request (such as a JSON body and an image) are related and should be processed together.

### **Key Characteristics:**

1. **Used for Bundling Related Content**
   - Common when sending data that has dependencies, like a JSON object describing an image along with the actual image file.
2. **Each Part Has Its Own Content-Type**

   - Every part of the request can have its own `Content-Type` (e.g., `application/json`, `image/png`, etc.).

3. **Boundary Parameter Required**
   - The `Content-Type: multipart/related` header must include a `boundary` parameter to separate different parts of the request.

### **Example Usage:**

#### **Scenario:** Sending a JSON metadata file and an image in a single HTTP request.

```http
POST /upload HTTP/1.1
Host: example.com
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

(binary data of the image)
--example-boundary--
```

### **Where is `multipart/related` Used?**

- **APIs that require metadata with media uploads** (e.g., Google Drive API).
- **Email attachments in SMTP** (when HTML emails include embedded images).
- **SOAP with attachments** (for XML-based services that send files).

Let me know if you need me to break it down further! 🚀
