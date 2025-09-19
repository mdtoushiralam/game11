// PROJECT: TSR Gaming Wallet (Full Starter)
// This document contains a minimal but functional starter for a production-oriented
// gaming-wallet website: a React frontend + Node.js/Express backend with SQLite for storage.
// It includes: Google sign-in (server-verify), deposit/withdraw endpoints, UPI linking,
// account verification simulation, transaction history, and admin basics.
// IMPORTANT: This starter is FOR DEVELOPMENT / TESTING ONLY. Do NOT use for real money
// without legal compliance, secure payment gateway setup, KYC, audits, and encryption.

---

# 1) Quick overview (Hindi)

Ye starter project tumhe ek complete front+back solution deta hai jisse tum local pe run karke
features test kar sako: Gmail login (token server-side verify), deposits/withdraws with limits,
UPI linking, account verify (simulated), transaction history, and a small admin route.

Main cheezen jo tumhe apne environment me set karni hongi (env vars):
- GOOGLE_CLIENT_ID (frontend button) — optional for demo
- GOOGLE_CLIENT_ID_SERVER (same as above) — used server-side to verify idTokens
- JWT_SECRET — server-side JWT signing secret
- RAZORPAY_KEY_ID, RAZORPAY_KEY_SECRET — agar Razorpay use karoge

---

# 2) How to run (summary)

1. Clone project files into two folders: `/frontend` and `/backend`.
2. In backend: `npm install`, set `.env`, then `node index.js` (or `nodemon`).
3. In frontend: `npm install`, `npm run dev` (Vite) or `npm start` (CRA).
4. Open frontend URL (usually http://localhost:5173) and test.

---

# 3) Backend (Node.js + Express + SQLite) — file: backend/index.js

```js
// backend/index.js
require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const jwt = require('jsonwebtoken');
const sqlite3 = require('sqlite3').verbose();
const {OAuth2Client} = require('google-auth-library');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const JWT_SECRET = process.env.JWT_SECRET || 'devsecret';
const GOOGLE_CLIENT_ID_SERVER = process.env.GOOGLE_CLIENT_ID_SERVER || 'your-google-client-id';
const googleClient = new OAuth2Client(GOOGLE_CLIENT_ID_SERVER);

// simple sqlite db
const db = new sqlite3.Database('./tsr.db');
// init
db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, name TEXT, verified INTEGER DEFAULT 0, upi TEXT, balance INTEGER DEFAULT 0)`);
  db.run(`CREATE TABLE IF NOT EXISTS tx (id INTEGER PRIMARY KEY, user_id INTEGER, type TEXT, amount INTEGER, status TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP)`);
});

// helper
function signJwt(payload) {
  return jwt.sign(payload, JWT_SECRET, {expiresIn: '7d'});
}

async function verifyGoogleToken(idToken) {
  const ticket = await googleClient.verifyIdToken({idToken, audience: GOOGLE_CLIENT_ID_SERVER});
  const payload = ticket.getPayload();
  return payload; // contains email, name, sub
}

// routes
app.post('/api/auth/google', async (req, res) => {
  try {
    const {idToken} = req.body;
    const payload = await verifyGoogleToken(idToken);
    const email = payload.email;
    const name = payload.name || '';

    db.serialize(() => {
      db.run(`INSERT OR IGNORE INTO users (email, name, balance) VALUES (?, ?, ?)` ,[email, name, 0]);
      db.get(`SELECT * FROM users WHERE email = ?`, [email], (err, row) => {
        if (err) return res.status(500).json({error: 'DB error'});
        const token = signJwt({userId: row.id});
        res.json({jwt: token, user: row});
      });
    });
  } catch (e) {
    console.error(e);
    res.status(400).json({error: 'Invalid Google token'});
  }
});

// auth middleware
function auth(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader) return res.status(401).json({error: 'Missing auth'});
  const token = authHeader.split(' ')[1];
  try {
    const payload = jwt.verify(token, JWT_SECRET);
    req.userId = payload.userId;
    next();
  } catch (e) {
    res.status(401).json({error: 'Invalid token'});
  }
}

// deposit (simulate payment gateway callback or create tx)
app.post('/api/deposit', auth, (req, res) => {
  const {amount} = req.body;
  const MIN = 100, MAX = 2000;
  const userId = req.userId;
  if (!amount || amount < MIN || amount > MAX) return res.status(400).json({error: 'Invalid amount'});

  // create transaction pending
  db.run(`INSERT INTO tx (user_id, type, amount, status) VALUES (?, 'deposit', ?, 'success')`, [userId, amount], function(err) {
    if (err) return res.status(500).json({error: 'DB error'});
    // update user balance
    db.run(`UPDATE users SET balance = balance + ? WHERE id = ?`, [amount, userId], function(err2) {
      if (err2) return res.status(500).json({error: 'DB error2'});
      db.get(`SELECT * FROM users WHERE id = ?`, [userId], (e, row) => res.json({ok:true, balance: row.balance}));
    });
  });
});

// withdraw
app.post('/api/withdraw', auth, (req, res) => {
  const {amount} = req.body;
  const MIN = 230;
  if (!amount || amount < MIN) return res.status(400).json({error: 'Invalid amount'});
  const userId = req.userId;
  db.get(`SELECT balance FROM users WHERE id = ?`, [userId], (err, row) => {
    if (err) return res.status(500).json({error:'DB error'});
    if (!row || row.balance < amount) return res.status(400).json({error:'Insufficient balance'});
    // create tx
    db.run(`INSERT INTO tx (user_id, type, amount, status) VALUES (?, 'withdraw', ?, 'success')`, [userId, amount], function(err2) {
      if (err2) return res.status(500).json({error:'DB error2'});
      db.run(`UPDATE users SET balance = balance - ? WHERE id = ?`, [amount, userId], function(err3) {
        if (err3) return res.status(500).json({error:'DB error3'});
        db.get(`SELECT balance FROM users WHERE id = ?`, [userId], (e, r) => res.json({ok:true, balance: r.balance}));
      });
    });
  });
});

// link upi
app.post('/api/link-upi', auth, (req, res) => {
  const {upi} = req.body;
  if (!upi) return res.status(400).json({error:'Missing upi'});
  // very simple validation
  if (!/@/.test(upi) && !/^[0-9]{6,14}$/.test(upi)) return res.status(400).json({error:'UPI format invalid'});
  db.run(`UPDATE users SET upi = ? WHERE id = ?`, [upi, req.userId], function(err) {
    if (err) return res.status(500).json({error:'DB error'});
    res.json({ok:true, upi});
  });
});

// verify account (simulated)
app.post('/api/verify', auth, (req, res) => {
  db.run(`UPDATE users SET verified = 1 WHERE id = ?`, [req.userId], function(err) {
    if (err) return res.status(500).json({error:'DB error'});
    res.json({ok:true, verified:1});
  });
});

// get profile
app.get('/api/profile', auth, (req, res) => {
  db.get(`SELECT id, email, name, verified, upi, balance FROM users WHERE id = ?`, [req.userId], (err, row) => {
    if (err) return res.status(500).json({error:'DB error'});
    res.json({user: row});
  });
});

// tx history
app.get('/api/tx', auth, (req, res) => {
  db.all(`SELECT id, type, amount, status, created_at FROM tx WHERE user_id = ? ORDER BY id DESC LIMIT 200`, [req.userId], (err, rows) => {
    if (err) return res.status(500).json({error:'DB error'});
    res.json({tx: rows});
  });
});

// admin route (very simple)
app.get('/admin/users', (req, res) => {
  // NOTE: protect this in production
  db.all(`SELECT id, email, name, verified, upi, balance FROM users`, [], (err, rows) => {
    if (err) return res.status(500).json({error:'DB error'});
    res.json({users: rows});
  });
});

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log('Backend running on', PORT));
```

---

# 4) Frontend (React + Vite) — file: frontend/src/main.jsx

```jsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')).render(<App />)
```

---

# frontend/src/App.jsx (includes Google Sign-In + calls to backend)

```jsx
import React, {useEffect, useState} from 'react';
import GamingSite from './GamingSite';

export default function App(){
  return <GamingSite />;
}
```

---

# frontend/src/GamingSite.jsx (updated to call backend API)

```jsx
import React, {useEffect, useState} from 'react';

const API = process.env.REACT_APP_API || 'http://localhost:4000';
const GOOGLE_CLIENT_ID = process.env.REACT_APP_GOOGLE_CLIENT_ID || '';

export default function GamingSite(){
  const [jwt, setJwt] = useState(localStorage.getItem('jwt'));
  const [user, setUser] = useState(null);
  const [balance, setBalance] = useState(0);
  const [message, setMessage] = useState('');

  useEffect(()=>{ if (jwt) fetchProfile(jwt); }, [jwt]);

  async function fetchProfile(token){
    const r = await fetch(API + '/api/profile',{headers:{Authorization:'Bearer '+token}});
    if (r.ok){ const data = await r.json(); setUser(data.user); setBalance(data.user.balance); localStorage.setItem('jwt', token);} else {setMessage('Session expired'); setJwt(null); localStorage.removeItem('jwt');}
  }

  // Demo: use Google Identity Services to get id_token, then send to backend
  async function handleGoogleSignInDemo(){
    // In production: use google.accounts.id.prompt and the real client id to get id_token.
    // Here we ASK the user to paste an idToken or simulate by calling a dev-only endpoint.
    const idToken = prompt('Paste Google ID token here (for dev) or leave empty to simulate');
    if (!idToken){ // simulate
      // call backend with an invalid token - backend will reject; for dev use a special route or use OAuth flow.
      alert('Please configure Google ID token for realistic flow. Using simulated demo login.');
      // for demo, call a dev server endpoint that creates a demo user - but our backend expects google token.
      // We'll instead simulate by asking backend to create a demo user via a debug route (not included). For now, skip.
      return;
    }
    const res = await fetch(API + '/api/auth/google', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({idToken})});
    if (!res.ok) return setMessage('Login failed');
    const data = await res.json();
    setJwt(data.jwt); localStorage.setItem('jwt', data.jwt); setUser(data.user); setBalance(data.user.balance);
  }

  async function deposit(amount){
    const r = await fetch(API + '/api/deposit', {method:'POST', headers:{'Content-Type':'application/json', Authorization:'Bearer '+jwt}, body: JSON.stringify({amount})});
    const j = await r.json(); if (r.ok) { setBalance(j.balance); setMessage('Deposit ok'); } else setMessage(j.error || 'Deposit failed');
  }

  async function withdraw(amount){
    const r = await fetch(API + '/api/withdraw', {method:'POST', headers:{'Content-Type':'application/json', Authorization:'Bearer '+jwt}, body: JSON.stringify({amount})});
    const j = await r.json(); if (r.ok) { setBalance(j.balance); setMessage('Withdraw ok'); } else setMessage(j.error || 'Withdraw failed');
  }

  async function linkUpi(upi){
    const r = await fetch(API + '/api/link-upi', {method:'POST', headers:{'Content-Type':'application/json', Authorization:'Bearer '+jwt}, body: JSON.stringify({upi})});
    const j = await r.json(); if (r.ok) setMessage('UPI linked: '+j.upi); else setMessage(j.error);
  }

  async function verify(){
    const r = await fetch(API + '/api/verify', {method:'POST', headers:{Authorization:'Bearer '+jwt}});
    const j = await r.json(); if (r.ok) setMessage('Verified'); else setMessage(j.error);
  }

  return (
    <div className="p-6 font-sans">
      <h1 className="text-2xl font-bold">TSR Gaming Wallet (Starter)</h1>
      <div className="mt-3">
        {user ? (
          <div>
            <div>Welcome {user.name} ({user.email})</div>
            <div>Balance: ₹{balance}</div>
            <button onClick={()=>{localStorage.removeItem('jwt'); setJwt(null); setUser(null);}}>Sign out</button>
          </div>
        ) : (
          <div>
            <button onClick={handleGoogleSignInDemo}>Sign in with Google (dev)</button>
            <div className="text-sm text-gray-500">(Follow README to enable real Google sign-in)</div>
          </div>
        )}
      </div>

      {/* actions */}
      <div className="mt-4 grid grid-cols-1 md:grid-cols-3 gap-3">
        <div>
          <h3>Deposit</h3>
          <input id="damount" type="number" placeholder="amount" />
          <button onClick={()=>deposit(Number(document.getElementById('damount').value))}>Deposit</button>
        </div>
        <div>
          <h3>Withdraw</h3>
          <input id="wamount" type="number" placeholder="amount" />
          <button onClick={()=>withdraw(Number(document.getElementById('wamount').value))}>Withdraw</button>
        </div>
        <div>
          <h3>Link UPI</h3>
          <input id="upi" placeholder="yourupi@bank" />
          <button onClick={()=>linkUpi(document.getElementById('upi').value)}>Link UPI</button>
        </div>
      </div>

      <div className="mt-4">
        <button onClick={verify}>Verify Account</button>
      </div>

      {message && <div className="mt-3 p-2 bg-yellow-100">{message}</div>}
    </div>
  );
}
```

---

# 5) Notes, security & next steps (must read)

1. **Never** trust frontend for amounts: validate on server and use payment gateway webhooks for final confirmation.
2. **KYC & Legal**: For handling money in India, follow RBI rules. Consult a lawyer. Add PAN/Aadhaar KYC only via secure uploads and storage.
3. **Encryption**: Use HTTPS, encrypt sensitive data at rest, protect DB backups.
4. **Payment Gateway**: Integrate Razorpay/Paytm with server-side order creation and webhook verification.
5. **Testing**: Use test accounts and test modes offered by payment providers.
6. **Audit**: Before launching, get a security & financial audit.

---

# 6) Want me to do next right now? (I will perform here immediately)

I can do one of these **right now in this document** (pick one and I will implement):
- A) Add full Google Identity client‑side code and instructions with snippet (assuming you will provide CLIENT_ID). 
- B) Add Razorpay server + client integration example (order creation + verify webhook).
- C) Replace demo front-end sign-in with a simple dev endpoint that creates a demo user (so you can test without Google tokens).
- D) Convert all UI labels to Hindi/Urdu.

Mujhe batao kaunsa choose karna hai — main turant woh change yahin add kar dunga aur code ready kar dunga.

---

// END OF DOCUMENT
