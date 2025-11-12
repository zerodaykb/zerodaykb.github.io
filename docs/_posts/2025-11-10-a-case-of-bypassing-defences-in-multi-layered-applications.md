---
layout: single
title: "RCE in ImageTragick - a case of bypassing defences in microservices application."
categories: [security]
tags: [microservices, API, fuzzing]
author_profile: true
---

## TLDR

I was able to bypass file-type validation by using allowed parameter value from another part of the ecosystem and achieve RCE.

## Background

Microservices architecture is a common approach to building modern applications. It allows for code reusability and modularity. 

In one of my contracts I tested app consisting of two parts - B2B and B2C website. In both applications there were upload forms:
- one allowing for uploading images and pdfs
- another allowing for uploading companies logos

Both of uploads were hitting same microservice for converting files with Imagemagic.

---
### B2B part:

You could specify attachment_type parameter value as "image" or "pdf":

```
POST /upload_attachment HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryvbOSgVpRh3Ra6iZT

------WebKitFormBoundaryvbOSgVpRh3Ra6iZT
Content-Disposition: form-data; name="attachment_type"

image
------WebKitFormBoundaryvbOSgVpRh3Ra6iZT
Content-Disposition: form-data; name="attachment[0]"; filename="image.png"
Content-Type: image/png

PNG
[image content]

------WebKitFormBoundaryvbOSgVpRh3Ra6iZT--
```

```
POST /upload_attachment HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryvbOSgVpRh3Ra6iZT

------WebKitFormBoundaryvbOSgVpRh3Ra6iZT
Content-Disposition: form-data; name="attachment_type"

pdf
------WebKitFormBoundaryvbOSgVpRh3Ra6iZT
Content-Disposition: form-data; name="attachment[0]"; filename="file.pdf"
Content-Type: application/pdf

%PDF
[pdf content]

------WebKitFormBoundaryvbOSgVpRh3Ra6iZT--
```


Note that:
- When you tried to specify different parameter you received 500 error.
- Content-type and magic bytes were correctly validated for both types.

---
### B2C part:

There was only option to upload company logo:

```
POST /upload_logo HTTP/1.1
Host: b2b.target.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryvbOSgVpRh3Ra6iZT

------WebKitFormBoundaryvbOSgVpRh3Ra6iZT
Content-Disposition: form-data; name="attachment[0]"; filename="logo.png"
Content-Type: image/png

PNG
[image content]

------WebKitFormBoundaryvbOSgVpRh3Ra6iZT--
```

Content-type and magic bytes were correctly validated again.

## The bug

Both endpoint were hitting same microservice which code looked like this:

```
valid = {"image", "pdf", "logo"}

if action in valid:
    convert()
else:
    raise Exception("Invalid action")
```

This microservice didn't check for file type when passing the file to conversion with ImageMagick.


By using parametr `action` with value `logo` in B2C website I was able to send Postscript file to the microservice and achieve RCE with ImageTragick.

**So in short**: you were able to bypass all the validations and achieve RCE with exactly and only one specificparameter value.

This shows how it is important to know your target. You could also generate a proper wordlist for the ecosystem and fuzz parameters but which is easier approach?

## Takeaways
- find which functionalities might be hitting same microservice
- compare API request and responses from both angles
- fuzz parameters
