🔐 Tab "Proteksi URL" - Fitur Baru
Deskripsi Fitur
Tab "Proteksi URL" adalah gateway/proteksi untuk Custom GPT yang memerlukan user login sebelum mengakses API endpoint.

Cara Kerja:
User buka Custom GPT Action → Trigger API call

Redirect ke Protected URL (contoh: https://gpt.banirisset.com/protect/abc123def456)

Tampil Login Form → User login dengan email + password

Atau gunakan OAuth → Sign in with n8n.rinjanihuub.id

Verify Credentials → Check di database/backend

Generate Token → Buat unique token untuk sesi ini

Return Token ke GPT → GPT bisa pakai token untuk akses API

Implementasi di Dashboard
Tab: "Proteksi URL"
Section 1: URL Protection Configuration
text
- Input: Protected URL Name (contoh: "Threads Analyzer Protection")
- Input: Redirect URL (alamat URL proteksi, misal: gpt.banirisset.com/protect/[id])
- Input: Backend Endpoint untuk verify (misal: n8n.banirisset.com/api/verify-user)
- Toggle: Require Login (Yes/No)
- Toggle: Allow OAuth (Yes/No)
- Save button
Section 2: Login Gateway (Tampil untuk User)
text
┌─────────────────────────────────────────┐
│        🔐 Akses Terproteksi             │
│                                         │
│  Email/Username:                        │
│  [________________]                     │
│                                         │
│  Password:                              │
│  [________________]                     │
│                                         │
│  Token (optional):                      │
│  [________________]                     │
│                                         │
│  [   Login    ]                         │
│                                         │
│  ──────── atau ────────                 │
│                                         │
│  [ Sign in with n8n.rinjanihuub.id ] ← OAuth
│                                         │
│  Status: ⏳ Waiting...                  │
└─────────────────────────────────────────┘
Section 3: Token Verification
text
Input URL: [Enter protected URL to test]
[Test Access]

Result:
- Protected: ✅ Yes / ❌ No
- Status: Active/Inactive
- Login Required: Yes/No
- OAuth Enabled: Yes/No
- Last access: [timestamp]
Backend Implementation (Node.js)
javascript
// protected-url.js
const express = require('express');
const router = express.Router();
const jwt = require('jsonwebtoken');

// GET /protect/:urlId - Display login form
router.get('/protect/:urlId', async (req, res) => {
  const { urlId } = req.params;
  const redirectUrl = req.query.redirect_uri;
  
  // Check if URL is registered & protected
  const protection = await db.getProtection(urlId);
  
  if (!protection) {
    return res.status(404).json({ error: 'URL not found' });
  }
  
  // Render login form
  res.send(`
    <!DOCTYPE html>
    <html>
    <head><title>Login Required</title></head>
    <body>
      <h2>🔐 Akses Terproteksi</h2>
      <form method="POST" action="/protect/${urlId}/verify">
        <input type="email" name="email" placeholder="Email" required>
        <input type="password" name="password" placeholder="Password" required>
        <input type="hidden" name="redirect_uri" value="${redirectUrl}">
        <button type="submit">Login</button>
      </form>
      
      <hr>
      <button onclick="oauthLogin('${urlId}')">
        Sign in with n8n.rinjanihuub.id
      </button>
      
      <script>
        function oauthLogin(urlId) {
          const redirectUri = encodeURIComponent('https://gpt.banirisset.com/protect/' + urlId + '/callback');
          window.location = 'https://n8n.rinjanihuub.id/oauth/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=' + redirectUri;
        }
      </script>
    </body>
    </html>
  `);
});

// POST /protect/:urlId/verify - Verify login credentials
router.post('/protect/:urlId/verify', async (req, res) => {
  const { urlId } = req.params;
  const { email, password, redirect_uri } = req.body;
  
  // Verify email & password
  const user = await db.getUserByEmail(email);
  if (!user || user.password !== hashPassword(password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  // Generate token
  const token = jwt.sign(
    { 
      userId: user.id,
      email: user.email,
      urlId: urlId
    },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  );
  
  // Redirect kembali ke GPT dengan token
  const callbackUrl = `${redirect_uri}?token=${token}&status=success`;
  res.redirect(callbackUrl);
});

// GET /protect/:urlId/callback - OAuth callback
router.get('/protect/:urlId/callback', async (req, res) => {
  const { code, state } = req.query;
  
  // Exchange code for access token dari n8n OAuth
  const accessToken = await exchangeOAuthCode(code);
  
  // Get user info
  const userInfo = await getUserInfoFromOAuth(accessToken);
  
  // Generate token untuk GPT
  const gptToken = jwt.sign(
    {
      userId: userInfo.id,
      email: userInfo.email,
      source: 'oauth',
      urlId: req.params.urlId
    },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  );
  
  res.redirect(`/protect/${req.params.urlId}/success?token=${gptToken}`);
});

// GET /protect/:urlId/verify-token - API endpoint verify token
router.post('/protect/:urlId/verify-token', async (req, res) => {
  const { token } = req.body;
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    res.json({ valid: true, user: decoded });
  } catch(error) {
    res.status(401).json({ valid: false, error: 'Invalid token' });
  }
});

module.exports = router;
Environment Variables
bash
# .env
JWT_SECRET=your_super_secret_key_here
OAUTH_CLIENT_ID=your_n8n_client_id
OAUTH_CLIENT_SECRET=your_n8n_client_secret
OAUTH_REDIRECT_URI=https://gpt.banirisset.com/protect/callback
DATABASE_URL=your_database_connection
Database Schema
sql
-- Protected URLs Table
CREATE TABLE protected_urls (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  url_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  redirect_url TEXT,
  backend_endpoint TEXT,
  require_login BOOLEAN DEFAULT true,
  allow_oauth BOOLEAN DEFAULT true,
  oauth_provider TEXT DEFAULT 'n8n',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Users Table (untuk login)
CREATE TABLE gpt_users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  name TEXT,
  oauth_id TEXT,
  oauth_provider TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Access Logs Table
CREATE TABLE access_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  protected_url_id UUID REFERENCES protected_urls(id),
  user_id UUID REFERENCES gpt_users(id),
  token TEXT,
  access_method TEXT, -- 'password' | 'oauth'
  ip_address TEXT,
  status TEXT, -- 'success' | 'failed'
  created_at TIMESTAMP DEFAULT NOW()
);
Frontend HTML (Login Page Component)
xml
<div class="protected-url-form">
  <div class="form-section">
    <h3>🔐 Proteksi URL dengan Login Gate</h3>
    
    <!-- Section 1: Setup -->
    <div class="form-group">
      <label>Nama Proteksi</label>
      <input type="text" id="protectionName" placeholder="Threads Analyzer Protection">
    </div>
    
    <div class="form-group">
      <label>URL yang akan diproteksi</label>
      <input type="url" id="protectionUrl" placeholder="https://gpt.banirisset.com/action/threads">
    </div>
    
    <div class="form-group">
      <label>Backend Verification Endpoint</label>
      <input type="url" id="verifyEndpoint" placeholder="https://n8n.banirisset.com/webhook/verify-user">
    </div>
    
    <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 16px;">
      <div>
        <label>
          <input type="checkbox" id="requireLogin" checked> Butuh Login
        </label>
      </div>
      <div>
        <label>
          <input type="checkbox" id="allowOAuth" checked> Allow OAuth
        </label>
      </div>
    </div>
    
    <button class="btn btn-primary" onclick="setupProtection()">💾 Setup Proteksi</button>
  </div>
  
  <!-- Section 2: Test Login -->
  <div class="form-section">
    <h3>🧪 Test Login Gateway</h3>
    
    <div class="info-box">
      💡 Simulasi user access ke protected URL
    </div>
    
    <div class="form-group">
      <label>Email Test</label>
      <input type="email" id="testEmail" placeholder="test@example.com">
    </div>
    
    <div class="form-group">
      <label>Password Test</label>
      <input type="password" id="testPassword" placeholder="password123">
    </div>
    
    <button class="btn btn-secondary" onclick="testLoginFlow()">Test Login Flow</button>
    
    <div id="testResult" style="margin-top: 16px;"></div>
  </div>
  
  <!-- Section 3: Active Protections -->
  <div class="form-section">
    <h3>✅ Proteksi URL Aktif</h3>
    
    <table>
      <thead>
        <tr>
          <th>Nama</th>
          <th>URL</th>
          <th>Login</th>
          <th>OAuth</th>
          <th>Status</th>
          <th>Aksi</th>
        </tr>
      </thead>
      <tbody id="protectionTable">
        <!-- Filled by JavaScript -->
      </tbody>
    </table>
  </div>
</div>
JavaScript Functions
javascript
// Setup Protection
async function setupProtection() {
  const name = document.getElementById('protectionName').value;
  const url = document.getElementById('protectionUrl').value;
  const endpoint = document.getElementById('verifyEndpoint').value;
  const requireLogin = document.getElementById('requireLogin').checked;
  const allowOAuth = document.getElementById('allowOAuth').checked;
  
  if (!name || !url || !endpoint) {
    alert('⚠️ Semua field harus diisi!');
    return;
  }
  
  const protection = {
    id: Date.now(),
    name,
    url,
    endpoint,
    requireLogin,
    allowOAuth,
    createdAt: new Date().toLocaleString('id-ID'),
    status: 'active'
  };
  
  let protections = JSON.parse(localStorage.getItem('urlProtections') || '[]');
  protections.push(protection);
  localStorage.setItem('urlProtections', JSON.stringify(protections));
  
  alert('✅ Proteksi URL berhasil dibuat!');
  loadProtections();
}

// Test Login Flow
async function testLoginFlow() {
  const email = document.getElementById('testEmail').value;
  const password = document.getElementById('testPassword').value;
  
  const result = document.getElementById('testResult');
  result.innerHTML = `<div class="alert alert-success">⏳ Testing...</div>`;
  
  // Simulasi
  setTimeout(() => {
    const token = `gpt_token_${Date.now()}`;
    result.innerHTML = `
      <div class="alert alert-success">
        ✅ Login Berhasil!<br>
        Token: <strong>${token}</strong>
      </div>
    `;
  }, 1000);
}

// Load Protections
function loadProtections() {
  const protections = JSON.parse(localStorage.getItem('urlProtections') || '[]');
  const table = document.getElementById('protectionTable');
  
  if (protections.length === 0) {
    table.innerHTML = '<tr><td colspan="6">Belum ada proteksi URL</td></tr>';
    return;
  }
  
  table.innerHTML = protections.map(p => `
    <tr>
      <td>${p.name}</td>
      <td style="font-size: 11px;">${p.url.substring(0, 30)}...</td>
      <td>${p.requireLogin ? '✅' : '❌'}</td>
      <td>${p.allowOAuth ? '✅' : '❌'}</td>
      <td><span class="status-badge status-active">Active</span></td>
      <td>
        <button class="btn btn-copy" onclick="deleteProtection(${p.id})">Delete</button>
      </td>
    </tr>
  `).join('');
}

// Delete Protection
function deleteProtection(id) {
  if (!confirm('Delete?')) return;
  
  let protections = JSON.parse(localStorage.getItem('urlProtections') || '[]');
  protections = protections.filter(p => p.id !== id);
  localStorage.setItem('urlProtections', JSON.stringify(protections));
  
  loadProtections();
}
Integrasi dengan N8N
Buat workflow di N8N untuk verify user:

text
Trigger: Webhook (POST /verify-user)
  ↓
Parse JSON (email, password)
  ↓
Database Query: SELECT * FROM gpt_users WHERE email = {{ email }}
  ↓
Compare password (hash)
  ↓
IF match:
  → Generate JWT token
  → Log access
  → Return { token, status: 'success' }
ELSE:
  → Return { status: 'failed', error: 'Invalid credentials' }
Security Best Practices
✅ Always use HTTPS
✅ Hash passwords dengan bcrypt
✅ Validate JWT token signature
✅ Set token expiration (1 hour)
✅ Log all access attempts
✅ Rate limit login attempts
✅ Use secure cookies untuk session
✅ Implement CSRF protection
✅ Never expose JWT secret
✅ Validate redirect_uri parameter