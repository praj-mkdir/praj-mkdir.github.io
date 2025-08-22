---
title: "Handling File Upload Status with S3 Pre-Signed URLs"
date: 2025-08-15T11:30:03+00:00
# weight: 1
aliases: ["/first"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
---

While working on a project, I decided to use S3 pre-signed URLs for file uploads. The main reason was that I didn’t want my backend server to handle large files directly. If a 500MB file came in, routing it through the server would increase latency, consume bandwidth, and unnecessarily load the application. With pre-signed URLs, the backend just generates a temporary signed URL, gives it to the client, and the client uploads directly to S3.

This approach worked really well for a few reasons. It improves performance since the file doesn’t take two hops (client → backend → S3), it saves infrastructure costs because my server isn’t handling big file streams, and it scales better when multiple users upload files at the same time. Security is also built in because the URL has an expiry time and only allows the operation I specify (like PUT for upload).

But after trying it out, I ran into a problem. The backend had no way of knowing if the upload was completed or not. For example:

- If a client cancelled the upload halfway, I wouldn’t know.  
- If the network failed during upload, I wouldn’t know.  
- Even if the file was uploaded successfully, there was no automatic notification to the backend.  

For my use case, this was a problem because I needed to track the state of the file. I wanted to mark it as pending when the URL was generated, then mark it as uploaded only after it was actually present in S3. I also needed a reliable point to trigger post-upload actions like virus scanning, encryption checks, or updating the user’s storage usage.

To solve this, I looked into S3 event notifications. S3 supports sending events whenever an object is created, deleted, or modified. These events can be sent to different AWS services such as Lambda, SNS, or SQS. Out of these, I chose SQS because I wanted a queue-based approach that my backend could consume at its own pace.

The flow I ended up with looked like this:

1. The client requests a file upload. The backend generates a pre-signed URL and stores a record for the file with status set to “pending upload.”  
2. The client uses that URL to upload the file directly to S3.  
3. Once the upload is successful, S3 automatically sends an event notification.  
4. That event is pushed into an SQS queue.  
5. My backend has a consumer that listens to SQS, processes these events, and updates the file status to “uploaded.” At that point, I can trigger additional actions like malware scanning, auditing, or updating user quotas.  

This solved the problem for me. Instead of relying on the client to tell me if the upload was successful (which can’t be fully trusted), I now rely on S3 itself to confirm. Using SQS in the middle gave me a reliable way to decouple uploads from backend processing, and it also allowed me to handle events asynchronously.

In the end, using pre-signed URLs plus S3 event notifications turned out to be a solid pattern. It kept uploads efficient and scalable, while still giving my backend the visibility it needed into the final state of the files.
