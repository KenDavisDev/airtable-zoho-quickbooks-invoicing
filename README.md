# Invoicing Automation - Time & Expenses

A webhook-triggered n8n workflow that generates invoices from Airtable time entries and expenses, creates Zoho invoices, and exports QuickBooks Desktop IIF files - all automatically.

## What This Does

When you're ready to invoice a client, mark the invoice record as ready in Airtable. This triggers a webhook that:

1. Fetches the invoice record from Airtable
2. Retrieves all associated time logs and expenses
3. Transforms them into Zoho invoice line items
4. Creates the invoice in Zoho Books
5. Generates a QuickBooks Desktop IIF file with proper account categorization
6. Uploads the IIF file to Dropbox for easy import

Perfect for consultants, freelancers, or service businesses that track time and expenses separately but need to bill them together.

## Key Features

- **Flexible Billing**: Handles time-only, expenses-only, or combined invoices
- **Multi-Source Data**: Combines time entries and expenses into a single invoice
- **Triple Output**: Creates Zoho invoice, downloads PDF, and generates QB Desktop IIF file
- **Smart Categorization**: Time goes to "Income Account", expenses to "Reimbursable Expenses"
- **Slack Notifications**: Real-time alerts when invoice creation fails
- **Error Handling**: Validates required data before creating invoices
- **Automatic File Naming**: `ClientName YYYYMMDD Invoice.iif` and matching PDF for easy organization

## Use Cases

- Monthly retainer billing with mixed time and expenses
- Project-based invoicing with reimbursable costs
- Consultant billing for services + travel/software expenses
- Freelancer invoicing with billable hours and expense pass-through
- Archiving invoice PDFs alongside QuickBooks files for record-keeping

## Data Flow

```
Airtable Automation triggers webhook
  → n8n fetches Invoice record
  → Check for Time Logs OR Expenses (at least one required)
    ├─ Has Time Logs?
    │   └─ Fetch each time log record
    └─ Has Expenses?
        └─ Fetch each expense record
  → Merge all items together
  → Transform to Zoho format
    • Time entries: quantity = hours, rate = billable rate
    • Expenses: quantity = 1, rate = expense amount
  → Validate required fields
    ├─ Missing data → Send Slack notification → Stop
    └─ Valid → Continue
  → Create Zoho invoice via API
    ├─ Success → Continue
    └─ Failure → Send Slack notification → Stop
  → Download invoice PDF from Zoho
  → Upload PDF to Dropbox
  → Generate QuickBooks IIF file
    • Time entries → Income Account
    • Expenses → Reimbursable Expenses
  → Upload IIF to Dropbox
```

## Airtable Schema

### Invoicing Table

| Field Name | Type | Description |
|------------|------|-------------|
| RecordID | Formula | Unique identifier (e.g., "20260130-ClientName") |
| Date | Date | Invoice date |
| Time Logs | Linked records | Links to Time Logs table |
| Expenses | Linked records | Links to Expenses table |
| Billable Rate | Number | Hourly rate for time entries |
| Zoho Customer ID | Number | Customer ID from Zoho Books |
| Invoice Total | Formula | Calculated total amount |

### Time Logs Table

| Field Name | Type | Description |
|------------|------|-------------|
| Date | Date | When work was performed |
| Title | Single line text | Description of work |
| Total Hours | Number | Hours worked |
| Project Name | Single line text | What to show as line item name |

### Expenses Table

| Field Name | Type | Description |
|------------|------|-------------|
| Expense Name | Single line text | What the expense was for |
| Date | Date | When expense occurred |
| Amount | Currency | Expense amount |
| Category | Single select | Expense category (Travel, Software, etc.) |

## Setup Instructions

### 1. Prerequisites

- n8n instance (cloud or self-hosted)
- Airtable with Invoicing, Time Logs, and Expenses tables
- Zoho Books account with OAuth2 configured
- Dropbox account for IIF file storage
- QuickBooks Desktop (for importing IIF files)

### 2. Import the Workflow

1. Download `workflow.json` from this repository
2. In n8n: **Import from File**
3. Select the JSON file

### 3. Configure Credentials

Replace placeholder credential IDs:

**Airtable Personal Access Token** (`YOUR_AIRTABLE_CREDENTIAL_ID`)
- Scopes: `data.records:read`, `schema.bases:read`
- Used in: Get Invoice, Get Time Log Record, Get Expense Record

**Zoho OAuth2** (`YOUR_ZOHO_OAUTH_CREDENTIAL_ID`)
- Scopes: `ZohoInvoice.invoices.CREATE`
- Setup in Zoho Developer Console
- Used in: Create Invoice in Zoho

**Dropbox OAuth2** (`YOUR_DROPBOX_CREDENTIAL_ID`)
- Scopes: `files.content.write`
- Used in: Upload QB file to Dropbox, Upload PDF to Dropbox

**Slack OAuth2** (`YOUR_SLACK_CREDENTIAL_ID` and `YOUR_SLACK_USER_ID`)
- Scopes: `chat:write`, `users:read`
- Used in: Error notification messages
- Find your Slack User ID: Click your profile → More → Copy member ID
- Optional: Remove Slack nodes if you don't want notifications

### 4. Configure IDs and Settings

**Airtable Base & Table IDs:**
- `YOUR_AIRTABLE_BASE_ID` - Your base ID (starts with `app...`)
- `YOUR_INVOICING_TABLE_ID` - Invoicing table ID (starts with `tbl...`)
- `YOUR_TIME_LOGS_TABLE_ID` - Time Logs table ID
- `YOUR_EXPENSES_TABLE_ID` - Expenses table ID

**Zoho Organization ID:**
- `YOUR_ZOHO_ORG_ID` - Found in Zoho Books Settings → Organization Profile

**Dropbox Path:**
Update in "Upload QB File to Dropbox" node:
```javascript
path: "=/Invoicing/{{ $json.filename }}"
```
Change `/Invoicing/` to your preferred folder.

### 5. Setup Airtable Automation

Create an automation in Airtable:

**Trigger**: When record matches conditions
- Table: Invoicing
- Conditions: `Ready to Invoice = checked` (or your trigger field)

**Action**: Send webhook request
- URL: `https://your-n8n.com/webhook/invoicing`
- Method: POST
- Body:
```json
{
  "recordId": "AIRTABLE_RECORD_ID()"
}
```

### 6. Test the Workflow

**Test Case 1: Time Entries Only**
1. Create invoice with linked time logs, no expenses
2. Trigger automation
3. Verify Zoho invoice created with time line items
4. Check IIF file shows entries under "Income Account"

**Test Case 2: Expenses Only**
1. Create invoice with no time logs, only expenses
2. Trigger automation
3. Verify Zoho invoice created with expense line items
4. Check IIF file shows entries under "Reimbursable Expenses"

**Test Case 3: Mixed Billing**
1. Create invoice with both time logs and expenses
2. Trigger automation
3. Verify both appear as separate line items in Zoho
4. Check IIF file properly categorizes each type

**Test Case 4: No Billable Items**
1. Create invoice with no time logs or expenses
2. Trigger automation
3. Should stop with error: "No billable items found"

### 7. Activate

Once all tests pass, activate the workflow in n8n.

## How Line Items Are Processed

### Time Entries in Zoho Invoice

```
Name: Project Name (from Time Log)
Description: 2024-01-15: Database optimization work
Rate: $150 (from Invoice.Billable Rate)
Quantity: 3.5 hours
Amount: $525.00
```

### Expenses in Zoho Invoice

```
Name: Category (e.g., "Travel", "Software")
Description: 2024-01-18: Flight to client site
Rate: $450.00 (from Expense.Amount)
Quantity: 1
Amount: $450.00
```

### QuickBooks IIF Format

**Time Entry Line:**
```
SPL  INVOICE  01/15/2024  Income Account  ClientName  -525.00  20260115-ClientName  N  3.5  150  DatabaseOptimization  N  2024-01-15: Database optimization work
```

**Expense Line:**
```
SPL  INVOICE  01/18/2024  Reimbursable Expenses  ClientName  -450.00  20260115-ClientName  N  1  450  Travel  N  2024-01-18: Flight to client site
```

## Edge Cases Handled

✅ **Time only** - Works perfectly, only time entries in invoice
✅ **Expenses only** - Works perfectly, only expenses in invoice
✅ **Both time & expenses** - Combined into single invoice
✅ **No billable items** - Stops with clear error message
✅ **Missing Zoho Customer ID** - Caught before API call
✅ **Zoho API failure** - Workflow stops, error message provided
✅ **Multiple entries per invoice** - All combined properly

## Customization Options

### Change Billable Rate Per Time Entry

Currently uses a single rate from the Invoice record. To bill different rates:

1. Add "Rate" field to Time Logs table
2. Modify the JavaScript in "Create Zoho Fields":
```javascript
const rate = item['Rate'] || billableRate; // Use entry rate if available
```

### Add Tax Handling

Zoho supports tax rates per line item:

```javascript
lineItems.push({
  name: projectName,
  description: `${date}: ${title}`,
  rate: billableRate,
  quantity: hours,
  tax_id: "YOUR_TAX_ID" // Add tax
});
```

### Different QuickBooks Accounts

Modify the IIF generation to use custom accounts:

```javascript
// For time entries
iifContent += `SPL...\\tConsulting Income\\t...`; // Instead of "Income Account"

// For expenses  
iifContent += `SPL...\\tExpense Reimbursements\\t...`; // Instead of "Reimbursable Expenses"
```

### Add Invoice Notes

Include custom notes in Zoho invoices by modifying:

```javascript
notes: `Thank you for your business! Invoice for ${invoiceData['RecordID']}`
```

### Email IIF File Instead of Dropbox

Replace the Dropbox node with Gmail node:

```javascript
// In Gmail node
attachments: "data:text/plain;base64,{{ $json.iifContent | base64encode }}"
subject: "QuickBooks Invoice File - {{ $json.filename }}"
```

## QuickBooks Import Instructions

Once the IIF file is in Dropbox:

1. Download the IIF file from Dropbox
2. Open QuickBooks Desktop
3. Go to **File → Utilities → Import → IIF Files**
4. Select the downloaded file
5. QuickBooks will create the invoice

**Note**: IIF format is for QuickBooks Desktop only. QuickBooks Online requires different integration (use Zoho's built-in QB Online sync instead).

## Troubleshooting

**"No billable items found" error:**
- Check that Time Logs or Expenses fields are actually linked
- Verify records exist in linked tables
- Ensure at least one field has data

**Zoho invoice creation fails:**
- Verify Zoho Customer ID is correct and exists in Zoho Books
- Check Zoho OAuth2 credentials are valid
- Look at Zoho API response for specific error message

**IIF file imports incorrectly:**
- Verify client name matches exactly in QuickBooks
- Check that "Income Account" and "Reimbursable Expenses" accounts exist in your QuickBooks chart of accounts
- Ensure date formats are MM/DD/YYYY

**Missing fields in line items:**
- Time entries need: Title, Total Hours, Date, Project Name
- Expenses need: Expense Name, Amount, Date, Category
- Check Airtable field names match exactly (case-sensitive)

**Dropbox upload fails:**
- Verify OAuth2 credentials
- Check folder path exists in Dropbox
- Ensure folder path doesn't have typos

## Production Best Practices

1. **Test with small invoices first** - Start with 1-2 line items before processing large invoices
2. **Backup Airtable** - Export Time Logs and Expenses monthly as CSV backup
3. **Monitor Zoho** - Check that created invoices match expectations
4. **Archive IIF files** - Keep Dropbox organized by moving old files to archive folder
5. **Track failures** - Set up error notifications (add Gmail nodes to error paths)

## Comparison to Manual Process

**Manual workflow:**
1. Open Airtable, review time logs (5 min)
2. Open Airtable, review expenses (3 min)
3. Log into Zoho, create invoice (10 min)
4. Manually enter each line item (15-30 min depending on quantity)
5. Export to QB Desktop format somehow (10-15 min)
6. Import to QuickBooks (5 min)

**Total: 45-70 minutes per invoice**

**Automated workflow:**
1. Check invoice is ready in Airtable (1 min)
2. Trigger automation (1 click)
3. Review Zoho invoice (2 min)
4. Import IIF to QuickBooks (2 min)

**Total: 5 minutes per invoice**

**Time saved: 40-65 minutes per invoice**

## Future Enhancements

Ideas for extending this workflow:

- Add email notification when invoice is created
- Auto-update Airtable with Zoho invoice ID and link
- Mark time logs and expenses as "Invoiced" automatically
- Generate PDF invoice and attach to email
- Send invoice directly to client via Zoho API
- Track invoice payment status back to Airtable

## License

MIT License - See LICENSE file for details

## About

Created by [Ken Davis](https://github.com/KenDavisDev) at [Lodgepole I/O](https://lodgepole.io) - Workflow design and operational systems consulting.

This workflow is used in production for monthly client invoicing, processing $10K-50K in billings per month since 2024.

## Related Workflows

- [Email-Based Time Tracking](../gmail-to-airtable-time-tracking) - How time entries get into Airtable
- [Receipt Processing](../receipt-processing-automation) - How expenses get into Airtable
