# Hardware Request Workflow - Quick Start Guide

## ⚡ 5-Minute Setup

### Prerequisites
- n8n instance (cloud or self-hosted)
- Email account (Gmail, Office 365, or SMTP)
- Database (Airtable, PostgreSQL, or MongoDB)
- Basic understanding of n8n

---

## 🚀 Step 1: Import the Workflow

1. Open your n8n instance
2. Click **Workflows** in the sidebar
3. Click **+New** → **Import from file**
4. Select `hardware-request-workflow.json`
5. Click **Import**

The workflow will appear with all 20 nodes already configured.

---

## 🔧 Step 2: Configure Credentials

### Email Setup (5 minutes)

**For Gmail:**
1. Go to **Credentials** (bottom left)
2. Click **+New**
3. Select **Gmail**
4. Authorize with your Google account
5. Name it "Hardware Request Email"

**For SMTP (Office 365/Custom):**
1. Go to **Credentials**
2. Click **+New**
3. Select **SMTP**
4. Enter:
   - Host: `smtp.office365.com` (or your SMTP server)
   - Port: `587`
   - User: Your email
   - Password: Your app password
5. Name it "Hardware Request Email"

### Database Setup (5 minutes)

**For Airtable:**
1. Get your Airtable API key: https://airtable.com/account/tokens
2. In n8n, go to **Credentials** → **+New** → **Airtable**
3. Paste your API key
4. Name it "Airtable Hardware"
5. Create a base called "Hardware Requests" in Airtable
6. Create a table called "hardwareRequests" with these fields:
   - requestId (Text)
   - employeeName (Text)
   - employeeEmail (Email)
   - hardwareType (Text)
   - status (Single select)
   - approvals (JSON)
   - assignedMember (JSON)
   - requestDate (Date)

**For PostgreSQL:**
1. In n8n, go to **Credentials** → **+New** → **PostgreSQL**
2. Enter your database connection details
3. Name it "Hardware DB"
4. Run this SQL:
```sql
CREATE TABLE hardware_requests (
  request_id VARCHAR(50) PRIMARY KEY,
  employee_id VARCHAR(50),
  employee_name VARCHAR(255),
  employee_email VARCHAR(255),
  hardware_type VARCHAR(100),
  status VARCHAR(50),
  approvals JSONB,
  assigned_member JSONB,
  request_date TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## ✏️ Step 3: Update Email Addresses

Edit these nodes in the workflow:

1. **"Send to IT Manager for Approval"**
   - Find line: `"sendTo": "=itoperationmanager@company.com"`
   - Replace with actual IT Manager email

2. **"Send Completion Report"**
   - Find line: `"sendTo": "hardwarerequests@company.com"`
   - Replace with your HR/Admin email

3. **"Assign to IT Operation Member"** (Function node)
   - Update the `members` array:
   ```javascript
   const members = [
     { name: 'John Doe', email: 'john.doe@company.com' },
     { name: 'Jane Smith', email: 'jane.smith@company.com' }
   ];
   ```

---

## 📍 Step 4: Connect Database Nodes

### For "Store Request in Database" node:
1. Click the node
2. Select **Airtable** or **PostgreSQL** credential
3. Choose table: **hardwareRequests**
4. Map fields to columns
5. Test the node

### For "Update Request in Database" node:
1. Click the node
2. Select same credential as above
3. Set **ID field**: requestId
4. Map fields to columns
5. Test the node

---

## 🔗 Step 5: Create Approval Form/Endpoint

You need a way to send approval responses. Options:

### Option A: Simple HTML Form
```html
<form action="YOUR_N8N_WEBHOOK_URL/manager-approval-webhook" method="POST">
  <input type="hidden" name="requestId" value="HW-1234567890">
  <input type="hidden" name="approved" value="true">
  <textarea name="approvalNotes" placeholder="Optional notes"></textarea>
  <button type="submit">Approve</button>
</form>
```

### Option B: Email Links
The workflow already includes approval links in emails. Configure your approval endpoint:

1. Create a simple web app (Node.js, Python, etc.)
2. When user clicks approval link, POST to:
   - `/webhook/manager-approval-webhook`
   - `/webhook/it-manager-approval-webhook`
3. Send this JSON:
```json
{
  "requestId": "HW-123456",
  "approved": true,
  "approvalNotes": "Looks good"
}
```

### Option C: Zapier/Make Integration
Forward email approvals to n8n webhooks automatically

---

## ✅ Step 6: Test the Workflow

### Test Data
Use this to trigger the workflow:

```json
{
  "employeeId": "EMP001",
  "employeeName": "John Smith",
  "employeeEmail": "john@company.com",
  "managerId": "MGR001",
  "managerEmail": "manager@company.com",
  "hardwareType": "Laptop",
  "description": "Dell XPS 13 for development",
  "urgency": "High",
  "estimatedBudget": 1500
}
```

### Testing Steps:
1. Go to **Workflows** → Your workflow
2. Click the initial webhook node
3. Click **Test workflow**
4. Paste the test data
5. Follow the flow and verify:
   - [ ] Manager receives email
   - [ ] Request appears in database
   - [ ] Status changes as approvals happen
   - [ ] IT member receives task
   - [ ] Emails are formatted correctly

---

## 🎯 Step 7: Activate & Monitor

1. Click **Activate** (top right)
2. Workflow is now live
3. Monitor executions:
   - Click **Executions** tab
   - See all workflow runs
   - Check logs for errors
   - View email deliveries

---

## 📊 Creating an Approval Frontend

### Simple HTML Dashboard

```html
<!DOCTYPE html>
<html>
<head>
  <title>Hardware Request Approvals</title>
  <style>
    body { font-family: Arial; margin: 20px; }
    .request { border: 1px solid #ddd; padding: 15px; margin: 10px 0; }
    button { padding: 10px 20px; margin: 5px; cursor: pointer; }
    .approve { background: #4CAF50; color: white; }
    .reject { background: #f44336; color: white; }
  </style>
</head>
<body>
  <h1>Hardware Requests Pending Approval</h1>
  <div id="requests"></div>

  <script>
    // Fetch pending requests from your database
    async function loadRequests() {
      const response = await fetch('/api/requests?status=Pending Manager Approval');
      const requests = await response.json();
      
      const html = requests.map(req => `
        <div class="request">
          <h3>${req.hardwareType}</h3>
          <p>Employee: ${req.employeeName}</p>
          <p>Reason: ${req.description}</p>
          <p>Budget: $${req.estimatedBudget}</p>
          
          <button class="approve" onclick="approve('${req.requestId}')">
            Approve
          </button>
          <button class="reject" onclick="reject('${req.requestId}')">
            Reject
          </button>
        </div>
      `).join('');
      
      document.getElementById('requests').innerHTML = html;
    }

    async function approve(requestId) {
      const notes = prompt('Approval notes (optional):');
      await fetch('YOUR_WEBHOOK_URL/manager-approval-webhook', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          requestId,
          approved: true,
          approvalNotes: notes || ''
        })
      });
      alert('Request approved!');
      loadRequests();
    }

    async function reject(requestId) {
      const reason = prompt('Rejection reason:');
      if (!reason) return;
      
      await fetch('YOUR_WEBHOOK_URL/manager-approval-webhook', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          requestId,
          approved: false,
          rejectionNotes: reason
        })
      });
      alert('Request rejected!');
      loadRequests();
    }

    loadRequests();
    // Refresh every 60 seconds
    setInterval(loadRequests, 60000);
  </script>
</body>
</html>
```

---

## 🔍 Monitoring Checklist

### Weekly Review:
- [ ] Check approval times (should be < 48 hours)
- [ ] Monitor rejection rate (should be < 20%)
- [ ] Review budget usage
- [ ] Confirm all emails are being sent

### Monthly Review:
- [ ] Generate approval reports
- [ ] Analyze hardware types requested
- [ ] Identify bottlenecks in approval chain
- [ ] Plan budget for next month

---

## 🐛 Common Issues & Fixes

| Issue | Solution |
|-------|----------|
| Emails not sending | Check email credentials, test SMTP connection |
| Database not updating | Verify credentials, ensure table exists with correct fields |
| Webhook not triggering | Confirm webhook URL is correct, check n8n logs |
| Approval not processing | Verify approval data format matches expected schema |
| Workflow not activating | Check all credentials are configured and saved |

---

## 📱 Mobile-Friendly Approval Form

```html
<div style="max-width: 500px; margin: 0 auto;">
  <h2>Approve Hardware Request</h2>
  
  <div style="background: #f5f5f5; padding: 15px; margin: 15px 0;">
    <p><strong>Request ID:</strong> HW-1234567890</p>
    <p><strong>Employee:</strong> John Smith</p>
    <p><strong>Hardware:</strong> Dell Laptop</p>
    <p><strong>Budget:</strong> $1,500</p>
  </div>

  <form onsubmit="submitApproval(event)">
    <textarea placeholder="Add approval notes..." style="width: 100%; height: 100px;"></textarea>
    
    <div style="margin-top: 15px;">
      <button type="submit" style="width: 100%; padding: 15px; background: #4CAF50; color: white; border: none; cursor: pointer;">
        ✓ Approve
      </button>
      <button type="button" onclick="submitReject()" style="width: 100%; padding: 15px; background: #f44336; color: white; border: none; cursor: pointer; margin-top: 10px;">
        ✗ Reject
      </button>
    </div>
  </form>
</div>

<script>
  function submitApproval(e) {
    e.preventDefault();
    const notes = e.target[0].value;
    sendApproval(true, notes);
  }

  function submitReject() {
    const reason = prompt('Please provide rejection reason:');
    if (reason) sendApproval(false, reason);
  }

  function sendApproval(approved, notes) {
    fetch('YOUR_WEBHOOK_URL/manager-approval-webhook', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        requestId: 'HW-1234567890',
        approved: approved,
        approvalNotes: notes
      })
    }).then(() => alert('Decision recorded!'));
  }
</script>
```

---

## 🎓 Next Steps

1. **Test** the workflow with sample data
2. **Customize** email templates with your branding
3. **Create** approval forms for managers
4. **Train** team on the new process
5. **Monitor** and optimize based on usage data
6. **Integrate** with your HRIS/IT systems

---

## 📞 Need Help?

- **n8n Docs**: https://docs.n8n.io
- **n8n Community**: https://community.n8n.io
- **Check Workflow Logs**: Executions tab shows all details
- **Test Individual Nodes**: Right-click any node → Test node

---

**Your hardware request workflow is ready to go! 🚀**
