# ⚡ Quick Reference - GPT Protection System

## 🔗 Important URLs

```
n8n Dashboard:
https://n8n.banirisset.com

Webhook Endpoints:
├─ Register: https://n8n.banirisset.com/webhook/register-gpt
├─ Verify:   https://n8n.banirisset.com/webhook/verify-access
└─ Schema:   https://n8n.banirisset.com/webhook/schema/{gpt_id}

Google Sheets:
https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit
```

---

## 🚀 Quick Actions

### Register New GPT

**cURL:**
```bash
curl -X POST https://n8n.banirisset.com/webhook/register-gpt \
  -H "Content-Type: application/json" \
  -d '{"gpt_name":"My GPT","gpt_url":"https://chatgpt.com/g/g-xxx","owner_email":"user@email.com"}'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "gpt_id": "gpt...",
    "api_key": "gpt_...",
    "schema_url": "https://...",
    "verify_url": "https://..."
  }
}
```

---

### Verify API Key

**cURL:**
```bash
curl -X POST https://n8n.banirisset.com/webhook/verify-access \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"action":"test"}'
```

**Success Response:**
```json
{
  "authorized": true,
  "gpt_id": "gpt...",
  "gpt_name": "...",
  "message": "Access granted"
}
```

**Error Response:**
```json
{
  "authorized": false,
  "error": "Invalid or inactive API key"
}
```

---

### Get OpenAPI Schema

**Browser atau cURL:**
```bash
curl https://n8n.banirisset.com/webhook/schema/gpt1737849600000
```

Copy hasil JSON → Paste ke CustomGPT Actions

---

## 🛠️ Common Tasks

### Task 1: Activate/Deactivate GPT

1. Open Google Sheets → GPT_Registry
2. Find GPT row
3. Change Status column (E):
   - `active` → GPT can be used
   - `inactive` → GPT blocked
   - `suspended` → Temporarily disabled

**Effect:** Immediate (next API call will check new status)

---

### Task 2: Regenerate API Key

**Manual method:**
1. Go to Google Sheets → GPT_Registry
2. Find GPT row
3. Delete old API key (column D)
4. Re-register GPT menggunakan webhook
5. Update API key di CustomGPT Actions

**Automated (future):**
Add workflow untuk regenerate

---

### Task 3: View Access Logs

1. Open Google Sheets → API_Logs
2. Sort by Timestamp (newest first)
3. Filter by:
   - GPT_ID → See logs for specific GPT
   - Status → See only success/failed
   - Date → See logs for specific period

---

### Task 4: Monitor Usage

**Quick Stats:**
1. Open Google Sheets → Analytics (jika sudah dibuat)
2. Lihat dashboard summary:
   - Total GPTs
   - API calls today/week/month
   - Success rate

**Manual Count:**
```
Total calls today:
=COUNTIF(API_Logs!A:A, ">="&TEXT(TODAY(),"YYYY-MM-DD"))

Calls for specific GPT:
=COUNTIF(API_Logs!B:B, "gpt1737849600000")
```

---

### Task 5: Add CustomGPT Protection

**Step-by-step:**

1. **Get GPT URL**
   ```
   Your GPT: https://chatgpt.com/g/g-abc123-your-gpt
   GPT ID: g-abc123-your-gpt
   ```

2. **Register**
   ```bash
   curl -X POST https://n8n.banirisset.com/webhook/register-gpt \
     -H "Content-Type: application/json" \
     -d '{
       "gpt_name":"Your GPT Name",
       "gpt_url":"https://chatgpt.com/g/g-abc123-your-gpt",
       "owner_email":"your@email.com"
     }'
   ```

3. **Copy dari response:**
   - `api_key`
   - `schema_url`

4. **Di CustomGPT Editor:**
   - Configure → Actions → Create new action
   - Import from URL → Paste schema_url
   - Authentication: API Key
   - Custom header: `X-API-Key`
   - Value: Paste api_key
   - Save

5. **Test:**
   - Click "Test" di action verifyAccess
   - Should return: `authorized: true`

6. **Update Instructions:**
   Add to GPT instructions:
   ```
   IMPORTANT: Before any action, call verifyAccess to check authorization.
   If authorized=false, inform user they need valid subscription.
   ```

7. **Done!** ✅

---

## 🔍 Troubleshooting Checklist

### Problem: Webhook not responding

**Check:**
- [ ] Workflow is **Active** (toggle di kanan atas workflow)
- [ ] Webhook nodes have **Production URL** (not test URL)
- [ ] URL correct: `https://n8n.banirisset.com/webhook/...`
- [ ] n8n instance is running
- [ ] No firewall blocking requests

**Fix:**
1. Go to n8n workflow
2. Click workflow name → Settings
3. Enable: "Execute workflow on production mode"
4. Save
5. Toggle workflow OFF then ON

---

### Problem: "Spreadsheet not found" error

**Check:**
- [ ] Google Sheets credentials authorized
- [ ] Spreadsheet ID correct in ALL Google Sheets nodes
- [ ] Sheet names match exactly:
  - `GPT_Registry` (not "Sheet1")
  - `API_Logs` (not "Sheet2")

**Fix:**
1. Open any Google Sheets node
2. Click credential → Edit
3. Re-authorize: "Connect my account"
4. Select correct Google account
5. Save
6. Update all nodes dengan credential yang sama

---

### Problem: API key not found in lookup

**Check:**
- [ ] API key typed correctly (case-sensitive)
- [ ] No extra spaces before/after key
- [ ] GPT registered in Google Sheets
- [ ] Status = "active"

**Debug:**
1. Open Google Sheets
2. Use Find (Ctrl+F)
3. Search for API key
4. Check Status column

---

### Problem: CustomGPT can't import schema

**Check:**
- [ ] Schema URL accessible (test in browser)
- [ ] Returns valid JSON
- [ ] No CORS errors

**Fix:**
1. Test schema URL di browser
2. Copy entire JSON response
3. Di CustomGPT:
   - Create new action
   - Schema → Manual
   - Paste JSON
   - Save

---

### Problem: n8n shows "Node execution error"

**Debug:**
1. Click workflow → Executions
2. Find failed execution (red X)
3. Click to expand
4. Check which node failed
5. Read error message

**Common errors:**
```
"Cannot read property..." → Data format issue
"Unauthorized" → Credentials expired
"Timeout" → Network issue / slow response
"Invalid JSON" → Check request body format
```

---

## 📊 Monitoring Commands

### Check if workflow is active

**In n8n:**
1. Go to Workflows
2. Find "GPT Protection System"
3. Check toggle switch → Should be **green/ON**

### Check recent executions

**In n8n:**
1. Click workflow
2. Executions tab (right side)
3. View list:
   - ✅ Green checkmark = Success
   - ❌ Red X = Failed
   - 🟡 Yellow = Running

### Check Google Sheets data

**Quick validation:**
```
Row count GPT_Registry: Should match # of registered GPTs
Row count API_Logs: Increases with each API call
Latest log timestamp: Should be recent if GPT actively used
```

---

## 🔐 Security Best Practices

### DO ✅
- Store API keys securely
- Use HTTPS only
- Hide API_Key column when sharing Sheets
- Regular audit of access logs
- Deactivate unused GPTs
- Use strong owner emails

### DON'T ❌
- Share API keys publicly
- Commit API keys to GitHub
- Use same API key for multiple GPTs
- Leave test GPTs active
- Ignore failed access attempts
- Give edit access to everyone

---

## 📱 Mobile Access

### View logs on mobile

**Google Sheets App:**
1. Install: Google Sheets app
2. Open: GPT Protection Database
3. Switch sheets using bottom tabs
4. Filter/Sort as needed

**Quick view:**
- Recent activity → API_Logs sheet
- GPT list → GPT_Registry sheet
- Stats → Analytics sheet

---

## 💾 Backup Strategy

### Daily auto-backup (recommended)

**Setup:**
1. Google Sheets → Add-ons
2. Install "Backup and Sync"
3. Schedule: Daily, 2 AM
4. Destination: Google Drive/Backups

### Manual backup

**Quick export:**
```
File → Download → 
  - Excel (.xlsx) → Full backup
  - CSV → Per sheet
  - PDF → Readable copy
```

**Frequency:**
- Weekly: Full backup
- Monthly: Archive old logs

---

## 🎯 Performance Tips

### Keep Sheets fast

- Archive logs older than 3 months
- Use filter views instead of hiding rows
- Limit active GPTs to actual usage
- Clean up test data regularly

### Optimize n8n

- Disable old test workflows
- Clear old execution logs (>1 month)
- Use production URLs, not test URLs
- Check workflow efficiency (execution time)

---

## 📞 Emergency Contacts

**If something breaks:**

1. **Check system status:**
   - n8n executions log
   - Google Sheets data integrity
   - Webhook response (Postman test)

2. **Quick fixes:**
   - Restart workflow (toggle OFF/ON)
   - Re-authorize Google Sheets
   - Clear browser cache (for dashboard)

3. **Rollback:**
   - Restore Google Sheets from backup
   - Re-import workflow from JSON file
   - Re-register GPTs if needed

---

## 🎓 Learning Resources

**n8n Documentation:**
https://docs.n8n.io

**Google Sheets Functions:**
https://support.google.com/docs/table/25273

**OpenAPI Specification:**
https://spec.openapis.org/oas/v3.1.0

**CustomGPT Actions:**
https://platform.openai.com/docs/actions

---

## 📝 Quick Notes Space

```
My n8n URL: _______________________________

My Spreadsheet ID: _______________________________

My first GPT:
  Name: _______________________________
  GPT ID: _______________________________
  API Key: _______________________________

Emergency backup date: _______________________________

Last audit date: _______________________________
```

---

**Print this for quick reference! 📄**