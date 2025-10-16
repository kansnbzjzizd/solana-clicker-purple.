# solana-clicker-purple.
<!doctype html>

<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Solana Clicker — Purple</title>
  <style>
    :root{--bg:#2b0b3a;--card:#36123f;--accent:#8a2be2;--muted:#bda6d9}
    html,body{height:100%;margin:0;font-family:Inter,system-ui,Arial,sans-serif;background:linear-gradient(180deg,var(--bg),#1a0520);color:white}
    .container{max-width:420px;margin:24px auto;padding:20px}
    .card{background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent);border-radius:14px;padding:16px;box-shadow:0 6px 18px rgba(0,0,0,0.5)}
    h1{margin:0 0 12px;font-size:20px}
    .balance-row{display:flex;justify-content:space-between;align-items:center;padding:12px 10px;border-radius:10px;background:rgba(255,255,255,0.02);margin-bottom:14px}
    .balance-label{font-size:13px;color:var(--muted)}
    .balance-amount{font-weight:700;font-size:18px}
    .click-area{display:flex;justify-content:center;margin:18px 0}
    .circle-btn{width:170px;height:170px;border-radius:50%;background:linear-gradient(135deg,var(--accent),#5c22c8);display:flex;flex-direction:column;justify-content:center;align-items:center;cursor:pointer;box-shadow:0 8px 24px rgba(138,43,226,0.25);border:6px solid rgba(255,255,255,0.04)}
    .circle-btn:active{transform:scale(0.98)}
    .sol-text{font-size:18px;font-weight:700}
    .tiny{font-size:12px;color:var(--muted)}
    .row{display:flex;gap:8px}
    button.btn{padding:10px 14px;border-radius:10px;border:0;background:#fff;color:#2b0b3a;font-weight:700;cursor:pointer}
    .ghost{background:transparent;border:1px solid rgba(255,255,255,0.06);color:var(--muted)}
    .footer{margin-top:12px;display:flex;justify-content:space-between;align-items:center}
    .wallet-btn{background:transparent;border:1px solid rgba(255,255,255,0.08);padding:8px 12px;border-radius:10px;color:var(--muted);cursor:pointer}
    /* Withdraw page */
    .hidden{display:none}
    .form-row{display:flex;gap:8px;margin:8px 0}
    input.input{flex:1;padding:10px;border-radius:8px;border:1px solid rgba(255,255,255,0.06);background:transparent;color:white}
    .owner-panel{margin-top:12px;padding:10px;border-radius:8px;background:rgba(0,0,0,0.1)}
    .request{padding:8px;border-radius:8px;margin-bottom:8px;background:rgba(255,255,255,0.02);display:flex;justify-content:space-between;align-items:center}
    small.muted{color:var(--muted)}
    @media(max-width:420px){.circle-btn{width:140px;height:140px}}
  </style>
</head>
<body>
  <div class="container">
    <div class="card" id="home">
      <h1>Solana Clicker — Purple Edition</h1>
      <div class="balance-row">
        <div>
          <div class="balance-label">Available Balance</div>
          <div class="balance-amount" id="displayBalance">0.000000 SOL</div>
        </div>
        <div>
          <button class="wallet-btn" id="connectBtn">Connect Wallet</button>
        </div>
      </div><div class="click-area">
    <div class="circle-btn" id="clicker">
      <div class="sol-text">SOL</div>
      <div class="tiny">Click to earn</div>
    </div>
  </div>

  <div style="text-align:center" class="tiny">Each click adds <strong>0.000001 SOL</strong> to your virtual balance</div>

  <div class="footer">
    <div class="row">
      <button class="btn" id="withdrawPageBtn">Withdraw</button>
      <button class="btn ghost" id="ownerBtn">Owner Panel</button>
    </div>
    <div class="tiny">ENGLISH ONLY</div>
  </div>
</div>

<!-- Withdraw Page -->
<div class="card hidden" id="withdrawPage">
  <h1>Withdraw</h1>
  <div class="form-row">
    <input class="input" id="withdrawAddr" placeholder="Recipient Solana address (e.g. your wallet)" />
  </div>
  <div class="form-row">
    <input class="input" id="withdrawAmt" placeholder="Amount (SOL)" />
  </div>
  <div style="display:flex;gap:8px;justify-content:flex-end;margin-top:8px">
    <button class="btn ghost" id="backBtn">Back</button>
    <button class="btn" id="requestWithdrawBtn">Request Withdraw</button>
  </div>

  <div class="owner-panel">
    <div class="tiny">Important — This interface <strong>does NOT</strong> send SOL automatically. It creates a withdrawal request that the owner must review and pay manually.</div>
  </div>
</div>

<!-- Owner Panel (protected by passphrase) -->
<div class="card hidden" id="ownerPage">
  <h1>Owner Panel</h1>
  <div class="form-row">
    <input class="input" id="ownerPass" placeholder="Owner passphrase" />
    <button class="btn" id="ownerLoginBtn">Login</button>
  </div>
  <div id="requestsList"></div>
  <div style="margin-top:10px" class="tiny">When you send SOL manually from your wallet, paste the transaction signature into the request and mark as Paid.</div>
</div>

  </div>  <script>
    // === CONFIG ===
    const CLICK_VALUE = 0.000001; // virtual increment per click
    const OWNER_PASS = 'changeme'; // change this before deployment

    // === UTIL / STORAGE ===
    function fmt(x){return Number(x).toFixed(6)}
    function getWalletKey(){return window.currentPublicKey || localStorage.getItem('connected_pk')}

    function loadState(){
      const pk = getWalletKey();
      if(!pk) return 0;
      const balances = JSON.parse(localStorage.getItem('balances')||'{}');
      return balances[pk] || 0;
    }
    function saveBalance(pk, val){
      const balances = JSON.parse(localStorage.getItem('balances')||'{}');
      balances[pk]=val; localStorage.setItem('balances',JSON.stringify(balances));
    }

    // Withdraw requests stored in localStorage
    function getRequests(){return JSON.parse(localStorage.getItem('withdraw_requests')||'[]')}
    function addRequest(req){const arr=getRequests();arr.push(req);localStorage.setItem('withdraw_requests',JSON.stringify(arr))}
    function updateRequests(arr){localStorage.setItem('withdraw_requests',JSON.stringify(arr))}

    // === UI ===
    const displayBalance = document.getElementById('displayBalance');
    const clicker = document.getElementById('clicker');
    const connectBtn = document.getElementById('connectBtn');
    const withdrawPageBtn = document.getElementById('withdrawPageBtn');
    const ownerBtn = document.getElementById('ownerBtn');
    const home = document.getElementById('home');
    const withdrawPage = document.getElementById('withdrawPage');
    const ownerPage = document.getElementById('ownerPage');
    const backBtn = document.getElementById('backBtn');
    const requestWithdrawBtn = document.getElementById('requestWithdrawBtn');
    const withdrawAddr = document.getElementById('withdrawAddr');
    const withdrawAmt = document.getElementById('withdrawAmt');
    const ownerLoginBtn = document.getElementById('ownerLoginBtn');
    const ownerPass = document.getElementById('ownerPass');
    const requestsList = document.getElementById('requestsList');

    function show(el){home.classList.add('hidden');withdrawPage.classList.add('hidden');ownerPage.classList.add('hidden');el.classList.remove('hidden')}

    function refreshBalance(){
      const val = loadState();
      displayBalance.textContent = fmt(val) + ' SOL';
    }

    // === Phantom wallet connect (basic) ===
    async function connectPhantom(){
      try{
        if(window.solana && window.solana.isPhantom){
          const resp = await window.solana.connect();
          const pk = resp.publicKey.toString();
          window.currentPublicKey = pk;
          localStorage.setItem('connected_pk', pk);
          connectBtn.textContent = pk.slice(0,6)+'...'+pk.slice(-4);
          refreshBalance();
        }else{
          alert('Phantom wallet not found. Install Phantom or open in compatible browser.');
        }
      }catch(e){console.error(e)}
    }

    connectBtn.addEventListener('click',connectPhantom);

    // Clicker logic
    clicker.addEventListener('click',()=>{
      const pk = getWalletKey();
      if(!pk){alert('Please connect your wallet first.');return}
      const bal = loadState();
      const newBal = +(bal + CLICK_VALUE).toFixed(6);
      saveBalance(pk,newBal);
      refreshBalance();
    });

    // Withdraw flow - create a request (owner must pay manually)
    withdrawPageBtn.addEventListener('click',()=>{show(withdrawPage)});
    backBtn.addEventListener('click',()=>{show(home)});

    requestWithdrawBtn.addEventListener('click',()=>{
      const pk = getWalletKey();
      if(!pk){alert('Connect wallet first');return}
      const amt = parseFloat(withdrawAmt.value);
      if(isNaN(amt) || amt<=0){alert('Enter a valid amount');return}
      const current = loadState();
      if(amt>current){alert('Not enough balance');return}

      const req={
        id:Date.now(),
        wallet:pk,
        to:withdrawAddr.value || pk,
        amount:amt,
        status:'pending',
        requestedAt:new Date().toISOString(),
        txSignature:null
      };
      addRequest(req);
      // deduct virtual balance
      saveBalance(pk, +(current-amt).toFixed(6));
      refreshBalance();
      alert('Withdraw request created. Owner will review and pay manually.');
      withdrawAddr.value=''; withdrawAmt.value='';
      show(home);
    });

    // Owner panel
    ownerBtn.addEventListener('click',()=>{show(ownerPage)});
    ownerLoginBtn.addEventListener('click',()=>{
      if(ownerPass.value===OWNER_PASS){renderRequests();}else{alert('Wrong passphrase')}
    });

    function renderRequests(){
      const arr = getRequests();
      requestsList.innerHTML='';
      if(arr.length===0){requestsList.innerHTML='<div class="tiny muted">No requests</div>';return}
      arr.slice().reverse().forEach(r=>{
        const div=document.createElement('div');div.className='request';
        const left=document.createElement('div');
        left.innerHTML=`<div><strong>${r.amount} SOL</strong> — <small class="muted">to ${r.to}</small></div><small class="muted">${new Date(r.requestedAt).toLocaleString()}</small>`;
        const right=document.createElement('div');
        if(r.status==='pending'){
          const payBtn=document.createElement('button');payBtn.className='btn ghost';payBtn.textContent='Mark Paid';
          payBtn.onclick=()=>{
            const sig = prompt('Paste transaction signature (from your wallet) after you send funds manually:');
            if(sig){
              r.status='paid';r.txSignature=sig;updateStatus(r.id,'paid',sig);
              renderRequests();
            }
          };
          right.appendChild(payBtn);
        }else{
          right.innerHTML=`<div style="text-align:right"><small class="muted">${r.status.toUpperCase()}</small><div style="font-size:12px;margin-top:6px">sig:<br>${r.txSignature}</div></div>`;
        }
        div.appendChild(left);div.appendChild(right);requestsList.appendChild(div);
      });
    }

    function updateStatus(id,status,sig){
      const arr=getRequests();
      const idx=arr.findIndex(x=>x.id===id);
      if(idx!==-1){arr[idx].status=status;arr[idx].txSignature=sig;updateRequests(arr)}
    }

    // initial UI
    if(localStorage.getItem('connected_pk')){
      connectBtn.textContent = localStorage.getItem('connected_pk').slice(0,6)+'...'+localStorage.getItem('connected_pk').slice(-4);
    }
    refreshBalance();
  </script></body>
</html>
