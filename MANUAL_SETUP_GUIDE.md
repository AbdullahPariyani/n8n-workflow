# Hardware Request Workflow - Manual Setup (If Import Fails)

If you're having issues importing the JSON, follow these steps to build the workflow manually in n8n.

---

## 📋 Manual Workflow Setup

### Step 1: Create the Initial Webhook

1. In n8n, click **+ Add Node**
2. Search for and select **Webhook**
3. Click on the Webhook node and configure:
   - **Method**: POST
   - **Path**: `hardware-request`
   - **Response Mode**: "On Received"
4. Click **Test** and copy the full webhook URL for later
5. Name it "Request Received"

---

### Step 2: Parse the Request Data

1. Click **+ Add Node**
2. Search for **Set** node
3. Add a Set node after the Webhook
4. Click **Add Assignment** and fill in these fields:

| Name | Value |
|------|-------|
| requestId | `='HW-' + $now.getTime()` |
| employeeId | `={{ $json.body.employeeId }}` |
| employeeName | `={{ $json.body.employeeName }}` |
| employeeEmail | `={{ $json.body.employeeEmail }}` |
| managerId | `={{ $json.body.managerId }}` |
| managerEmail | `={{ $json.body.managerEmail }}` |
| hardwareType | `={{ $json.body.hardwareType }}` |
| description | `={{ $json.body.description }}` |
| urgency | `={{ $json.body.urgency \|\| 'Normal' }}` |
| estimatedBudget | `={{ $json.body.estimatedBudget }}` |
| status | `='Pending Manager Approval'` |

5. Name it "Initialize Request"

---

### Step 3: Send Email to Manager

1. Click **+ Add Node**
2. Search for **Email Send**
3. Configure the Email node:
   - **From Email**: Use your configured email sender
   - **To Email**: `={{ $json.managerEmail }}`
   - **Subject**: `Hardware Request Approval Required - {{ $json.requestId }}`
   - **Email Type**: HTML
   - **HTML Message**: (paste the HTML template below)

**HTML Template for Manager Email:**
```html
<html><body style='font-family: Arial, sans-serif; max-width: 600px;'>
<div style='background-color: #f5f5f5; padding: 20px; border-radius: 5px;'>
  <h2 style='color: #333;'>Hardware Request Approval</h2>
  <p>Dear Manager,</p>
  
  <p>You have a new hardware request to approve from <strong>{{ $json.employeeName }}</strong>:</p>
  
  <table style='border-collapse: collapse; width: 100%; margin: 20px 0; background: white;'>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Request ID:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.requestId }}</td>
    </tr>
    <tr>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Employee:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.employeeName }}</td>
    </tr>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Hardware:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.hardwareType }}</td>
    </tr>
    <tr>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Description:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.description }}</td>
    </tr>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Budget:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>${{ $json.estimatedBudget }}</td>
    </tr>
  </table>
  
  <p>Please log in to the approval system to review and approve/reject this request.</p>
  <p style='color: #666; font-size: 12px; margin-top: 20px;'>Best regards,<br>Hardware Request System</p>
</div>
</body></html>
```

4. Name it "Send to Manager"

---

### Step 4: Create Approval Webhook

1. Click **+ Add Node**
2. Search for **Webhook**
3. Configure:
   - **Method**: POST
   - **Path**: `manager-approval`
   - **Response Mode**: "On Received"
4. Copy this webhook URL
5. Name it "Manager Approval Response"

---

### Step 5: Check Manager's Decision

1. Click **+ Add Node**
2. Search for **If** node
3. Configure the condition:
   - **Condition**: String
   - **Value 1**: `={{ $json.body.approved }}`
   - **Operation**: equals
   - **Value 2**: `true`
4. Name it "Manager Approved?"

This will create two paths:
- **True path** (top): If manager approved
- **False path** (bottom): If manager rejected

---

### Step 6A: Manager Approved Path

**Add this Set node on the TRUE path:**

1. Click **+ Add Node** on the TRUE path
2. Add **Set** node
3. Add assignments:

| Name | Value |
|------|-------|
| status | `='Pending IT Manager Approval'` |
| managerApproved | `={{ true }}` |
| managerApprovalDate | `={{ $now.toISOString() }}` |
| managerApprovalNotes | `={{ $json.body.approvalNotes \|\| '' }}` |

4. Name it "Update - Manager Approved"

---

### Step 6B: Manager Rejected Path

**Add this Set node on the FALSE path:**

1. Click **+ Add Node** on the FALSE path
2. Add **Set** node
3. Add assignments:

| Name | Value |
|------|-------|
| status | `='Rejected by Manager'` |
| managerApproved | `={{ false }}` |
| rejectionNotes | `={{ $json.body.rejectionNotes \|\| 'No reason provided' }}` |

4. Name it "Update - Manager Rejected"

---

### Step 7: Send Rejection Email to Employee

1. On the FALSE path, click **+ Add Node**
2. Add **Email Send**
3. Configure:
   - **To Email**: `={{ $json.employeeEmail }}`
   - **Subject**: `Hardware Request Rejected - {{ $json.requestId }}`
   - **HTML Message**:
```html
<html><body style='font-family: Arial, sans-serif; max-width: 600px;'>
<div style='background-color: #f5f5f5; padding: 20px; border-radius: 5px;'>
  <h2 style='color: #d32f2f;'>Request Status Update</h2>
  <p>Dear {{ $json.employeeName }},</p>
  
  <p>Your hardware request <strong>{{ $json.requestId }}</strong> has been <strong style='color: #d32f2f;'>REJECTED</strong> by your manager.</p>
  
  <p><strong>Reason:</strong> {{ $json.rejectionNotes }}</p>
  
  <p>Please contact your manager to discuss this decision.</p>
  
  <p style='color: #666; font-size: 12px; margin-top: 20px;'>Best regards,<br>Hardware Request System</p>
</div>
</body></html>
```
4. Name it "Reject Notification"

---

### Step 8: Send to IT Manager (Approval Path)

1. On the TRUE path from "Update - Manager Approved", click **+ Add Node**
2. Add **Email Send**
3. Configure:
   - **To Email**: `itoperationmanager@company.com`
   - **Subject**: `Hardware Request - {{ $json.requestId }} - Manager Approved`
   - **HTML Message**:
```html
<html><body style='font-family: Arial, sans-serif; max-width: 600px;'>
<div style='background-color: #f5f5f5; padding: 20px; border-radius: 5px;'>
  <h2 style='color: #333;'>Hardware Request - IT Manager Approval Required</h2>
  <p>Dear IT Operations Manager,</p>
  
  <p>A hardware request has been approved by the employee's manager:</p>
  
  <table style='border-collapse: collapse; width: 100%; margin: 20px 0; background: white;'>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Request ID:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.requestId }}</td>
    </tr>
    <tr>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Employee:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.employeeName }}</td>
    </tr>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Hardware:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.hardwareType }}</td>
    </tr>
    <tr>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Budget:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>${{ $json.estimatedBudget }}</td>
    </tr>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Manager Notes:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.managerApprovalNotes }}</td>
    </tr>
  </table>
  
  <p>Please log in to the approval system to review and approve/reject this request.</p>
  <p style='color: #666; font-size: 12px; margin-top: 20px;'>Best regards,<br>Hardware Request System</p>
</div>
</body></html>
```
4. Name it "Send to IT Manager"

---

### Step 9: IT Manager Approval Webhook

1. Click **+ Add Node**
2. Add **Webhook**
3. Configure:
   - **Method**: POST
   - **Path**: `it-manager-approval`
   - **Response Mode**: "On Received"
4. Name it "IT Manager Approval Response"

---

### Step 10: Check IT Manager's Decision

1. Click **+ Add Node**
2. Add **If** node
3. Configure:
   - **Condition**: String
   - **Value 1**: `={{ $json.body.approved }}`
   - **Operation**: equals
   - **Value 2**: `true`
4. Name it "IT Manager Approved?"

---

### Step 11A: IT Manager Approved (TRUE Path)

1. On TRUE path, add **Set** node
2. Add assignments:

| Name | Value |
|------|-------|
| status | `='Assigned for Handover'` |
| itManagerApproved | `={{ true }}` |
| itManagerApprovalDate | `={{ $now.toISOString() }}` |
| assignedMember | `={{ {'name': 'IT Support Team', 'email': 'itsupport@company.com'} }}` |

3. Name it "Update - IT Approved"

---

### Step 11B: IT Manager Rejected (FALSE Path)

1. On FALSE path, add **Set** node
2. Add assignments:

| Name | Value |
|------|-------|
| status | `='Rejected by IT Manager'` |
| itManagerApproved | `={{ false }}` |
| rejectionNotes | `={{ $json.body.rejectionNotes \|\| 'No reason provided' }}` |

3. Name it "Update - IT Rejected"

---

### Step 12: Send to IT Team Member

From "Update - IT Approved" node, add **Email Send**:
- **To Email**: `itsupport@company.com`
- **Subject**: `Hardware Handover Task - {{ $json.requestId }}`
- **HTML Message**:
```html
<html><body style='font-family: Arial, sans-serif; max-width: 600px;'>
<div style='background-color: #f5f5f5; padding: 20px; border-radius: 5px;'>
  <h2 style='color: #333;'>Hardware Handover Assignment</h2>
  <p>Dear IT Operations Team,</p>
  
  <p>You have been assigned a hardware handover task:</p>
  
  <table style='border-collapse: collapse; width: 100%; margin: 20px 0; background: white;'>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Request ID:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.requestId }}</td>
    </tr>
    <tr>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Employee:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.employeeName }} ({{ $json.employeeEmail }})</td>
    </tr>
    <tr style='background-color: #f2f2f2;'>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Hardware:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.hardwareType }}</td>
    </tr>
    <tr>
      <td style='border: 1px solid #ddd; padding: 12px; font-weight: bold;'>Urgency:</td>
      <td style='border: 1px solid #ddd; padding: 12px;'>{{ $json.urgency }}</td>
    </tr>
  </table>
  
  <p>Please coordinate with the employee for hardware delivery and configuration.</p>
  <p style='color: #666; font-size: 12px; margin-top: 20px;'>Best regards,<br>Hardware Request System</p>
</div>
</body></html>
```
4. Name it "Send Handover Task"

---

### Step 13: Notify Employee of Approval

From "Update - IT Approved" node, add another **Email Send**:
- **To Email**: `={{ $json.employeeEmail }}`
- **Subject**: `Hardware Request Approved! - {{ $json.requestId }}`
- **HTML Message**:
```html
<html><body style='font-family: Arial, sans-serif; max-width: 600px;'>
<div style='background-color: #f5f5f5; padding: 20px; border-radius: 5px;'>
  <h2 style='color: #4CAF50;'>✓ Request Approved!</h2>
  <p>Dear {{ $json.employeeName }},</p>
  
  <p>Great news! Your hardware request <strong>{{ $json.requestId }}</strong> has been <strong style='color: #4CAF50;'>APPROVED</strong>.</p>
  
  <p><strong>Hardware Type:</strong> {{ $json.hardwareType }}</p>
  <p>The IT Operations team will contact you shortly to arrange delivery and setup.</p>
  
  <p style='color: #666; font-size: 12px; margin-top: 20px;'>Best regards,<br>Hardware Request System</p>
</div>
</body></html>
```
5. Name it "Employee Approval Notification"

---

### Step 14: Send IT Manager Rejection

From "Update - IT Rejected" node, add **Email Send**:
- **To Email**: `={{ $json.employeeEmail }}`
- **Subject**: `Hardware Request Rejected - {{ $json.requestId }}`
- **HTML Message**: (Same as Step 7 rejection email)
4. Name it "Employee Rejection - IT"

---

## 🧪 Testing Your Workflow

### Test Request Data

Send this JSON to your webhook:

```json
{
  "employeeId": "EMP001",
  "employeeName": "John Smith",
  "employeeEmail": "john.smith@company.com",
  "managerId": "MGR001",
  "managerEmail": "manager.email@company.com",
  "hardwareType": "Laptop",
  "description": "Dell XPS 13 for development work",
  "urgency": "High",
  "estimatedBudget": 1500
}
```

### Manager Approval Test

Send this to the manager-approval webhook:

```json
{
  "approved": true,
  "approvalNotes": "Budget is within limits. Approved!"
}
```

### IT Manager Approval Test

Send this to the it-manager-approval webhook:

```json
{
  "approved": true,
  "approvalNotes": "Will order from vendor. ETA 5 days."
}
```

---

## ⚙️ Configuration Checklist

- [ ] Configure Email credentials in n8n
- [ ] Update `itoperationmanager@company.com` with real email
- [ ] Update `itsupport@company.com` with IT team email
- [ ] Test webhook to ensure it works
- [ ] Test all email nodes individually
- [ ] Verify all connections between nodes
- [ ] Activate the workflow
- [ ] Send test request through the complete flow

---

## 🔗 Webhook URLs Reference

Keep these URLs handy:

| Webhook | Path | Used For |
|---------|------|----------|
| Request Webhook | `hardware-request` | Employee submits request |
| Manager Approval | `manager-approval` | Manager approves/rejects |
| IT Manager Approval | `it-manager-approval` | IT Manager approves/rejects |

---

## 📊 Next Steps

1. Once the workflow is working, add database integration (Airtable/PostgreSQL)
2. Create an approval form for managers
3. Add approval deadline reminders
4. Create a dashboard to track requests
5. Set up analytics for approval metrics

---

## 🆘 Troubleshooting

**Issue**: Email not sending
- Check email credentials are configured
- Verify recipient email is correct
- Test the email node separately

**Issue**: Webhook not triggering
- Verify webhook URL is correct
- Check that workflow is activated
- Look at execution logs for errors

**Issue**: Data not passing between nodes
- Use the Test button on each node
- Check variable names match exactly
- Verify expressions use correct syntax

---

This manual setup gives you full control and helps you understand how the workflow functions!
