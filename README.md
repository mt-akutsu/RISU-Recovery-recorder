<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<meta name="robots" content="noindex">
<title>面談録音</title>
<style>
:root{--bg:#f4f6f9;--card:#fff;--ink:#1c2430;--mute:#5d6b7c;--line:#dfe5ec;--acc:#245b9c;--acc-ink:#fff;--rec:#c8372d;--ok:#1f7a4d;--warn-bg:#fff4dc;--warn-ink:#6b4a00;--err:#b3261e}
@media(prefers-color-scheme:dark){:root{--bg:#12161c;--card:#1b2129;--ink:#e8edf3;--mute:#9aa7b6;--line:#2c3541;--acc:#6aa7e6;--acc-ink:#0b1622;--rec:#ef6b61;--ok:#5cc58f;--warn-bg:#3a2f12;--warn-ink:#f0d48a;--err:#ff8a80}}
*{box-sizing:border-box}
html,body{margin:0}
body{background:var(--bg);color:var(--ink);font:16px/1.6 system-ui,-apple-system,"Hiragino Sans","Noto Sans JP",sans-serif;padding:0 16px calc(24px + env(safe-area-inset-bottom))}
main{max-width:520px;margin:0 auto}
h1{font-size:20px;margin:18px 0 4px}
.sub{color:var(--mute);font-size:14px;margin:0 0 14px}
.card{background:var(--card);border:1px solid var(--line);border-radius:12px;padding:14px;margin:0 0 12px}
.notice{background:var(--warn-bg);color:var(--warn-ink);border-radius:10px;padding:10px 12px;font-size:14px;margin:0 0 12px}
label{display:block;font-size:14px;color:var(--mute);margin:10px 0 4px}
input[type=text],input[type=password],input[type=url]{width:100%;font:inherit;padding:11px 12px;border:1px solid var(--line);border-radius:8px;background:var(--bg);color:var(--ink)}
button{font:inherit;border:1px solid var(--line);background:var(--card);color:var(--ink);border-radius:10px;padding:12px 16px;min-height:48px;cursor:pointer}
button.primary{background:var(--acc);color:var(--acc-ink);border-color:var(--acc)}
button:disabled{opacity:.45}
.row{display:flex;gap:8px;flex-wrap:wrap}
.row>button{flex:1 1 auto}
.stage{text-align:center;padding:8px 0 2px}
#timer{font-size:44px;font-variant-numeric:tabular-nums;letter-spacing:.02em;margin:4px 0}
#state{font-size:14px;color:var(--mute);min-height:22px}
#rec{width:128px;height:128px;border-radius:50%;border:0;background:var(--rec);color:#fff;font-size:17px;font-weight:700;margin:10px auto;display:block;position:relative}
#rec.on{border-radius:26px;width:104px;height:104px}
#meter{height:10px;border-radius:5px;background:var(--line);overflow:hidden;margin:10px 0 2px}
#meter i{display:block;height:100%;width:0;background:var(--ok);transition:width .08s linear}
.msg{font-size:14px;margin:8px 0 0;min-height:21px}
.msg.err{color:var(--err)}.msg.ok{color:var(--ok)}
progress{width:100%;height:10px;margin-top:10px}
details summary{cursor:pointer;font-weight:600}
.small{font-size:13px;color:var(--mute)}
.pend{border-top:1px solid var(--line);padding-top:10px;margin-top:10px}
.pend:first-child{border-top:0;padding-top:0;margin-top:0}
[hidden]{display:none!important}
</style>
</head>
<body>
<main>
<h1>面談録音</h1>
<p class="sub">録音して、Google ドライブの指定フォルダに保存します。</p>

<div class="notice"><b>録音中は画面を点けたまま、このページを開いたままにしてください。</b>画面を消したり別のアプリに切り替えたりすると、録音が止まることがあります。録音は Google ドライブに保存されます（このページには残りません）。</div>

<div class="card" id="pendCard" hidden>
  <b>送れていない録音があります</b>
  <div id="pendList"></div>
</div>

<div class="card">
  <label for="memo" style="margin-top:0">ファイル名に付ける目印（任意）</label>
  <input type="text" id="memo" maxlength="30" placeholder="例：11月定期" autocomplete="off">
  <div class="stage">
    <div id="timer">00:00</div>
    <div id="state">待機中</div>
    <button id="rec" type="button">録音開始</button>
    <div id="meter" aria-hidden="true"><i></i></div>
    <div class="row" style="margin-top:10px"><button id="pause" type="button" disabled>一時停止</button></div>
  </div>
  <progress id="prog" max="100" value="0" hidden></progress>
  <div class="msg" id="msg" role="status"></div>
  <div class="row" id="after" hidden style="margin-top:8px">
    <button id="retry" type="button" class="primary" hidden>もう一度送る</button>
    <button id="dl" type="button" hidden>この端末に保存</button>
    <button id="again" type="button" hidden>続けて録音する</button>
  </div>
</div>

<details class="card" id="setup">
  <summary>設定（初回のみ）</summary>
  <label for="gurl">受け取り窓口の URL（…/exec）</label>
  <input type="url" id="gurl" placeholder="https://script.google.com/macros/s/…/exec" autocomplete="off">
  <label for="gpass">合言葉</label>
  <input type="password" id="gpass" autocomplete="off">
  <div class="row" style="margin-top:12px">
    <button id="savecfg" type="button" class="primary">保存して接続を確かめる</button>
  </div>
  <div class="msg" id="cfgmsg"></div>
  <p class="small">URL と合言葉は、この端末のブラウザにだけ保存されます。</p>
</details>
</main>

<script>
(function(){
'use strict';
const $=s=>document.querySelector(s);
const q=new URLSearchParams(location.search);
const UNIT=(Number(q.get('unit'))>0?Number(q.get('unit')):3*1024*1024); // 262144 の倍数（テスト用に変更可）
const LS='mendanRec.v1';
const cfg={url:'',pass:''};

/* ---------- 設定 ---------- */
function loadCfg(){
  try{const j=JSON.parse(localStorage.getItem(LS)||'{}');cfg.url=j.url||'';cfg.pass=j.pass||''}catch(e){}
  const h=new URLSearchParams(location.hash.slice(1));      // 共有リンク：#u=URL&k=合言葉
  if(h.get('u')){cfg.url=h.get('u');cfg.pass=h.get('k')||cfg.pass;saveCfg();try{history.replaceState(null,'',location.pathname+location.search)}catch(e){}}
  $('#gurl').value=cfg.url;$('#gpass').value=cfg.pass;
  if(!cfg.url||!cfg.pass)$('#setup').open=true;
}
function saveCfg(){try{localStorage.setItem(LS,JSON.stringify(cfg))}catch(e){}}

/* ---------- 端末内の一時保存（IndexedDB。途中でページが閉じても録音を失わないため） ---------- */
let dbp=null;
function db(){
  if(dbp)return dbp;
  dbp=new Promise((res,rej)=>{
    const r=indexedDB.open('mendanRec',1);
    r.onupgradeneeded=()=>{const d=r.result;d.createObjectStore('meta',{keyPath:'rid'});const c=d.createObjectStore('chunks',{autoIncrement:true});c.createIndex('rid','rid')};
    r.onsuccess=()=>res(r.result);r.onerror=()=>rej(r.error);
  });
  return dbp;
}
async function tx(store,mode,fn){
  const d=await db();
  return new Promise((res,rej)=>{const t=d.transaction(store,mode);const out=fn(t.objectStore(store));t.oncomplete=()=>res(out&&out.result!==undefined?out.result:undefined);t.onerror=()=>rej(t.error);t.onabort=()=>rej(t.error)});
}
const idbPutMeta=m=>tx('meta','readwrite',s=>s.put(m)).catch(()=>{});
const idbAddChunk=c=>tx('chunks','readwrite',s=>s.add(c)).catch(()=>{});
const idbAllMeta=()=>tx('meta','readonly',s=>s.getAll()).catch(()=>[]);
async function idbChunks(rid){
  const d=await db();
  return new Promise((res,rej)=>{const out=[];const r=d.transaction('chunks').objectStore('chunks').index('rid').openCursor(IDBKeyRange.only(rid));r.onsuccess=()=>{const c=r.result;if(c){out.push(c.value);c.continue()}else res(out.sort((a,b)=>a.seq-b.seq))};r.onerror=()=>rej(r.error)});
}
async function idbDrop(rid){
  const d=await db();
  await new Promise(res=>{const t=d.transaction(['meta','chunks'],'readwrite');t.objectStore('meta').delete(rid);const r=t.objectStore('chunks').index('rid').openKeyCursor(IDBKeyRange.only(rid));r.onsuccess=()=>{const c=r.result;if(c){t.objectStore('chunks').delete(c.primaryKey);c.continue()}};t.oncomplete=res;t.onerror=res;t.onabort=res});
}
async function blobOf(meta){
  const cs=await idbChunks(meta.rid);
  return new Blob(cs.map(c=>c.blob),{type:baseMime(meta.mime)});
}

/* ---------- 画面の部品 ---------- */
const ui={
  msg(t,cls){const m=$('#msg');m.textContent=t||'';m.className='msg'+(cls?' '+cls:'')},
  state(t){$('#state').textContent=t},
  timer(ms){const s=Math.floor(ms/1000),h=Math.floor(s/3600),m=Math.floor(s%3600/60),x=s%60;$('#timer').textContent=(h?h+':':'')+String(m).padStart(2,'0')+':'+String(x).padStart(2,'0')},
  prog(p){const e=$('#prog');if(p==null){e.hidden=true;return}e.hidden=false;e.value=p},
  after(show){$('#after').hidden=!show}
};
const baseMime=m=>String(m||'').split(';')[0];
const extOf=m=>baseMime(m)==='audio/mp4'?'m4a':baseMime(m)==='audio/webm'?'webm':'bin';
const mb=n=>n<1048576?Math.max(1,Math.round(n/1024))+' KB':(n/1048576).toFixed(1)+' MB';
const dur=ms=>ms<60000?Math.max(1,Math.round(ms/1000))+'秒':Math.round(ms/60000)+'分';
function pad(n){return String(n).padStart(2,'0')}
function stamp(t){const d=new Date(t);return d.getFullYear()+pad(d.getMonth()+1)+pad(d.getDate())+'_'+pad(d.getHours())+pad(d.getMinutes())}
function fileName(meta){
  const memo=(meta.memo||'').replace(/[\\\/:*?"<>|\s]+/g,'_').replace(/^_+|_+$/g,'');
  return '面談_'+stamp(meta.startedAt)+(memo?'_'+memo:'')+'.'+extOf(meta.mime);
}

/* ---------- 録音 ---------- */
let S='idle';                 // idle / recording / paused / uploading / failed / done
let stream=null,mr=null,actx=null,raf=0,wake=null;
let cur=null,parts=[],seq=0,pending=Promise.resolve();
let t0=0,accum=0,tick=0,quietSince=0,quietWarn=false,lastHidden=false;

function pickMime(){
  if(!window.MediaRecorder)return '';
  const c=['audio/mp4;codecs=mp4a.40.2','audio/mp4','audio/webm;codecs=opus','audio/webm'];
  return c.find(t=>{try{return MediaRecorder.isTypeSupported(t)}catch(e){return false}})||'';
}
function elapsed(){return accum+(S==='recording'?performance.now()-t0:0)}
function setS(s){
  S=s;
  const rec=$('#rec');
  rec.classList.toggle('on',s==='recording'||s==='paused');
  rec.textContent=(s==='recording'||s==='paused')?'録音終了':'録音開始';
  rec.disabled=(s==='uploading');
  $('#pause').disabled=!(s==='recording'||s==='paused');
  $('#pause').textContent=s==='paused'?'再開':'一時停止';
  $('#memo').disabled=(s!=='idle'&&s!=='done'&&s!=='failed');
  ui.state({idle:'待機中',recording:'録音中',paused:'一時停止中',uploading:'ドライブに保存中…',failed:'保存できませんでした',done:'保存しました'}[s]||'');
}
async function lockScreen(){try{if(navigator.wakeLock&&!wake){wake=await navigator.wakeLock.request('screen');wake.addEventListener('release',()=>{wake=null})}}catch(e){}}
function unlockScreen(){try{if(wake)wake.release()}catch(e){}wake=null}

async function start(){
  if(!cfg.url||!cfg.pass){$('#setup').open=true;ui.msg('先に「設定」で、受け取り窓口の URL と合言葉を入れてください。','err');return}
  if(!navigator.mediaDevices||!navigator.mediaDevices.getUserMedia){ui.msg('このブラウザでは録音できません。https で開いているか、Chrome で開いているかを確認してください。','err');return}
  const mime=pickMime();
  if(!mime){ui.msg('このブラウザは録音の形式に対応していません。Chrome を最新にしてください。','err');return}
  ui.msg('');ui.after(false);ui.prog(null);
  try{
    stream=await navigator.mediaDevices.getUserMedia({audio:{channelCount:1,echoCancellation:false,noiseSuppression:false,autoGainControl:true}});
  }catch(e){
    ui.msg(e&&e.name==='NotAllowedError'?'マイクの使用が許可されていません。ブラウザのアドレスバーの設定から、マイクを「許可」にしてください。':'マイクを開けませんでした：'+(e&&e.message||e),'err');return;
  }
  const rid=Date.now().toString(36)+Math.random().toString(36).slice(2,6);
  cur={rid,mime,startedAt:Date.now(),memo:$('#memo').value.trim(),status:'recording',name:'',size:0,session:'',next:0};
  parts=[];seq=0;pending=Promise.resolve();accum=0;quietSince=0;
  await idbPutMeta(cur);
  try{
    mr=new MediaRecorder(stream,{mimeType:mime,audioBitsPerSecond:64000});
  }catch(e){ui.msg('録音を始められませんでした：'+(e&&e.message||e),'err');stream.getTracks().forEach(t=>t.stop());await idbDrop(rid);return}
  mr.ondataavailable=e=>{if(e.data&&e.data.size){parts.push(e.data);const n=seq++,b=e.data;pending=pending.then(()=>idbAddChunk({rid,seq:n,blob:b}))}};
  mr.onstop=onStopped;
  mr.onerror=e=>ui.msg('録音でエラーが起きました。ここまでの録音は保存を試みます。','err');
  stream.getTracks().forEach(t=>t.addEventListener('ended',()=>{if(S==='recording'||S==='paused'){ui.msg('マイクが切れました（通話などの割り込みの可能性があります）。ここまでの録音を保存します。','err');finish()}}));
  mr.start(5000);
  t0=performance.now();setS('recording');lockScreen();meterStart();
  tick=setInterval(()=>{ui.timer(elapsed())},250);
}
function finish(){
  if(!mr||mr.state==='inactive'){if(S==='recording'||S==='paused')onStopped();return}
  if(S==='recording')accum+=performance.now()-t0;
  try{mr.stop()}catch(e){onStopped()}
}
function togglePause(){
  if(!mr)return;
  if(S==='recording'){accum+=performance.now()-t0;try{mr.pause()}catch(e){}setS('paused')}
  else if(S==='paused'){try{mr.resume()}catch(e){}t0=performance.now();setS('recording')}
}
let stopping=false;
async function onStopped(){
  if(stopping)return;stopping=true;
  clearInterval(tick);meterStop();
  try{stream&&stream.getTracks().forEach(t=>t.stop())}catch(e){}
  await pending;
  const meta=cur;meta.durationMs=Math.round(accum||elapsed());
  const blob=new Blob(parts,{type:baseMime(meta.mime)});
  stopping=false;
  if(!blob.size){ui.msg('録音が空でした。もう一度録音してください。','err');await idbDrop(meta.rid);setS('idle');unlockScreen();return}
  meta.size=blob.size;meta.name=fileName(meta);meta.status='unsent';
  await idbPutMeta(meta);
  ui.timer(meta.durationMs);
  await send(meta,blob);
}

/* ---------- 音量メーター（マイクに音が入っているかの確認用） ---------- */
function meterStart(){
  try{
    const AC=window.AudioContext||window.webkitAudioContext;actx=new AC();
    const src=actx.createMediaStreamSource(stream),an=actx.createAnalyser();an.fftSize=1024;src.connect(an);
    const buf=new Uint8Array(an.fftSize),bar=$('#meter i');quietSince=performance.now();
    const loop=()=>{
      an.getByteTimeDomainData(buf);let m=0;for(let i=0;i<buf.length;i++){m=Math.max(m,Math.abs(buf[i]-128))}
      const lv=Math.min(1,m/64);bar.style.width=(lv*100)+'%';
      if(S==='recording'){
        if(lv>0.04){quietSince=performance.now();if(quietWarn){quietWarn=false;ui.msg('')}}
        else if(performance.now()-quietSince>15000){ui.msg('音がほとんど入っていません。マイクをふさいでいないか、話し手の近くに置いているか確認してください。','err');quietWarn=true;quietSince=performance.now()}
      }
      raf=requestAnimationFrame(loop);
    };
    loop();
  }catch(e){}
}
function meterStop(){cancelAnimationFrame(raf);try{actx&&actx.close()}catch(e){}actx=null;$('#meter i').style.width='0'}

/* ---------- 送信（受け取り窓口へ分割して送る） ---------- */
class SendErr extends Error{constructor(m,fatal){super(m);this.fatal=!!fatal}}
async function api(body){
  let r,t;
  try{
    r=await fetch(cfg.url,{method:'POST',headers:{'Content-Type':'text/plain;charset=utf-8'},body:JSON.stringify(Object.assign({pass:cfg.pass},body)),redirect:'follow'});
    t=await r.text();
  }catch(e){throw new SendErr('通信できませんでした。電波や Wi-Fi を確認してください。')}
  let j;try{j=JSON.parse(t)}catch(e){throw new SendErr('受け取り窓口から想定外の応答がありました。URL が正しいか、公開の設定が「全員」になっているかを確認してください。（応答：HTTP '+r.status+' '+String(t).replace(/<[^>]*>/g,' ').replace(/\s+/g,' ').trim().slice(0,120)+'）',body.action!=='chunk')}
  if(j&&j.ok===false){
    if(j.error==='auth')throw new SendErr('合言葉が違います。「設定」で確認してください。',true);
    if(j.error==='session')throw Object.assign(new SendErr('送信の途中経過が失われました。最初から送り直します。'),{session:true});
    if(j.error==='drive')throw new SendErr('ドライブへの保存でエラーになりました：'+(j.message||''));
    throw new SendErr('受け取り窓口でエラーが起きました：'+(j.message||j.error||''));
  }
  return j;
}
function b64(blob){return new Promise((res,rej)=>{const r=new FileReader();r.onload=()=>res(String(r.result).split(',')[1]||'');r.onerror=()=>rej(r.error);r.readAsDataURL(blob)})}
const sleep=ms=>new Promise(r=>setTimeout(r,ms));

async function send(meta,blob){
  setS('uploading');ui.after(false);ui.prog(0);ui.msg('');lockScreen();
  window.onbeforeunload=e=>{e.preventDefault();e.returnValue=''};
  try{
    let restarted=false;
    for(;;){
      try{
        if(!meta.session||meta.session==='x'){
          const s=await api({action:'start',name:meta.name,mime:baseMime(meta.mime),size:blob.size});
          meta.session=s.id;meta.next=0;meta.name=s.name||meta.name;await idbPutMeta(meta);
        }
        while(meta.next<blob.size){
          const off=meta.next,part=blob.slice(off,Math.min(off+UNIT,blob.size));
          let res=null,lastErr=null;
          for(let a=0;a<4&&!res;a++){
            try{res=await api({action:'chunk',id:meta.session,offset:off,data:await b64(part)})}
            catch(e){if(e.fatal||e.session)throw e;lastErr=e;ui.msg('送信をやり直しています…（'+(a+1)+'/3）');await sleep(1500*(a+1))}
          }
          if(!res)throw lastErr;
          if(res.done){meta.next=blob.size}else{meta.next=Math.min(res.next,blob.size)}
          ui.prog(Math.round(meta.next/blob.size*100));
          await idbPutMeta(meta);
          if(res.done)break;
        }
        break;
      }catch(e){
        if(e.session&&!restarted){restarted=true;meta.session='x';meta.next=0;continue}
        throw e;
      }
    }
    await idbDrop(meta.rid);
    window.onbeforeunload=null;unlockScreen();
    setS('done');ui.prog(100);
    ui.msg('ドライブに保存しました：'+meta.name+'（'+(meta.durationMs?dur(meta.durationMs)+'・':'')+mb(blob.size)+'）','ok');
    ui.after(true);$('#retry').hidden=true;$('#dl').hidden=true;$('#again').hidden=false;
    renderPending();
  }catch(e){
    window.onbeforeunload=null;unlockScreen();
    setS('failed');ui.prog(null);
    ui.msg((e&&e.message)||'送れませんでした。','err'+'');
    ui.after(true);$('#retry').hidden=false;$('#dl').hidden=false;$('#again').hidden=true;
    failed={meta,blob};
    renderPending();
  }
}
let failed=null;
function download(meta,blob){
  const a=document.createElement('a');a.href=URL.createObjectURL(blob);a.download=meta.name||fileName(meta);document.body.appendChild(a);a.click();setTimeout(()=>{URL.revokeObjectURL(a.href);a.remove()},4000);
}

/* ---------- 送れていない録音（前回途中で止まったものなど） ---------- */
async function renderPending(){
  const list=await idbAllMeta();
  const box=$('#pendList');box.textContent='';
  const items=list.filter(m=>!(cur&&m.rid===cur.rid&&(S==='recording'||S==='paused'||S==='uploading')));
  $('#pendCard').hidden=!items.length;
  items.sort((a,b)=>a.startedAt-b.startedAt).forEach(m=>{
    const d=document.createElement('div');d.className='pend';
    const t=document.createElement('div');
    const dt=new Date(m.startedAt);
    t.textContent=(dt.getMonth()+1)+'/'+dt.getDate()+' '+pad(dt.getHours())+':'+pad(dt.getMinutes())+' の録音'+(m.size?'（'+mb(m.size)+'）':'')+(m.status==='recording'?'　※録音中に中断されたものです':'');
    const r=document.createElement('div');r.className='row';r.style.marginTop='6px';
    const b1=document.createElement('button');b1.type='button';b1.className='primary';b1.textContent='ドライブに送る';
    const b2=document.createElement('button');b2.type='button';b2.textContent='端末に保存';
    const b3=document.createElement('button');b3.type='button';b3.textContent='破棄';
    b1.onclick=async()=>{
      if(S==='recording'||S==='paused'||S==='uploading')return;
      cur=m;const blob=await blobOf(m);
      if(!blob.size){ui.msg('この録音のデータが見つかりません。','err');return}
      m.size=blob.size;if(!m.name)m.name=fileName(m);m.status='unsent';await send(m,blob)};
    b2.onclick=async()=>{const blob=await blobOf(m);if(!m.name)m.name=fileName(m);download(m,blob)};
    b3.onclick=async()=>{if(confirm('この録音を端末から削除します。よろしいですか？')){await idbDrop(m.rid);renderPending()}};
    r.append(b1,b2,b3);d.append(t,r);box.append(d);
  });
}

/* ---------- 操作 ---------- */
$('#rec').onclick=()=>{if(S==='recording'||S==='paused')finish();else if(S!=='uploading')start()};
$('#pause').onclick=togglePause;
$('#retry').onclick=()=>{if(failed)send(failed.meta,failed.blob)};
$('#dl').onclick=()=>{if(failed)download(failed.meta,failed.blob)};
$('#again').onclick=()=>{setS('idle');ui.msg('');ui.after(false);ui.prog(null);ui.timer(0)};
$('#savecfg').onclick=async()=>{
  cfg.url=$('#gurl').value.trim();cfg.pass=$('#gpass').value;saveCfg();
  const m=$('#cfgmsg');m.className='msg';m.textContent='確認中…';
  try{
    const r=await api({action:'ping'});
    m.className='msg ok';m.textContent='接続できました。保存先フォルダ：'+(r.folder||'');
  }catch(e){m.className='msg err';m.textContent=e.message}
};
document.addEventListener('visibilitychange',()=>{
  if(document.visibilityState==='visible'){
    if(S==='recording'||S==='paused'||S==='uploading')lockScreen();
    if(lastHidden&&(S==='recording')&&mr&&mr.state!=='recording')ui.msg('録音が止まっていた可能性があります。録音時間を確認してください。','err');
    lastHidden=false;
  }else if(S==='recording')lastHidden=true;
});
window.addEventListener('beforeunload',e=>{if(S==='recording'||S==='paused'){e.preventDefault();e.returnValue=''}});

loadCfg();setS('idle');ui.timer(0);renderPending();
if(!window.isSecureContext)ui.msg('このページは https で開く必要があります（マイクが使えません）。','err');
window.__rec={get state(){return S},cfg,UNIT};
})();
</script>
</body>
</html>
