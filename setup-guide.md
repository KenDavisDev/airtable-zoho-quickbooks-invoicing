# Setup Guide - Invoicing Automation

This guide walks through setting up the complete invoicing system from scratch.

## Overview

This workflow connects three systems:
1. **Airtable** - Stores time logs, expenses, and invoice records
2. **Zoho Books** - Creates online invoices for clients
3. **QuickBooks Desktop** - Maintains your accounting records via IIF import

## Part 1: Airtable Structure

### Table 1: Time Logs

Create this table to track billable hours:

| Field Name | Type | Settings |
|------------|------|----------|
| Date | Date | Date work was performed |
| Title | Single line text | Short description of work |
| Description | Long text | Detailed notes (optional) |
| Total Hours | Number | Precision: 2 decimals, format: Duration |
| Project Name | Single line text | What this work is for |
| Client | Linked record | Link to Clients table |
| Invoice | Linked record | Link to Invoicing table (set when invoiced) |

### Table 2: Expenses

Create this table to track reimbursable expenses:

| Field Name | Type | Settings |
|------------|------|----------|
| Expense Name | Single line text | What the expense was for |
| Date | Date | When expense occurred |
| Amount | Currency | Expense amount |
| Category | Single select | Options: Travel, Meals, Software, Supplies, Other |
| Receipt | Attachment | Receipt image/PDF |
| Client | Linked record | Link to Clients table |
| Invoice | Linked record | Link to Invoicing table (set when invoiced) |
| Submitted | Checkbox | Ready for processing |
| Synced to Dropbox | Checkbox | Receipt backed up |

### Table 3: Clients

Basic client information:

| Field Name | Type | Settings |
|------------|------|----------|
| Client Name | Single line text | Company or person name |
| Zoho Customer ID | Number | ID from Zoho Books (required for invoicing) |
| Billable Rate | Currency | Default hourly rate |
| Time Logs | Linked records | Shows all time entries |
| Expenses | Linked records | Shows all expenses |
| Invoices | Linked records | Shows all invoices |

### Table 4: Invoicing

The main invoice records:

| Field Name | Type | Formula/Settings |
|------------|------|------------------|
| RecordID | Formula | `CONCATENATE(YEAR({Date}), IF(MONTH({Date})<10, "0", ""), MONTH({Date}), IF(DAY({Date})<10, "0", ""), DAY({Date}), "-", {Client Name})` |
| Client Name | Linked record | Link to Clients table (single) |
| Date | Date | Invoice date |
| Time Logs | Linked records | Link to Time Logs table (multiple) |
| Expenses | Linked records | Link to Expenses table (multiple) |
| Billable Rate | Lookup | From Client.Billable Rate |
| Zoho Customer ID | Lookup | From Client.Zoho Customer ID |
| Time Total | Rollup | Sum of Time Logs.Total Hours × Billable Rate |
| Expense Total | Rollup | Sum of Expenses.Amount |
| Invoice Total | Formula | `{Time Total} + {Expense Total}` |
| Ready to Invoice | Checkbox | Trigger for automation |
| Zoho Invoice ID | Single line text | Set after creation (optional) |

## Part 2: Zoho Books Setup

### 1. Create Zoho Books Account

1. Go to zoho.com/books
2. Sign up for account (free trial available)
3. Complete organization setup

### 2. Add Customers

For each client:
1. Go to **Customers** → **New Customer**
2. Enter customer details
3. Note the **Customer ID** (visible in URL or customer details)
4. Copy this ID to Airtable Clients table

### 3. Create Line Item Categories

1. Go to **Settings** → **Items**
2. Create items for common services:
   - "Consulting Services"
   - "Database Design"
   - "Workflow Automation"
   - Etc.
3. Create items for expense categories:
   - "Travel Expenses"
   - "Software Subscriptions"
   - Etc.

**Note**: The workflow uses the Project Name and Category fields as line item names, so these should match what you want to appear on invoices.

### 4. Setup OAuth2 Credentials

1. Go to Zoho Developer Console: api-console.zoho.com
2. Click **Add Client** → **Server-based Applications**
3. Fill in:
   - Client Name: "n8n Invoicing"
   - Homepage URL: Your n8n instance URL
   - Authorized Redirect URIs: `https://your-n8n.com/rest/oauth2-credential/callback`
4. Save and note the **Client ID** and **Client Secret**
5. In n8n:
   - Create new OAuth2 credential
   - Use Client ID and Client Secret
   - Authorization URL: `https://accounts.zoho.com/oauth/v2/auth`
   - Access Token URL: `https://accounts.zoho.com/oauth/v2/token`
   - Scope: `ZohoInvoice.invoices.CREATE`
6. Connect and authorize

## Part 3: QuickBooks Desktop Setup

### 1. Create Required Accounts

In QuickBooks, ensure these accounts exist:

**Chart of Accounts:**
- **Accounts Receivable** (Asset account, type: Accounts Receivable)
- **Income Account** (Income account, your main income account)
- **Reimbursable Expenses** (Income or Other Income account)

If these don't exist, create them:
1. Lists → Chart of Accounts
2. Account → New
3. Choose appropriate type and name exactly as shown

### 2. Add Customers

Customer names in QuickBooks must match the client names from Airtable:
1. Customers → Customer Center
2. New Customer
3. Name must match what appears after the dash in Invoice RecordID
   - Example: If RecordID is "20260130-AcmeAuto", customer name is "AcmeAuto"

### 3. Create Service Items

1. Lists → Item List
2. New → Service
3. Create items matching your Project Names:
   - "Database Optimization"
   - "Workflow Design"
   - Etc.
4. Assign to "Income Account"

### 4. Create Expense Items

1. Lists → Item List
2. New → Service
3. Create items matching your Categories:
   - "Travel"
   - "Software"
   - Etc.
4. Assign to "Reimbursable Expenses" account

## Part 4: n8n Workflow Setup

### 1. Import Workflow

1. Download workflow.json
2. n8n → Import from File
3. Open the imported workflow

### 2. Configure All Nodes

**Workflow Configuration node:**
- Update `zohoOrganizationId` with your Zoho org ID
- Find it: Zoho Books → Settings → Organization Profile

**Get Invoice node:**
- Base ID: Your Airtable base ID
- Table ID: Invoicing table ID
- Credential: Your Airtable token

**Get Time Log Record node:**
- Base ID: Same as above
- Table ID: Time Logs table ID
- Credential: Same Airtable token

**Get Expense Record node:**
- Base ID: Same as above
- Table ID: Expenses table ID  
- Credential: Same Airtable token

**Create Invoice in Zoho node:**
- Credential: Your Zoho OAuth2 credential
- Test with a sample invoice first

**Upload QB File to Dropbox node:**
- Path: Set to your preferred folder
- Credential: Your Dropbox OAuth2

### 3. Test Each Path

**Test 1: Time Only**
1. Create test invoice with only time logs
2. Send test webhook: `POST https://your-n8n.com/webhook/invoicing`
```json
{ "recordId": "YOUR_TEST_INVOICE_ID" }
```
3. Check workflow execution in n8n
4. Verify Zoho invoice created
5. Check IIF file in Dropbox

**Test 2: Expenses Only**
1. Create test invoice with only expenses
2. Trigger via webhook
3. Verify results

**Test 3: Mixed**
1. Create invoice with both
2. Trigger and verify

### 4. Setup Airtable Automation

1. Airtable → Automations → Create
2. Name: "Process Invoice"
3. Trigger: When record matches conditions
   - Table: Invoicing
   - When: Field "Ready to Invoice" = checked
4. Action: Send webhook request
   - URL: Your n8n webhook URL (from Webhook node)
   - Method: POST
   - Body: `{"recordId": "AIRTABLE_RECORD_ID()"}`
5. Test with a sample record
6. Turn on automation

## Part 5: Daily Workflow

### For the User (You)

**Recording Time:**
1. Add entries to Time Logs table as you work
2. Link to appropriate Client

**Recording Expenses:**
1. Add expenses to Expenses table
2. Upload receipt
3. Check "Submitted" to trigger receipt backup

**Creating Invoice:**
1. Go to Invoicing table
2. Create new record
3. Set Client, Date
4. Link relevant Time Logs
5. Link relevant Expenses
6. Review totals
7. Check "Ready to Invoice"
8. Automation runs automatically

**After Invoice Created:**
1. Check Zoho Books for created invoice
2. Download IIF file from Dropbox
3. Import to QuickBooks Desktop
4. Review in QuickBooks
5. Send invoice to client (via Zoho or QuickBooks)

## Part 6: Monthly Maintenance

**Week 1:**
- Review time logs for accuracy
- Ensure all receipts are uploaded and backed up

**Week 2:**
- Prepare invoices for all clients
- Link appropriate time logs and expenses

**Week 3:**
- Trigger invoice creation
- Import IIF files to QuickBooks
- Send invoices to clients

**Week 4:**
- Track payments
- Follow up on overdue invoices

## Troubleshooting Common Issues

**IIF file won't import:**
- Customer name mismatch between Airtable and QuickBooks
- Accounts don't exist in QuickBooks
- Date format issues (should be MM/DD/YYYY)

**Zoho invoice creation fails:**
- Customer ID not found in Zoho Books
- Missing required fields
- OAuth2 token expired (reconnect in n8n)

**Webhook doesn't trigger:**
- Check Airtable automation is ON
- Verify webhook URL is correct
- Look at n8n execution history for errors

**Wrong amounts calculated:**
- Check Billable Rate is set correctly
- Verify Total Hours are accurate
- Ensure Expense amounts are correct

## Need Help?

Open an issue in this repository or contact [lodgepole.io](https://lodgepole.io) for assistance.
