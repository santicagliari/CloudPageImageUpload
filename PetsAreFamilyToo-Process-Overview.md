# Pets Are Family Too — How the Submission Page Works

## What this is

A Cloud Page in Marketing Cloud where residents submit a photo or video of their pet, a short story, and a signed release form. Submissions are logged in a Data Extension, and the photo/video file itself is stored in Box.

## Systems involved

| System | Role |
|---|---|
| **SFMC Cloud Page** (`PetsAreFamilyToo.html`) | Hosts the form. Looks up the resident's name, records the submission. Uses only native AMPscript/SSJS — **no external Marketing Cloud API calls**. |
| **Browser (JavaScript)** | Validates the file, shrinks photos, uploads the file, then submits the form. |
| **Heroku app** (`sfmc-box-upload`) | A small Node.js service whose only job is: take a file, hand it to Box. |
| **Box** | Where the actual photo/video files end up. |
| **Data Extension** (`MSR_Operations_DE_PetsAreFamilyToo_FormResults`) | Stores the form answers + a reference to the file's name — not the file itself. |

## Step by step

1. **Resident opens the link** (contains their `contact_id` and `lease_id`).
   The page looks up their name from the CRM contact record so it can be used later in the file name.

2. **Resident fills out the form**: uploads a photo or video, writes their story, signs the release, lists any minors/additional adults.

3. **On Submit, the browser does two things in sequence — before the page itself is ever submitted:**
   - If it's a **photo**: shrinks/re-compresses it in the browser (so it's small and fast to upload).
   - If it's a **video**: leaves it as-is, up to 25MB.
   - It builds a file name like `JohnDoe_PetsAreFamilyToo_08052026_123.mp4`.
   - It uploads that file **directly from the browser to the Heroku app** (not through Salesforce).

4. **The Heroku app receives the file**, checks a shared secret to confirm the request is legitimate, logs into Box using a service account, and uploads the file to the designated Box folder.

5. **Once the upload succeeds**, the browser submits the actual Cloud Page form — this part is now just small text fields (name, story, dates, consent, the file's name) — no file data.

6. **The Cloud Page's server-side script** validates those fields and writes one row into the Data Extension, including the file name so anyone can match a form submission back to its file in Box.

7. **The resident sees a thank-you message** confirming their entry.

## Why the upload happens directly to Heroku (and not through the Cloud Page form)

Salesforce Marketing Cloud Cloud Pages have a practical limit on how large a single form submission can be. A photo is small enough to fit, but a video is not — especially once encoded for transmission. Routing the file straight from the browser to the Heroku app sidesteps that limit; the Cloud Page itself only ever handles small text data.

## Marketing Cloud API usage

**None.** The Cloud Page's server-side logic (name lookup, saving the submission) runs entirely through built-in AMPscript/SSJS functions inside the page — not through the external Marketing Cloud REST or SOAP API. The Heroku app doesn't call Marketing Cloud either; it only talks to Box.

## Known limitation

- Videos are capped at 25MB.
- The file goes to Heroku before the resident's info is confirmed as saved — in rare cases (e.g., the resident closes the tab right after upload but before the final submit), a file could land in Box without a matching Data Extension row.
