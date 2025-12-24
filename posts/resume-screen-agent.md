---
title: API Credentials Setup Guide - Candidate Screening Workflow
date: 2024-11-12
---

This guide walks you through setting up all the API credentials needed for the Day 16 Candidate Screening AI Agent workflow in n8n Cloud.

## Prerequisites

- An n8n Cloud account (sign up at [n8n.io](https://n8n.io))
- A Google account (Gmail)
- Access to Google Cloud Console (for API keys)

---

## Overview of Required Credentials

This workflow requires **3 types of credentials**:

1. **Google Gemini API** - For AI-powered resume analysis
2. **Gmail OAuth2** - For sending email notifications
3. **Google Sheets OAuth2** - For logging candidates to a spreadsheet

### 📸 Navigating the n8n Credentials Interface

**Visual Reference:** When you open the Credentials section in n8n Cloud, you'll see:
- A **left sidebar** with tabs including "Credentials" highlighted
- A **search bar** at the top to find credential types
- A list of **existing credentials** (if any) with service logos/icons
- An **"+ Add Credential"** button or option in the menu
- A **pop-up window** that appears when adding new credentials, showing:
  - A search field to find the service (e.g., "gmail", "google sheets")
  - A list of matching credential types to select
  - Once selected, a configuration form with required fields

**Documentation:** [n8n Credentials Documentation](https://docs.n8n.io/credentials/add-edit-credentials/) - General guide on creating and editing credentials

---

## Step 1: Set Up Google Gemini API Credentials

The Google Gemini API is used by the AI Agent to analyze resumes and rate candidates.

**📸 What You'll See:** The n8n credentials interface shows a searchable list of available integrations. When you search for "Google Gemini" or "Google PaLM", you'll see a credential type with a simple API key input field.

### 1.1 Get Your Google Gemini API Key

1. Go to [Google AI Studio](https://makersuite.google.com/app/apikey) or [Google Cloud Console](https://console.cloud.google.com/)
2. Sign in with your Google account
3. Navigate to **APIs & Services** → **Credentials**
4. Click **+ CREATE CREDENTIALS** → **API Key**
5. Copy your API key (you'll need this in n8n)
6. **Important:** Restrict the API key to "Generative Language API" for security

### 1.2 Add Credentials in n8n Cloud

1. Log in to your [n8n Cloud](https://app.n8n.cloud) account
2. Click on your profile icon (top right) → **Settings**
3. Navigate to **Credentials** in the left sidebar
4. Click **+ Add Credential**
5. Search for **"Google Gemini"** or **"Google PaLM"**
6. Select **Google Gemini (PaLM) API**
7. Enter a name for your credential (e.g., "Google Gemini API Key")
8. Paste your API key in the **API Key** field
9. Click **Save**

✅ **Test:** You can test the connection by clicking "Test" if available.

**📸 Visual Guide:**
- **n8n Documentation:** [Google Gemini (PaLM) credentials setup](https://docs.n8n.io/integrations/builtin/credentials/googleai/) - Contains screenshots of the credential setup interface
- **Image Reference:** The n8n credentials page shows a search interface where you can type "Google Gemini" or "Google PaLM" to find the credential type. The credential form will display a single "API Key" field where you paste your key from Google AI Studio.

---

## Step 2: Set Up Gmail OAuth2 Credentials

Gmail credentials are used to send two types of emails:
- Confirmation email to candidates
- Notification email to HR team

**📸 What You'll See:** The Gmail OAuth2 setup in n8n displays a form with OAuth Redirect URL, Client ID, and Client Secret fields. A yellow banner provides helpful instructions about using the redirect URL in Google Cloud Console.

### 2.1 Create OAuth 2.0 Credentials in Google Cloud Console

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Navigate to **APIs & Services** → **Credentials**
4. Click **+ CREATE CREDENTIALS** → **OAuth client ID**
5. If prompted, configure the OAuth consent screen:
   - Choose **External** (unless you have a Google Workspace)
   - Fill in required fields:
     - App name: "n8n Gmail Integration"
     - User support email: Your email
     - Developer contact: Your email
   - Click **Save and Continue**
   - Add scopes: `https://www.googleapis.com/auth/gmail.send`
   - Click **Save and Continue**
   - Add test users (your email) if needed
   - Click **Save and Continue** → **Back to Dashboard**

6. Back in Credentials, click **+ CREATE CREDENTIALS** → **OAuth client ID**
7. Choose **Web application** as the application type
8. Name it: "n8n Gmail OAuth"
9. Under **Authorized redirect URIs**, add:
   ```
   https://app.n8n.cloud/rest/oauth2-credential/callback
   ```
   (If using self-hosted n8n, use your n8n URL instead)
10. Click **Create**
11. **Copy both the Client ID and Client Secret** (you'll need these)

### 2.2 Enable Gmail API

1. In Google Cloud Console, go to **APIs & Services** → **Library**
2. Search for **"Gmail API"**
3. Click on it and press **Enable**

### 2.3 Add Gmail Credentials in n8n Cloud

1. In n8n Cloud, go to **Settings** → **Credentials**
2. Click **+ Add Credential**
3. Search for **"Gmail"**
4. Select **Gmail OAuth2 API**
5. Enter a name (e.g., "My Gmail Account")
6. Fill in the fields:
   - **Client ID:** Paste your OAuth Client ID from Step 2.1
   - **Client Secret:** Paste your OAuth Client Secret from Step 2.1
7. Click **Connect my account**
8. You'll be redirected to Google to authorize n8n
9. Select your Gmail account and click **Allow**
10. You'll be redirected back to n8n
11. Click **Save**

✅ **Test:** The credential should now show as connected.

**📸 Visual Guide:**
- **n8n Documentation:** [Gmail OAuth2 credentials setup](https://docs.n8n.io/integrations/builtin/credentials/gmail/) - Shows the OAuth2 credential form
- **Image Reference:** The n8n Gmail OAuth2 credential setup page displays:
  - An **OAuth Redirect URL** field (pre-filled with `http://localhost:5678/rest/oauth2-credential/callback` for self-hosted, or `https://app.n8n.cloud/rest/oauth2-credential/callback` for n8n Cloud)
  - **Client ID** field (required)
  - **Client Secret** field (required)
  - A yellow banner with instructions: "In Google Drive, use the URL above when prompted to enter an OAuth callback or redirect URL"
  - A **"Connect my account"** button that initiates the OAuth flow
- **Video Tutorial:** [Connect n8n to the Google API in 5 minutes | Sheets, Drive, Gmail, etc.](https://www.youtube.com/watch?v=miUnVwkndbM) - Visual walkthrough of the OAuth setup process

---

## Step 3: Set Up Google Sheets OAuth2 Credentials

Google Sheets credentials are used to automatically log candidate information and AI ratings.

**📸 What You'll See:** Similar to Gmail, the Google Sheets OAuth2 credential form shows the redirect URL, Client ID, and Client Secret fields. The interface includes a reminder to enable the Google Sheets API in Google Cloud Console.

### 3.1 Create OAuth 2.0 Credentials for Google Sheets

1. In [Google Cloud Console](https://console.cloud.google.com/), go to **APIs & Services** → **Credentials**
2. Click **+ CREATE CREDENTIALS** → **OAuth client ID**
3. Choose **Web application**
4. Name it: "n8n Google Sheets OAuth"
5. Under **Authorized redirect URIs**, add:
   ```
   https://app.n8n.cloud/rest/oauth2-credential/callback
   ```
6. Click **Create**
7. **Copy both the Client ID and Client Secret**

### 3.2 Enable Google Sheets API

1. In Google Cloud Console, go to **APIs & Services** → **Library**
2. Search for **"Google Sheets API"**
3. Click on it and press **Enable**

### 3.3 Add Google Sheets Credentials in n8n Cloud

1. In n8n Cloud, go to **Settings** → **Credentials**
2. Click **+ Add Credential**
3. Search for **"Google Sheets"**
4. Select **Google Sheets OAuth2 API**
5. Enter a name (e.g., "My Google Sheets")
6. Fill in the fields:
   - **Client ID:** Paste your OAuth Client ID from Step 3.1
   - **Client Secret:** Paste your OAuth Client Secret from Step 3.1
7. Click **Connect my account**
8. Authorize n8n to access Google Sheets
9. Click **Save**

✅ **Test:** The credential should now show as connected.

**📸 Visual Guide:**
- **n8n Documentation:** [Google Sheets OAuth2 credentials setup](https://docs.n8n.io/integrations/builtin/credentials/googlesheets/) - Contains screenshots of the credential configuration
- **Image Reference:** The Google Sheets OAuth2 setup interface in n8n is similar to Gmail:
  - **OAuth Redirect URL** field showing the callback URL
  - **Client ID** and **Client Secret** input fields
  - Yellow information banner: "Make sure that you have enabled the Google Sheets API in the Google Cloud Console"
  - **"Connect my account"** button to start OAuth authorization
- **Video Tutorial:** [Connect n8n to the Google API in 5 minutes](https://www.youtube.com/watch?v=miUnVwkndbM) - Covers Google Sheets setup along with other Google services
- **Blog Post:** [Sending Automated Congratulations with Google Sheets, Twilio, and n8n](https://blog.n8n.io/sending-automated-congratulations-with-google-sheets-twilio-and-n8n/) - Includes screenshots of Google Sheets integration

---

## Step 4: Prepare Your Google Sheet

Before using the workflow, create a Google Sheet with the following columns:

1. Open [Google Sheets](https://sheets.google.com)
2. Create a new spreadsheet
3. Name it: "Candidate Screening Results"
4. Add these column headers in Row 1:
   - **Full Name**
   - **E-mail**
   - **Linkedin**
   - **Expectation**
   - **CV**
   - **AI Rating**
5. Copy the **Spreadsheet ID** from the URL:
   ```
   https://docs.google.com/spreadsheets/d/[SPREADSHEET_ID]/edit
   ```
6. You'll use this ID when configuring the Google Sheets node in the workflow

---

## Step 5: Assign Credentials to Workflow Nodes

Once all credentials are set up, you need to assign them to the workflow nodes:

### 5.1 Import the Day 16 Workflow

1. In n8n Cloud, click **+ Add workflow**
2. Click the **⋮** menu (three dots) → **Import from File**
3. Upload the `Day 16.json` file
4. The workflow will import with placeholder credentials

### 5.2 Assign Google Gemini Credentials

1. Click on the **"Google Gemini Chat Model"** node
2. In the **Credential** dropdown, select your Google Gemini credential
3. The node should now show your credential name

### 5.3 Assign Gmail Credentials

1. Click on the **"Confirmation of CV Submission"** node
2. In the **Credential** dropdown, select your Gmail OAuth2 credential
3. Repeat for the **"Inform HR New CV Received"** node

### 5.4 Assign Google Sheets Credentials

1. Click on the **"Candidate Lists"** node
2. In the **Credential** dropdown, select your Google Sheets OAuth2 credential
3. Update the **Document ID** field with your spreadsheet ID from Step 4

---

## Troubleshooting

### Issue: "Invalid API Key" for Google Gemini

**Solution:**
- Verify you copied the full API key
- Check that the API key is enabled in Google Cloud Console
- Ensure "Generative Language API" is enabled for your project

### Issue: "OAuth2 connection failed" for Gmail/Sheets

**Solution:**
- Verify the redirect URI matches exactly: `https://app.n8n.cloud/rest/oauth2-credential/callback`
- Check that Gmail API or Google Sheets API is enabled in Google Cloud Console
- Try disconnecting and reconnecting the credential
- Ensure you're using the correct OAuth client (not a service account)

### Issue: "Permission denied" when writing to Google Sheets

**Solution:**
- Make sure the Google account you authorized has edit access to the spreadsheet
- Share the spreadsheet with the authorized email if needed
- Check that the spreadsheet ID is correct

### Issue: "Email not sending" from Gmail

**Solution:**
- Verify the Gmail account has "Less secure app access" enabled (if required)
- Check that the "Send email" scope was added during OAuth consent screen setup
- Ensure the email addresses in the workflow are valid

---

## Security Best Practices

1. **Restrict API Keys:** Limit your Google Gemini API key to only the APIs you need
2. **Use OAuth2:** Always use OAuth2 for Gmail and Google Sheets (more secure than API keys)
3. **Regular Rotation:** Rotate your API keys periodically
4. **Monitor Usage:** Check Google Cloud Console regularly for unexpected API usage
5. **Limit Scope:** Only grant the minimum permissions needed (e.g., `gmail.send` only, not full Gmail access)

---

## Quick Checklist

Before running the workflow, verify:

- [ ] Google Gemini API key is set up and working
- [ ] Gmail OAuth2 credentials are connected
- [ ] Google Sheets OAuth2 credentials are connected
- [ ] Google Sheet is created with correct column headers
- [ ] Spreadsheet ID is updated in the "Candidate Lists" node
- [ ] All nodes have credentials assigned
- [ ] Workflow is activated (toggle in top right)

---

## Next Steps

Once all credentials are set up:

1. **Test the workflow** by submitting a test application through the form
2. **Customize the AI prompt** in the "Using AI Analysis & Rating" node for your specific job requirements
3. **Update email addresses** in the Gmail nodes to match your HR team
4. **Share the form link** from the "Application Form" node with candidates

---

## Need Help?

If you encounter issues:

1. Check the n8n [documentation](https://docs.n8n.io)
2. Review the [n8n community forum](https://community.n8n.io)
3. Verify all APIs are enabled in Google Cloud Console
4. Double-check redirect URIs match exactly

---

**Congratulations!** You're now ready to use the AI Candidate Screening workflow. 🎉

