# Hardware Request Approval Workflow - Complete Guide

## 📋 Overview

This n8n workflow automates the process of hardware requests from employees through multiple approval stages to final handover by IT operations.

### Workflow Stages
1. **Employee Request** → Employee submits hardware request
2. **Manager Approval** → Direct manager reviews and approves/rejects
3. **IT Manager Approval** → IT Operations Manager reviews budget and feasibility
4. **IT Handover** → Approved request assigned to IT team member for delivery

---

## 🔄 Workflow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                     EMPLOYEE SUBMITS REQUEST                         │
└─────────────────────────┬──────────────────────────────────────────┘
                          │
                          ▼
                ┌──────────────────────┐
                │  Initialize Request  │
                │  - Generate ID       │
                │  - Set Status        │
                │  - Create Timestamp  │
                └─────────┬────────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
                ▼                   ▼
        ┌──────────────┐    ┌──────────────┐
        │ Store in DB  │    │ Send to Mgr  │
        └──────────────┘    └──────┬───────┘
                                   │
                        ┌──────────▼──────────┐
                        │  MANAGER DECISION   │
                        └─────┬────────┬─────┘
                              │        │
                        Approved   Rejected
                              │        │
                              ▼        ▼
                    ┌──────────────┐  Notify Employee
                    │ Send to IT   │  (Rejection)
                    │ Manager      │
                    └──────┬───────┘
                           │
                    ┌──────▼──────┐
                    │ IT MANAGER  │
                    │  DECISION   │
                    └─────┬──┬────┘
                          │  │
                    Approved Rejected
                          │  │
                          ▼  └─→ Notify Employee
                    ┌──────────┐  (Rejection)
                    │  Assign  │
                    │ IT Member│
                    └────┬─────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    Send Task        Notify Employee   Update DB
    to IT Member     (Approved)
        │
        └─→ Mark as Completed
            Send Final Report
```

---

## 📝 Input Schema

The workflow expects the following data when triggered:

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

### Field Descriptions:
- **employeeId**: Unique identifier for the employee
- **employeeName**: Full name of requesting employee
- **employeeEmail**: Email for notifications
- **managerId**: Employee's direct manager ID
- **managerEmail**: Manager's email address
- **hardwareType**: Type of hardware (Laptop, Desktop, Monitor, etc.)
- **description**: Detailed description of hardware request
- **urgency**: Priority level (Low, Normal, High, Critical)
- **estimatedBudget**: Estimated cost of hardware

---

## 🔑 Key Nodes Explained

### 1. **Webhook - Request Received**
- **Purpose**: Entry point for hardware requests
- **Trigger**: Can be called from a form, API, or another system
- **Output**: Raw request data

### 2. **Initialize Request**
- **Purpose**: Structure the request data and create unique ID
- **Assigns**: 
  - Unique Request ID (format: HW-[timestamp])
  - Initial status: "Pending Manager Approval"
  - Empty approval objects for tracking

### 3. **Send to Manager for Approval**
- **Purpose**: Email manager with request details
- **Includes**: Interactive approval link
- **Email Contains**: Request ID, employee info, hardware details, budget

### 4. **Manager Approval Webhook**
- **Purpose**: Captures manager's approval/rejection decision
- **Expected Data**:
  ```json
  {
    "requestId": "HW-1234567890",
    "approved": true,
    "managerNotes": "Approved. Budget is within limits."
  }
  ```

### 5. **Check Manager Approval**
- **Purpose**: Routes workflow based on manager's decision
- **True Path**: Request approved → Forward to IT Manager
- **False Path**: Request rejected → Notify employee

### 6. **Update Request - Manager Approved**
- **Purpose**: Update request status and record manager approval
- **Status**: "Pending IT Manager Approval"
- **Records**: Approval date and notes

### 7. **Send to IT Manager for Approval**
- **Purpose**: Escalate to IT Operations Manager
- **Includes**: All previous approvals and comments
- **Budget Review**: IT Manager checks feasibility and budget

### 8. **IT Manager Approval Webhook**
- **Purpose**: Captures IT Manager's decision
- **Expected Data**:
  ```json
  {
    "requestId": "HW-1234567890",
    "approved": true,
    "approvalNotes": "Budget approved. Will order next week."
  }
  ```

### 9. **Assign to IT Operation Member**
- **Purpose**: Distribute workload among IT team
- **Logic**: Random assignment (can be customized for load balancing)
- **Assigns**: 
  - IT member from available pool
  - Sets status to "Assigned for Handover"

### 10. **Send Handover Task to IT Member**
- **Purpose**: Notify assigned IT member of the task
- **Includes**: Employee contact info, hardware details, urgency
- **Action**: IT member marks task as completed

### 11. **Notify Employee - Approved**
- **Purpose**: Inform employee of approval
- **Includes**: Assigned IT member contact info
- **Tone**: Celebratory and helpful

### 12. **Update Request in Database**
- **Purpose**: Store final request state
- **Updates**: All approval records, assignments, timestamps

### 13. **Send Completion Report**
- **Purpose**: Document handover completion
- **Recipients**: HR/Admin team
- **Records**: Who received what, when, and from whom

---

## 🔌 Integration Points

### Database Integration (Airtable/PostgreSQL)
The workflow stores requests in a database at two points:

**Store Request in Database:**
```
Base: hardwareRequests
Fields:
  - requestId (Primary)
  - employeeName
  - hardwareType
  - status
  - approvals (JSON)
  - assignedMember (JSON)
  - timestamps
```

**Update Request in Database:**
- Updates status throughout the workflow
- Records all approval dates and notes
- Tracks assignment details

### Email Integration
**Services Used:**
- Email Send node (built-in to n8n)
- Configuration: SMTP or Gmail API

**Required Setup:**
1. Configure email credentials in n8n
2. Update recipient email addresses:
   - Manager email (dynamic from form)
   - IT Operations Manager: `itoperationmanager@company.com`
   - IT Team members (in assignment logic)
   - HR/Admin: `hardwarerequests@company.com`

### Webhook Integration
**Three approval webhooks respond to:**

1. **Manager Approval**: `/manager-approval-webhook`
2. **IT Manager Approval**: `/it-manager-approval-webhook`
3. **Handover Completion**: `/handover-complete-webhook`

---

## 📊 Status Tracking

The workflow maintains these statuses:

| Status | Meaning | Next Step |
|--------|---------|-----------|
| Pending Manager Approval | Awaiting direct manager | Manager reviews |
| Pending IT Manager Approval | Manager approved, awaiting IT | IT Manager reviews |
| Assigned for Handover | IT Manager approved | IT member delivers |
| Rejected by Manager | Manager declined | Employee notified |
| Rejected by IT Manager | IT Manager declined | Employee notified |
| Completed | Hardware delivered | Final report sent |

---

## ⚙️ Configuration Steps

### Step 1: Set Up Email
1. Go to n8n → Credentials
2. Add Email Send credentials (SMTP or Gmail)
3. Test the connection

### Step 2: Configure Database
1. Choose database: Airtable or PostgreSQL
2. Create table: `hardwareRequests`
3. Add fields matching the schema above
4. Get API credentials

### Step 3: Update Email Recipients
Edit these nodes:
- **Send to Manager**: Uses dynamic email from request data
- **Send to IT Manager**: Update `itoperationmanager@company.com`
- **Send to IT Member**: Update IT team member emails
- **Send Completion Report**: Update `hardwarerequests@company.com`

### Step 4: Customize IT Team Members
Edit the **Assign to IT Operation Member** function node:

```javascript
const members = [
  { name: 'John Doe', email: 'john.doe@company.com' },
  { name: 'Jane Smith', email: 'jane.smith@company.com' },
  { name: 'Mike Johnson', email: 'mike.johnson@company.com' }
];
```

### Step 5: Create Approval Form
Create a webhook or form that collects approval responses:

```javascript
// Approval form should POST to:
POST /webhook/manager-approval-webhook

// With data:
{
  "requestId": "HW-1234567890",
  "approved": true/false,
  "approvalNotes": "Text notes"
}
```

### Step 6: Test the Workflow
1. Activate the workflow
2. Send test request to initial webhook
3. Follow the approval chain
4. Verify database updates
5. Check emails are sent

---

## 🔍 Testing Checklist

- [ ] Initial request webhook receives data correctly
- [ ] Manager email is sent with correct details
- [ ] Manager approval webhook processes response
- [ ] Request marked as approved in database
- [ ] IT Manager receives escalation email
- [ ] IT Manager approval updates database
- [ ] IT team member receives task assignment
- [ ] Employee receives approval notification
- [ ] Completion report sent to admin
- [ ] All statuses updated correctly
- [ ] Rejection path works (manager rejects)
- [ ] Rejection path works (IT Manager rejects)

---

## 🛠️ Customization Options

### Load Balancing for IT Members
Replace random assignment with round-robin:

```javascript
// Get assigned count for each member and assign to least busy
```

### Budget Approval Rules
Add automatic budget checking:

```javascript
const maxBudget = 2000;
if (estimatedBudget > maxBudget) {
  // Flag for higher-level approval
}
```

### Urgency Handling
Expedite high-urgency requests:

```javascript
if (urgency === 'Critical') {
  // Send to executive manager
  // Skip one approval level
}
```

### Hardware Catalog Integration
Connect to inventory system:

```javascript
// Fetch available hardware from inventory
// Auto-add specs to email
// Reserve item upon approval
```

### Slack Integration
Replace email with Slack notifications:

```javascript
// Add Slack node instead of Email Send
// Post to approval channels
// Get responses via Slack buttons
```

---

## 📈 Monitoring and Analytics

### Track These Metrics:
- Average approval time per manager
- Approval rate (approved vs rejected)
- Time to handover after approval
- Hardware type distribution
- Budget utilization

### Create Dashboard Query:
```sql
SELECT 
  status,
  COUNT(*) as count,
  AVG(DATEDIFF(day, requestDate, approvalDate)) as avg_days,
  SUM(estimatedBudget) as total_budget
FROM hardwareRequests
GROUP BY status
```

---

## 🔐 Security Considerations

1. **Sensitive Data**: Emails contain employee names and IDs
   - Use secure email transmission (TLS/SSL)
   - Encrypt database at rest

2. **Approval Links**: Include tokens in approval URLs
   - Generate unique token per request
   - Token expires after 7 days
   - Verify token before processing approval

3. **Database Access**: Restrict access by role
   - Managers: View only their team's requests
   - IT Managers: View all requests
   - IT Members: View assigned requests

4. **Audit Trail**: Log all approvals
   - Who approved
   - When approved
   - From which IP
   - Any changes made

---

## 🐛 Troubleshooting

### Issue: Emails not sending
- **Check**: Email credentials in n8n
- **Check**: SMTP settings
- **Check**: Email addresses are correct
- **Solution**: Test email send node separately

### Issue: Database connection failing
- **Check**: Database credentials
- **Check**: Network connectivity
- **Check**: Table exists and fields match
- **Solution**: Test database connection in credentials

### Issue: Webhook not triggering
- **Check**: Webhook URL is correct
- **Check**: POST data matches expected schema
- **Check**: Workflow is active
- **Solution**: Use n8n test data to simulate webhook

### Issue: Approval response not processing
- **Check**: Webhook URL in approval email is correct
- **Check**: Response JSON matches expected format
- **Check**: requestId matches
- **Solution**: Check n8n logs for error details

---

## 📞 Support & Contact

For issues or customizations:
- Review n8n documentation: https://docs.n8n.io
- Check webhook logs in n8n
- Enable execution history for debugging
- Use n8n community forum for help

---

## 📋 Summary

This workflow provides:
- ✅ Automated multi-level approval process
- ✅ Email notifications at each stage
- ✅ Database tracking and audit trail
- ✅ Flexible assignment of IT resources
- ✅ Clear status updates to all parties
- ✅ Rejection handling at each level
- ✅ Final handover documentation

The workflow is production-ready and can be customized to fit your organization's specific needs.
