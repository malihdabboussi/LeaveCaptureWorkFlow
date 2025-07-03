# **Lead‑Capture & Enrichment Workflow**

**Purpose** – Automatically turn a *forwarded* email (from Gmail/Outlook) into a clean, enriched CRM lead that is stored in Supabase and linked to its company record.


# **Node Overview**

1 . **IMAP trigger** – watches a dedicated inbox.  
2 . **AI extractor** – gets lead fields from the forwarded email.  
3 . **Email verifier** – Hunter.io validity check.  
4 . **Email checker** – throws on invalid result.  
5 . **Lead upsert** – SQL above.  
6 . **Company enrichment** – TheCompaniesAPI call.  
7 . **Company upsert** – SQL above.  


# **AI Models**

In this Challenge, I used two AI models.
### **Mistral-Small Model**
**Used for:**
* **data extraction (first name ,last name , domain , email , company) ** 
* **check if the domain was private or public domain (google,outlook,..etc) if public domain empty string**
* **output as JSON format**


#### **Prompt**

The prompt consists of the output format and rules, and supported with example.

```
You will receive a JSON array in the variable  {{ $json.textPlain }}.
Each element has:
  • metadata      – an object holding e-mail headers  
  • textPlain     – the raw e-mail body (may contain a forwarded message)

Your task: extract the ORIGINAL sender’s details and output a **single JSON
object** with these keys (always include all keys, even if empty):

  first_name   last_name   email   company   domain

──────────────────────────
Extraction order & rules
──────────────────────────
1️⃣  Check {{ $json[0].textPlain }} for a line that starts with **“From:”**
    or **“From:”** inside a forwarded / original message block.
    Example pattern:  
      From: Michael Chen <m.chen88@gmail.com>

    • Parse the name before “< >”.  
      – Split on spaces: the first token → **first_name**, remaining → **last_name**.  
    • Parse the address inside “< >” → **email**.  
    • domain = part after “@” in the e-mail.

2️⃣  Decide if the domain is *public* (generic mailbox) or *private* (company):
      Public domains list ⟨gmail.com, yahoo.com, outlook.com, hotmail.com,
      icloud.com, aol.com, protonmail.com⟩.  
      If domain ∈ public-list → **domain = ""** (empty).

3️⃣  **Company extraction**  
      • If domain is private (not in list) → company = Pascal-cased domain
        without TLD (e.g. “tkminds.com” → “Tkminds”).  
      • If domain is public ("") → scan the e-mail body for a phrase of the
        form “at …” or bold/quoted company name, e.g.  
          “I’m the VP of Ops at **Acme Corporation**,”  
        Use that matched phrase (strip “at”, punctuation, and * or **).

4️⃣  If step 1 fails entirely, fall back to the envelope sender in  
      {{ $json[0].metadata['return-path'] }} and repeat steps 1-3.

──────────────────────────
Output rules
──────────────────────────
• **Output only the JSON object – no markdown, no back-ticks, no extra text.**
• Keys must appear in this order: first_name, last_name, email, company, domain.
• Values must be strings (use "" when not available).

Example (desired):

{
  "first_name": "Michael",
  "last_name": "Chen",
  "email": "m.chen88@gmail.com",
  "company": "Acme Corporation",
  "domain": ""
}
```

### **Output‑parser model (second Mistral‑small call)**

Re‑formats the main model’s response so it fits downstream nodes without any additional cleaning.

## **Flow Helpers**

### **Email Verifier**
Instead of a hard‑coded regex, the workflow calls hunter.io to check whether the address is valid and deliverable,
and to retrieve a confidence score.

### **Email Checker Code**

If Hunter’s status or score indicates an invalid address, the node throws the error “Email not valid.”

### **Postgres SQL Query**

The query inserts a new lead or does a no‑op update on conflict, then always returns the row:

```sql

INSERT INTO leads (first_name, last_name, email)
VALUES ($1, $2, $3)                          
ON CONFLICT (email)
DO UPDATE
   SET email = EXCLUDED.email               
RETURNING id, first_name, last_name, email; 
```

### **Email Parser**

Cleans the incoming data and produces a usable JSON object, e.g.


```json

[
    {
    "id": "54",
    "first_name": "Maria",
    "last_name": "Garcia",
    "email": "maria@summit-logistics.com",
    "domain": "summit-logistics.com",
    "company": "Summit Logistics"
    }
]
```



```javascript
return $input.all().map(item => {
  const email  = item.json.email || '';
  const domain = $('If').first().json.domain;
  
  return {
      ...item.json,
      domain,
      email,
      company:$('If').first().json.company
  };
});
```


### **Company Enrichment By Name**

A GET request to thecompaniesapi.com (similar to Clearbit) using the company name from the previous step.

### **Data Construction**

Builds the payload that will be inserted or updated in the companies table. If enrichment fails, falls back to the minimal data from the parser.

```javascript
const apiItem = $input.all()[0].json || {}

// enrichment succeed check
let success   = Array.isArray(apiItem.companies) &&
                  apiItem.companies.length > 0;

let companyPayload;
if (success) {
  // Extract Companies Attribute
  const c = apiItem.companies[0];
  //build the data
  companyPayload = {
    name : c.about.name        || $('email  parser').first().json.email,
    domain       : c?.domain?.domain      || $('email  parser').first().json.domain,
    description  : c?.descriptions?.primary ?? null,
    logo     : c?.assets?.logoSquare.src     ?? null,
  };
} else {
 //Fallback to our self-constructed data
  companyPayload = {
    name :  $('email  parser').first().json.company,
    domain       : $('email  parser').first().json.domain,
    description  : null,
    logo     : null,
  };
}

//return our constructed data to be inserted or updated in database
return [
  {
    json: {
      lead_id : $('email  parser').first().json.id, 
      ...companyPayload,
      success,
      "apitem":apiItem.statusCode
    }
  }
];
```

**Output:**

```json

[
    {
    "lead_id": "54",
    "name": "Summit Logistics Inc",
    "domain": "summitlogisticsinc.com",
    "description": "Best rates Need a lane quoted? Scroll down to our "contact us" and fill out the form! We would love to quote a lane for you. On time delivery Efficiency is about quality, that is why we hire the best drivers to make sure your loads are delivered safely and on time. Quality control Our... Read More »Home",
    "logo": "https://thecompaniesapi.s3.fr-par.scw.cloud/companies/logos/square/summitlogisticsinc.com.jpg",
    "success": true
    }
]
```


### **Upsert company**

Handles inserts and updates safely, escaping single quotes in the description:

```sql
INSERT INTO companies (name, domain, description, logo_url)
VALUES ('{{ $json.name }}',
        '{{ $json.domain }}',
        '{{ $json.description.replace(/'/g, "''") }}',
        '{{ $json.logo }}')
ON CONFLICT (domain)
DO UPDATE
   SET name        = COALESCE(EXCLUDED.name,        companies.name),
       description = COALESCE(EXCLUDED.description, companies.description),
       logo_url    = COALESCE(EXCLUDED.logo_url,    companies.logo_url)
RETURNING id, name, domain;
```
