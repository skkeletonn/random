## SoundCloud Track Downloader and Easy Cover/profile pitcture/banner downloader

**NOTE**: ALL EXPLANATION IS FOR PC ONLY, on mobile by default you shouldnt be able to get extension in the first place im pretty sure.
How to use this script:
1. Download the `Tampermonkey` Extension and search up a tutorial on how to set it up for your specific browser.
2. Add a new script (`Create a new script...`) and paste the following code in before saving:
<details>
	<summary>Click to expand</summary>
	
```js
// ==UserScript==
// @name         SoundCloud Track Downloader n shit
// @namespace    https://github.com/local/sc-hq-dl
// @version      7.4.1
// @description  Layout-independent track tools, cover art, and profile avatar/banner downloads. Originals when available; progressive streams otherwise.
// @match        https://soundcloud.com/*
// @grant        GM_xmlhttpRequest
// @grant        unsafeWindow
// @connect      soundcloud.com
// @connect      api-v2.soundcloud.com
// @connect      api.soundcloud.com
// @connect      sndcdn.com
// @connect      *.sndcdn.com
// @connect      *.soundcloud.com
// @noframes
// @run-at       document-start
// ==/UserScript==

(() => {
  'use strict';
  if (window.self !== window.top) return;
  const win = typeof unsafeWindow !== 'undefined' ? unsafeWindow : window;
  const API = 'https://api-v2.soundcloud.com';
  let clientId = null, cidPending = null;
  const excluded = new Set(['sets','likes','reposts','tracks','albums','popular-tracks','comments','followers','following','station']);
  const reserved = new Set(['you','discover','stream','search','upload','settings','messages','notifications','charts','stations','library','terms-of-use','pages','jobs','pro','go']);
  function accessibleDocuments() {
    const docs = [], seen = new Set();
    function visit(doc) {
      if (!doc || seen.has(doc)) return;
      seen.add(doc); docs.push(doc);
      for (const frame of doc.querySelectorAll('iframe')) {
        try { visit(frame.contentDocument); } catch {}
      }
    }
    visit(document);
    return docs;
  }
  function queryEverywhere(selector) {
    return accessibleDocuments().flatMap(doc => [...doc.querySelectorAll(selector)]);
  }
  function capture(value) {
    try {
      const u = new URL(String(value), location.href);
      if (!/(^|\.)soundcloud\.com$/.test(u.hostname)) return;
      const id = u.searchParams.get('client_id');
      if (id && /^[a-zA-Z0-9]{20,}$/.test(id)) clientId = id;
    } catch {}
  }
  try {
    const open = win.XMLHttpRequest.prototype.open;
    win.XMLHttpRequest.prototype.open = function(method, url) { capture(url); return open.apply(this, arguments); };
    const fetch = win.fetch;
    if (fetch) win.fetch = function(...args) { capture(args[0]?.url || args[0]); return fetch.apply(this, args); };
  } catch (e) { console.debug('[SCHQ] Request observation unavailable', e); }
  try { new PerformanceObserver(list => list.getEntries().forEach(e => capture(e.name))).observe({type:'resource', buffered:true}); } catch {}

  function request(url, binary = false) {
    return new Promise((resolve, reject) => GM_xmlhttpRequest({
      method:'GET', url, anonymous:true, responseType:binary ? 'arraybuffer' : 'text', timeout:120000,
      onload(r) {
        if (r.status < 200 || r.status >= 300) return reject(new Error(`HTTP ${r.status}`));
        resolve({body:binary ? r.response : r.responseText, headers:r.responseHeaders || '', url:r.finalUrl || url});
      },
      onerror:() => reject(new Error('Network error (check userscript host permissions).')),
      ontimeout:() => reject(new Error('Request timed out.')),
      onabort:() => reject(new Error('Request cancelled.'))
    }));
  }
  async function json(url) { return JSON.parse((await request(url)).body); }
  async function getClientId() {
    if (clientId) return clientId;
    for (const doc of accessibleDocuments()) {
      try { doc.defaultView.performance.getEntriesByType('resource').forEach(e => capture(e.name)); } catch {}
    }
    if (clientId) return clientId;
    if (!cidPending) cidPending = (async () => {
      const scripts = [...new Set(accessibleDocuments().flatMap(doc => [...doc.scripts].map(s => s.src)))].filter(src => {
        try { return /(^|\.)sndcdn\.com$/.test(new URL(src).hostname); } catch { return false; }
      }).slice(-12).reverse();
      for (const src of scripts) {
        try {
          const text = (await request(src)).body;
          const m = text.match(/client_id\s*[:=]\s*["']([a-zA-Z0-9]{20,})["']/);
          if (m) { clientId = m[1]; break; }
        } catch {}
      }
      if (!clientId) throw new Error('No client_id found. Play a track, then retry.');
      return clientId;
    })().finally(() => { cidPending = null; });
    return cidPending;
  }
  async function api(path, params = {}) {
    const u = new URL(path, API);
    u.searchParams.set('client_id', await getClientId());
    Object.entries(params).forEach(([k,v]) => { if (v != null) u.searchParams.set(k,v); });
    return json(u.href);
  }
  async function resolve(url, kind) {
    const data = await api('/resolve', {url});
    if (data?.kind !== kind) throw new Error(`This URL is not a ${kind}.`);
    return data;
  }
  function trackURL(href) {
    try {
      const u = new URL(href, location.href), p = u.pathname.split('/').filter(Boolean);
      if (u.hostname !== 'soundcloud.com' || reserved.has(p[0]) || excluded.has(p[1])) return null;
      if (p.length !== 2 && !(p.length === 3 && /^s-/.test(p[2]))) return null;
      const out = new URL(u.origin + '/' + p.join('/'));
      if (u.searchParams.has('secret_token')) out.searchParams.set('secret_token', u.searchParams.get('secret_token'));
      return out.href;
    } catch { return null; }
  }
  function profileURL() {
    const p = location.pathname.split('/').filter(Boolean);
    if (!p.length || reserved.has(p[0])) return null;
    if (p.length === 1 || (p.length === 2 && excluded.has(p[1]))) return `https://soundcloud.com/${p[0]}`;
    return null;
  }
  function playerURL() {
    const selectors = ['.playbackSoundBadge__titleLink', '.playbackSoundBadge a[href]', '[class*="playbackSoundBadge"] a[href]', '[data-testid*="player"] a[href]'];
    for (const sel of selectors) for (const a of queryEverywhere(sel)) {
      const url = trackURL(a.href); if (url) return url;
    }
    return null;
  }
  const safe = s => String(s || 'Unknown').replace(/[\\/:*?"<>|\x00-\x1f]/g, '_').trim().slice(0,160);
  function filename(t, ext) {
    const uploader = t.user?.username || 'Unknown', artist = t.publisher_metadata?.artist;
    return safe(uploader + (artist && artist.toLowerCase() !== uploader.toLowerCase() ? ' ∕∕ ' + artist : '') + ' - ' + t.title) + '.' + ext;
  }
  function save(bytes, name, mime) {
    const url = URL.createObjectURL(new Blob([bytes], {type:mime || 'application/octet-stream'}));
    const a = document.createElement('a'); a.href=url; a.download=name; document.body.append(a); a.click(); a.remove();
    setTimeout(() => URL.revokeObjectURL(url), 60000);
  }
  function sizedImage(url, size) { return url?.replace(/-(mini|tiny|small|badge|large|t\d+x\d+|crop|original)(?=\.[a-z]+(?:\?|$))/i, '-' + size); }
  function detect(buffer) {
    const b = new Uint8Array(buffer), s = (o,n) => String.fromCharCode(...b.slice(o,o+n));
    if (b[0]===0xff && b[1]===0xd8) return ['jpg','image/jpeg'];
    if (s(1,3)==='PNG') return ['png','image/png'];
    if (s(0,3)==='GIF') return ['gif','image/gif'];
    if (s(0,4)==='RIFF' && s(8,4)==='WEBP') return ['webp','image/webp'];
    if (s(0,4)==='RIFF' && s(8,4)==='WAVE') return ['wav','audio/wav'];
    if (s(0,4)==='FORM' && /AIFF|AIFC/.test(s(8,4))) return ['aiff','audio/aiff'];
    if (s(0,4)==='fLaC') return ['flac','audio/flac'];
    if (s(0,4)==='OggS') return ['ogg','audio/ogg'];
    if (s(4,4)==='ftyp') return ['m4a','audio/mp4'];
    if (b[0]===0xff && (b[1]&0xf6)===0xf0) return ['aac','audio/aac'];
    if (s(0,3)==='ID3' || (b[0]===0xff && (b[1]&0xe0)===0xe0 && (b[1]&6)!==0)) return ['mp3','audio/mpeg'];
    return null;
  }
  async function image(url, size) {
    if (!url) throw new Error('No image available.');
    let r;
    try { r = await request(sizedImage(url,size),true); } catch { r = await request(url,true); }
    const type = detect(r.body);
    if (!type || !type[1].startsWith('image/')) throw new Error('Image format not recognized.');
    return {bytes:r.body, ext:type[0], mime:type[1]};
  }
  const enc = s => new TextEncoder().encode(s);
  function join(...arrays) {
    const out = new Uint8Array(arrays.reduce((n,a) => n+a.length,0)); let p=0;
    for (const a of arrays) { out.set(a,p); p+=a.length; } return out;
  }
  const syncsafe = n => new Uint8Array([(n>>>21)&127,(n>>>14)&127,(n>>>7)&127,n&127]);
  function frame(id, payload) { return join(enc(id),syncsafe(payload.length),new Uint8Array(2),payload); }
  function tagMP3(raw, track, cover) {
    let b = new Uint8Array(raw);
    if (b.length>=10 && String.fromCharCode(...b.slice(0,3))==='ID3') {
      const n=((b[6]&127)<<21)|((b[7]&127)<<14)|((b[8]&127)<<7)|(b[9]&127);
      const end=10+n+(b[3]===4 && (b[5]&16) ? 10 : 0);
      if (end<=b.length) b=b.slice(end);
    }
    const frames=[frame('TIT2',join(new Uint8Array([3]),enc(track.title || 'Unknown'))),frame('TPE1',join(new Uint8Array([3]),enc(track.publisher_metadata?.artist || track.user?.username || 'Unknown')))];
    if (cover) frames.push(frame('APIC',join(new Uint8Array([3]),enc(cover.mime),new Uint8Array([0,3,0]),new Uint8Array(cover.bytes))));
    const payload=join(...frames);
    return join(new Uint8Array([73,68,51,4,0,0]),syncsafe(payload.length),payload,b);
  }
  async function downloadTrack(url, status) {
    status('Resolving track…'); const t=await resolve(url,'track');
    let source=null;
    if (t.downloadable && t.has_downloads_left !== false) {
      status('Checking original download…');
      try {
        const d=await api(`/tracks/${t.id}/download`,{secret_token:t.secret_token});
        if (d.redirectUri) source={url:d.redirectUri, original:true};
      } catch (e) { console.debug('[SCHQ] Original unavailable',e); }
    }
    if (!source) {
      const list=(t.media?.transcodings || []).filter(x => x.format?.protocol==='progressive' && !x.snipped);
      list.sort((a,b) => Number(b.quality==='hq')-Number(a.quality==='hq'));
      for (const tr of list) {
        try {
          const d=await api(tr.url,{track_authorization:t.track_authorization, secret_token:t.secret_token});
          if (d.url) { source={url:d.url, original:false}; break; }
        } catch (e) { console.debug('[SCHQ] Transcoding unavailable', e); }
      }
    }
    if (!source) throw new Error('No downloadable original or full progressive stream. HLS-only/restricted tracks are not supported.');
    status('Downloading audio…'); const r=await request(source.url,true);
    const type=detect(r.body);
    if (!type || !type[1].startsWith('audio/')) throw new Error('Unrecognized audio format; not saving with a guessed extension.');
    let bytes=r.body, embedded=false;
    if (!source.original && type[0]==='mp3') {
      status('Adding MP3 metadata / artwork…'); let cover=null;
      try { cover=await image(t.artwork_url || t.user?.avatar_url,'t500x500'); } catch {}
      bytes=tagMP3(bytes,t,cover); embedded=!!cover;
    }
    save(bytes,filename(t,type[0]),type[1]);
    status(`Saved ${type[0].toUpperCase()}${source.original ? ' · original, unchanged' : ' · stream'}${embedded ? ' · cover embedded' : ''}`);
  }
  async function downloadCover(url,status) {
    status('Resolving cover…'); const t=await resolve(url,'track');
    const im=await image(t.artwork_url || t.user?.avatar_url,'t500x500');
    save(im.bytes,filename(t,'').slice(0,-1)+' [cover].'+im.ext,im.mime); status('Cover saved.');
  }
  function visibleBannerURL() {
    const selectors = '.profileHeaderBackground__visual, .profileHeaderInfo__visual, .profileHeader__visual, [data-testid="profile-banner"]';
    for (const el of document.querySelectorAll(selectors)) {
      const bg = el.style.backgroundImage || getComputedStyle(el).backgroundImage;
      const match = bg.match(/url\(\s*(?:"([^"]*)"|'([^']*)'|([^)]*))\s*\)/i);
      const src = match ? (match[1] || match[2] || match[3]).trim() : el.querySelector('img')?.src;
      if (src) {
        try { const u = new URL(src,location.href); if (/^https?:$/.test(u.protocol)) return u.href; } catch {}
      }
    }
    return null;
  }
  async function downloadProfile(url, which, status) {
    const displayedBanner = which === 'banner' && profileURL() === url ? visibleBannerURL() : null;
    status(`Resolving ${which}…`);
    let user;
    try { user = await resolve(url,'user'); }
    catch (e) {
      if (!displayedBanner) throw e;
      user = {username: new URL(url).pathname.split('/').filter(Boolean)[0]};
    }
    let src = user.avatar_url;
    if (which === 'banner') {
      const visuals = user.visuals?.visuals || [];
      src = displayedBanner || visuals.find(v => v.visual_type === 'image')?.visual_url;
      if (!src) throw new Error('No banner found on this profile.');
    }
    status(`Downloading ${which}…`);
    const im = await image(src,'original');
    save(im.bytes,safe(user.username)+' ['+which+'].'+im.ext,im.mime);
    status(`${which === 'banner' ? 'Banner' : 'PFP'} saved.`);
  }

  const CSS=`
  #schq-page-tools,.schq-inline{font:12px/1.4 system-ui,sans-serif;box-sizing:border-box}
  #schq-page-tools{display:flex;align-items:center;flex-wrap:wrap;gap:6px;position:relative;clear:both;margin:10px 0;padding:4px 0;max-width:100%;flex:0 0 auto}
  #schq-page-tools button,.schq-inline button{appearance:none;background:transparent;color:#f50;border:1px solid #f50;border-radius:16px;padding:7px 12px;cursor:pointer;font:600 11px/1.3 system-ui,sans-serif;white-space:nowrap}
  #schq-page-tools button:hover,.schq-inline button:hover{background:#f50;color:#fff}
  #schq-page-tools button:disabled,.schq-inline button:disabled{opacity:.45;cursor:wait}
  #schq-page-tools.schq-profile-tools button{background:#222;color:#fff;border:1px solid #ddd;box-shadow:0 1px 4px #0006}
  #schq-page-tools.schq-profile-tools button:hover{background:#444;color:#fff;border-color:#fff}
  .schq-native-actions{flex-wrap:nowrap!important;overflow:visible!important}
  .schq-native-actions > #schq-page-tools{display:inline-flex!important;flex:0 0 auto!important;flex-wrap:nowrap;width:auto;min-width:76px;margin:0 0 0 4px!important;padding:0;gap:4px;align-self:center;overflow:visible}
  .schq-native-actions > #schq-page-tools button{display:inline-flex;align-items:center;justify-content:center;flex:0 0 36px;width:36px;height:36px;min-width:36px;padding:0;border-radius:50%;background:#222;color:#fff;border-color:#bbb}
  .schq-native-actions > #schq-page-tools button svg{width:19px;height:19px;pointer-events:none}
  .schq-native-actions > #schq-page-tools small:not(:empty){position:absolute;right:0;bottom:calc(100% + 8px);width:240px;max-width:70vw;padding:8px 10px;border-radius:6px;background:#222;color:#fff;box-shadow:0 2px 10px #0005;z-index:10;pointer-events:none}
  .schq-native-actions > #schq-page-tools button:hover{background:#444;border-color:#fff}
  #schq-page-tools small{flex-basis:100%;color:#999;overflow-wrap:anywhere}
  #schq-page-tools small:empty{display:none}
  .schq-inline{display:inline-flex;gap:5px;align-items:center;margin:5px;position:relative;z-index:2;flex-wrap:wrap}
  .schq-inline small{max-width:210px;color:#999;overflow-wrap:anywhere}
  `;
  function button(label, parent, work, status) {
    const b=parent.ownerDocument.createElement('button'); b.type='button'; b.textContent=label;
    b.addEventListener('click', async e => {
      e.preventDefault(); e.stopPropagation(); b.disabled=true;
      try { await work(status); } catch(err) { status(err.message); console.error('[SCHQ]',err); }
      finally { b.disabled=false; }
    }); parent.append(b); return b;
  }
  let pageTools, contextKey = '';
  function nativeTrackActions() {
    const ignored = 'aside, [role="dialog"], .playControls, .playbackSoundBadge, .soundList__item, .trackList__item, .trackItem, [data-testid="track-card"], [data-testid="track-item"]';
    for (const share of queryEverywhere('button[aria-label="Share" i]')) {
      if (share.closest(ignored)) continue;
      let row = share.parentElement;
      for (let depth=0; row && depth<3; depth++, row=row.parentElement) {
        if (row.matches('main, body, [role="main"]') || row.querySelector('h1')) break;
        if (!row.querySelector('button[aria-label="Copy link" i]') ||
            !row.querySelector('button[aria-label="More actions" i]')) continue;
        if (!row.getClientRects().length) break;
        return row;
      }
    }
    return null;
  }
  function pageMount(profile) {
    if (!profile) {
      const native = nativeTrackActions();
      if (native) return {host:native, native:true};
      for (const sel of ['.listenEngagement__footer', '[data-testid="track-actions"]', '.fullHero .soundActions']) {
        const host = queryEverywhere(sel).find(el => el.getClientRects().length);
        if (host) return {host, native:true};
      }
      return null;
    }
    for (const sel of ['.profileHeaderInfo', '.profileHeader', '[data-testid="profile-header"]']) {
      const host = document.querySelector(sel);
      if (host) return {host};
    }
    const bg = document.querySelector('.profileHeaderBackground__visual');
    if (bg?.parentElement) return {host:bg.parentElement, after:true};
    const main = document.querySelector('main, [role="main"], .l-main');
    return main ? {host:main, prepend:true} : null;
  }
  function clearPageTools() {
    pageTools?.parentElement?.classList.remove('schq-native-actions');
    pageTools?.remove();
  }
  function updatePageTools() {
    const page = trackURL(location.href), profile = profileURL();
    if (!page && !profile) { clearPageTools(); contextKey=''; return; }
    const mount = pageMount(profile);
    if (!mount) { clearPageTools(); return; }
    const key = JSON.stringify([page,profile]);
    if (!pageTools || key !== contextKey) {
      clearPageTools();
      pageTools = mount.host.ownerDocument.createElement('div'); pageTools.id = 'schq-page-tools';
      if (profile) pageTools.classList.add('schq-profile-tools');
      pageTools.setAttribute('aria-label','SoundCloud download tools');
      const st = mount.host.ownerDocument.createElement('small'); st.setAttribute('role','status'); st.setAttribute('aria-live','polite');
      let statusTimer;
      const status = t => {
        st.textContent=t; clearTimeout(statusTimer);
        statusTimer=setTimeout(()=>{st.textContent='';},12000);
      };
      if (page) {
        const dl = button('',pageTools,s=>downloadTrack(page,s),status);
        dl.title='Download track'; dl.setAttribute('aria-label','Download track');
        dl.innerHTML='<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" aria-hidden="true"><path d="M12 3v12m-5-5 5 5 5-5M4 16v5h16v-5"/></svg>';
        const art = button('',pageTools,s=>downloadCover(page,s),status);
        art.title='Download cover'; art.setAttribute('aria-label','Download cover');
        art.innerHTML='<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" aria-hidden="true"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8" cy="8" r="1.5"/><path d="m3 17 5-5 4 4 4-6 5 7"/></svg>';
      } else {
        button('↓ PFP',pageTools,s=>downloadProfile(profile,'pfp',s),status);
        button('↓ Banner',pageTools,s=>downloadProfile(profile,'banner',s),status);
      }
      pageTools.append(st); contextKey=key;
    }
    if (pageTools.parentElement && pageTools.parentElement !== mount.host) {
      pageTools.parentElement.classList.remove('schq-native-actions');
    }
    if (mount.native) mount.host.classList.add('schq-native-actions');
    if (mount.after) {
      if (mount.host.nextElementSibling !== pageTools) mount.host.insertAdjacentElement('afterend',pageTools);
    } else if (pageTools.parentElement !== mount.host) {
      if (mount.prepend) mount.host.prepend(pageTools); else mount.host.append(pageTools);
    }
  }
  const cardSelector='.soundList__item, .sound, .trackList__item, .trackItem, [data-testid="track-card"], [data-testid="track-item"]';
  function urlInCard(card) {
    const preferred=card.querySelectorAll('a.soundTitle__title, a.trackItem__trackTitle, .trackItem__trackTitle a, a[itemprop="url"], a[class*="Title"], a[class*="title"]');
    for(const a of preferred){ const u=trackURL(a.href);if(u)return u; }
    const urls=[...new Set([...card.querySelectorAll('a[href]')].map(a=>trackURL(a.href)).filter(Boolean))];
    return urls.length===1?urls[0]:null;
  }
  function inline(host,getURL, {cover = true} = {}) {
    if([...host.children].some(e=>e.classList.contains('schq-inline')))return;
    const wrap=host.ownerDocument.createElement('span');wrap.className='schq-inline';
    const st=host.ownerDocument.createElement('small');st.setAttribute('role','status');
    const status=t=>{st.textContent=t;};
    const run=fn=>async s=>{const url=getURL();if(!url)throw new Error('Track URL unavailable. Open the track page and retry.');await fn(url,s);};
    button('↓ DL',wrap,run(downloadTrack),status);
    if (cover) button('Cover',wrap,run(downloadCover),status);
    wrap.append(st);host.append(wrap);
  }
  function scan() {
    if(!document.body)return;
    watchDocuments();
    updatePageTools();
    const badge = document.querySelector('.playbackSoundBadge, [class*="playbackSoundBadge"]');
    if (badge && playerURL()) inline(badge,playerURL,{cover:false});
    queryEverywhere(cardSelector).forEach(card=>{
      if(card.querySelector(cardSelector) || !urlInCard(card))return;
      const host=card.querySelector('.soundActions, .sc-button-toolbar, [data-testid="track-actions"]');
      if(host)inline(host,()=>urlInCard(card));
    });
  }
  const watchedDocuments = new Map();
  let scanTimer;
  function scheduleScan() {
    if (!scanTimer) scanTimer = setTimeout(() => { scanTimer=null; scan(); },150);
  }
  function watchDocuments() {
    const live = new Set(accessibleDocuments());
    for (const [doc, observer] of watchedDocuments) {
      if (!live.has(doc)) {
        observer.disconnect(); doc.removeEventListener('load',scheduleScan,true);
        watchedDocuments.delete(doc);
      }
    }
    for (const doc of live) {
      if (!doc.documentElement) continue;
      if (!doc.getElementById('schq-styles')) {
        const style=doc.createElement('style'); style.id='schq-styles'; style.textContent=CSS;
        (doc.head || doc.documentElement).append(style);
      }
      if (watchedDocuments.has(doc)) continue;
      const observer = new MutationObserver(mutations => {
        if (mutations.some(m => !m.target.closest?.('#schq-page-tools,.schq-inline,#schq-styles'))) scheduleScan();
      });
      observer.observe(doc.documentElement,{
        childList:true,subtree:true,attributes:true,attributeFilter:['href','aria-label']
      });
      doc.addEventListener('load',scheduleScan,true);
      watchedDocuments.set(doc,observer);
    }
  }
  function start() {
    window.addEventListener('popstate',scheduleScan);
    setInterval(() => { if (document.visibilityState==='visible') scan(); },1500);
    scan();
  }
  if(document.readyState==='loading') document.addEventListener('DOMContentLoaded',start,{once:true});else start();
})();

```
</details>

thats it. it just adds cool feautres.

When downloading a track you also get the Cover, Name of uploader, Artist's name, Track name all on the .mp3 file.
