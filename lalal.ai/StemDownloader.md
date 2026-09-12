## Lalal.ai Stem downloader

**NOTE**: ALL EXPLANATION IS FOR PC ONLY, on mobile by default you shouldnt be able to get extension in the first place im pretty sure.
How to use this script:
1. Download the `Tampermonkey` Extension and search up a tutorial on how to set it up for your specific browser.
2. Add a new script (`Create a new script...`) and paste the following code in before saving:
<details>
	<summary>Click to expand</summary>
	
```js
// ==UserScript==
// @name         LALAL.ai Stem Downloader
// @namespace    stemdownloader
// @version      1.4.0
// @description  A compact, plain-language panel for saving captured LALAL.ai stems.
// @match        *://www.lalal.ai/*
// @match        *://lalal.ai/*
// @require      https://cdnjs.cloudflare.com/ajax/libs/lamejs/1.2.1/lame.min.js
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function () {
    'use strict';
    var JOIN_TRIM_MS = 10;
    var MP3_BITRATE_KBPS = 192;

    var trackGroups = Object.create(null);
    var trackOrder = [];
    var panel = null;
    var minimized = false;
    var detailsOpen = false;
    var audioContext = null;
    var statusMessage = '';
    var statusKind = '';
    var busy = false;
    var activeIndex = -1;
    var generation = 0;
    var renderTimer = null;

    var ICONS = {
        download: '<path d="M12 3v12m-4-4 4 4 4-4M4 16v4h16v-4"/>',
        wave: '<path d="M4 10v4m4-8v12m4-15v18m4-15v12m4-8v4"/>',
        minus: '<path d="M5 12h14"/>',
        plus: '<path d="M5 12h14M12 5v14"/>',
        chevron: '<path d="m9 5 7 7-7 7"/>',
        check: '<path d="m5 12 4 4L19 6"/>'
    };
    function icon(name) {
        return '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">' + ICONS[name] + '</svg>';
    }

    var CSS = `
#stem-dl-panel{--sd-yellow:#f8d937;--sd-text:#f5f4ef;--sd-muted:#aaa9a2;position:fixed;bottom:20px;right:20px;z-index:2147483647;width:320px;max-width:calc(100vw - 24px);max-height:calc(100vh - 40px);overflow:auto;box-sizing:border-box;padding:18px;background:#20211f;color:var(--sd-text);border:1px solid #44453e;border-radius:18px;box-shadow:0 12px 42px #0005;font:13px/1.45 system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;text-align:left;color-scheme:dark;isolation:isolate}
#stem-dl-panel *,#stem-dl-panel *::before,#stem-dl-panel *::after{box-sizing:border-box}
#stem-dl-panel button{appearance:none;margin:0;font:inherit;text-transform:none;letter-spacing:normal;cursor:pointer;transition:background .15s,border-color .15s;box-shadow:none}
#stem-dl-panel button:disabled{cursor:default;opacity:.45}
#stem-dl-panel button:focus-visible,#stem-dl-panel summary:focus-visible{outline:2px solid var(--sd-yellow);outline-offset:3px}
#stem-dl-panel svg{display:block;width:18px;height:18px;flex:none}
#stem-dl-panel .sd-header{display:flex;align-items:center;gap:10px}
#stem-dl-panel .sd-logo{display:grid;place-items:center;width:34px;height:34px;border:1px solid #615a2c;border-radius:10px;color:var(--sd-yellow);background:#343222;flex:none}
#stem-dl-panel .sd-heading{flex:1;min-width:0}
#stem-dl-panel .sd-title{font-size:15px;font-weight:650;letter-spacing:-.3px;line-height:1.2}
#stem-dl-panel .sd-subtitle{margin-top:3px;color:var(--sd-muted);font-size:10px;letter-spacing:1.2px;text-transform:uppercase}
#stem-dl-panel .sd-toggle{display:grid;place-items:center;width:28px;height:28px;padding:5px;border:0;border-radius:7px;background:transparent;color:#aaa9a2}
#stem-dl-panel .sd-toggle:hover{background:#363732;color:#fff}
#stem-dl-panel .sd-intro{margin:16px 0 13px;font-size:12px;color:#bbbcb4;line-height:1.5}
#stem-dl-panel .sd-list{display:flex;flex-direction:column;gap:7px;max-height:40vh;overflow:auto}
#stem-dl-panel .sd-row{display:flex;align-items:center;gap:10px;padding:11px 10px;background:#292a27;border:1px solid #3a3b35;border-radius:11px}
#stem-dl-panel .sd-number{display:grid;place-items:center;width:29px;height:32px;color:#8f9186;font:500 11px/1 ui-monospace,monospace;flex:none}
#stem-dl-panel .sd-track{min-width:0;flex:1}
#stem-dl-panel .sd-name{font-size:13px;font-weight:600;line-height:1.3}
#stem-dl-panel .sd-count{margin-top:3px;font-size:10px;color:var(--sd-muted)}
#stem-dl-panel .sd-save{display:flex;align-items:center;justify-content:center;gap:5px;min-width:65px;padding:7px 9px;background:#363730;color:#f1f1e9;border:1px solid #55564a;border-radius:7px;font-size:11px;font-weight:600;flex:none}
#stem-dl-panel .sd-save svg{width:14px;height:14px}
#stem-dl-panel .sd-save:hover:not(:disabled){background:#444639;border-color:#83866c}
#stem-dl-panel .sd-all{display:flex;justify-content:center;align-items:center;gap:8px;width:100%;margin-top:12px;padding:11px;background:var(--sd-yellow);color:#222313;border:1px solid var(--sd-yellow);border-radius:9px;font-size:12px;font-weight:700}
#stem-dl-panel .sd-all:hover:not(:disabled){background:#ffe76b;border-color:#ffe76b}
#stem-dl-panel .sd-footer{display:flex;justify-content:space-between;align-items:center;gap:8px;margin-top:14px;padding-top:12px;border-top:1px solid #3b3c35}
#stem-dl-panel .sd-format{display:inline-flex;align-items:center;gap:6px;color:var(--sd-muted);font-size:10px}
#stem-dl-panel .sd-format b{font-weight:600;font-size:9px;letter-spacing:.7px;color:#dbddd1;padding:2px 5px;background:#34362f;border:1px solid #4b4e40;border-radius:4px}
#stem-dl-panel .sd-reset{padding:3px 0;border:0;background:transparent;color:#aaa9a2;font-size:10px}
#stem-dl-panel .sd-reset:hover:not(:disabled){color:#fff;text-decoration:underline}
#stem-dl-panel details{margin-top:10px;color:var(--sd-muted);font-size:11px}
#stem-dl-panel summary{display:flex;align-items:center;gap:5px;list-style:none;cursor:pointer;width:fit-content;font-size:10px}
#stem-dl-panel summary::-webkit-details-marker{display:none}
#stem-dl-panel summary svg{width:11px;height:11px;transition:transform .15s}
#stem-dl-panel details[open] summary svg{transform:rotate(90deg)}
#stem-dl-panel .sd-help{padding:8px 0 0;line-height:1.6}
#stem-dl-panel .sd-help p{margin:0 0 7px}
#stem-dl-panel .sd-help p:last-child{margin-bottom:0}
#stem-dl-panel .sd-status{margin-top:12px;padding:8px 10px;background:#2c3027;border:1px solid #474f3c;border-radius:7px;color:#d5e5bd;font-size:11px;overflow-wrap:anywhere}
#stem-dl-panel .sd-status:empty{display:none}
#stem-dl-panel .sd-status[data-kind="error"]{background:#352b27;border-color:#654739;color:#ffc6a4}
#stem-dl-panel .sd-empty{padding:22px 12px;text-align:center;border:1px dashed #505144;border-radius:11px;background:#252620}
#stem-dl-panel .sd-empty-icon{display:flex;justify-content:center;margin-bottom:10px;color:var(--sd-yellow)}
#stem-dl-panel .sd-empty strong{display:block;font-size:12px;font-weight:600}
#stem-dl-panel .sd-empty p{margin:5px 0 0;color:var(--sd-muted);font-size:11px}
#stem-dl-panel.sd-minimized{width:190px;padding:11px 13px;border-radius:13px}
#stem-dl-panel.sd-minimized .sd-logo{width:29px;height:29px;border-radius:8px}
#stem-dl-panel.sd-minimized .sd-title{font-size:13px}
#stem-dl-panel.sd-minimized .sd-subtitle{font-size:9px;letter-spacing:0;text-transform:none}
@media(max-width:480px){#stem-dl-panel{right:12px;bottom:12px}}
@media(prefers-reduced-motion:reduce){#stem-dl-panel *{transition:none!important}}
`;

    function getLame() {
        return typeof lamejs !== 'undefined' ? lamejs : window.lamejs;
    }
    function hasMP3() { var lib = getLame(); return !!(lib && lib.Mp3Encoder); }
    function make(tag, className, text) {
        var el = document.createElement(tag);
        if (className) el.className = className;
        if (text != null) el.textContent = text;
        return el;
    }
    function makeButton(label, className, action, iconName) {
        var b = make('button', className);
        b.type = 'button';
        if (iconName) b.innerHTML = icon(iconName);
        if (label) b.appendChild(make('span', '', label));
        b.addEventListener('click', function(e) { e.preventDefault(); e.stopPropagation(); action(); });
        return b;
    }
    function ensurePanel() {
        if (!document.body) return false;
        if (!document.getElementById('stem-dl-style')) {
            var style = make('style'); style.id = 'stem-dl-style'; style.textContent = CSS;
            (document.head || document.documentElement).appendChild(style);
        }
        if (panel && panel.isConnected) return true;
        panel = make('section'); panel.id = 'stem-dl-panel';
        panel.setAttribute('aria-label', 'Stem downloads');
        document.body.appendChild(panel);
        return true;
    }
    function setStatus(message, kind) {
        statusMessage = message || ''; statusKind = kind || '';
        refreshPanel();
    }
    function queueRefresh() {
        if (renderTimer) return;
        renderTimer = setTimeout(function() { renderTimer = null; refreshPanel(); }, 100);
    }
    function refreshPanel() {
        if (!ensurePanel()) return;
        var focused = panel.contains(document.activeElement) ? document.activeElement.dataset.focus : null;
        panel.replaceChildren();
        panel.classList.toggle('sd-minimized', minimized);
        var header = make('div', 'sd-header');
        var logo = make('div', 'sd-logo'); logo.innerHTML = icon('wave'); header.appendChild(logo);
        var heading = make('div', 'sd-heading');
        heading.appendChild(make('div', 'sd-title', 'Your stems'));
        heading.appendChild(make('div', 'sd-subtitle', minimized ? (busy ? 'Saving…' : trackOrder.length + ' collected') : 'Audio downloads'));
        header.appendChild(heading);
        var toggle = makeButton('', 'sd-toggle', function() { minimized = !minimized; refreshPanel(); }, minimized ? 'plus' : 'minus');
        toggle.setAttribute('aria-label', minimized ? 'Expand downloads' : 'Minimize downloads');
        toggle.setAttribute('aria-expanded', String(!minimized)); toggle.dataset.focus = 'toggle';
        header.appendChild(toggle); panel.appendChild(header);
        if (!minimized) {
            if (!trackOrder.length) {
                panel.appendChild(make('p', 'sd-intro', 'Save the audio you play, in one place.'));
                var empty = make('div', 'sd-empty');
                var emptyIcon = make('div', 'sd-empty-icon'); emptyIcon.innerHTML = icon('wave'); empty.appendChild(emptyIcon);
                empty.appendChild(make('strong', '', 'Press play to get started'));
                empty.appendChild(make('p', '', 'Your stems will appear here.'));
                panel.appendChild(empty);
            } else {
                panel.appendChild(make('p', 'sd-intro', 'Play the full audio before saving.'));
                var list = make('div', 'sd-list');
                trackOrder.forEach(function(base, i) {
                    var count = Object.keys(trackGroups[base] || {}).length;
                    var row = make('div', 'sd-row');
                    row.appendChild(make('span', 'sd-number', String(i + 1).padStart(2, '0')));
                    var track = make('div', 'sd-track');
                    track.appendChild(make('div', 'sd-name', 'Stem ' + (i + 1)));
                    track.appendChild(make('div', 'sd-count', count + (count === 1 ? ' part collected' : ' parts collected')));
                    row.appendChild(track);
                    var save = makeButton(busy && activeIndex === i ? 'Saving…' : 'Save', 'sd-save', function() { downloadTrack(i); }, 'download');
                    save.disabled = busy || !count; save.dataset.focus = 'save-' + i;
                    save.setAttribute('aria-label', 'Save Stem ' + (i + 1)); row.appendChild(save); list.appendChild(row);
                });
                panel.appendChild(list);
                if (trackOrder.length > 1) {
                    var all = makeButton(busy ? 'Saving…' : 'Save all', 'sd-all', downloadAll, 'download');
                    all.disabled = busy; all.dataset.focus = 'all'; panel.appendChild(all);
                }
            }
            var footer = make('div', 'sd-footer');
            var format = make('span', 'sd-format', 'File type ');
            format.appendChild(make('b', '', hasMP3() ? 'MP3' : 'WAV')); footer.appendChild(format);
            if (trackOrder.length) {
                var reset = makeButton('Clear list', 'sd-reset', function() {
                    if (busy || !confirm('Clear the collected audio? Play it again to collect it.')) return;
                    generation++; trackOrder.length = 0; trackGroups = Object.create(null); setStatus('');
                });
                reset.disabled = busy; reset.dataset.focus = 'clear'; footer.appendChild(reset);
            }
            panel.appendChild(footer);
            var details = make('details'); details.open = detailsOpen;
            var summary = make('summary'); summary.innerHTML = icon('chevron'); summary.appendChild(document.createTextNode('Details')); summary.dataset.focus = 'details'; details.appendChild(summary);
            var help = make('div', 'sd-help');
            help.appendChild(make('p', '', 'Stems are numbered as they arrive. Listen to tell which one is vocals, music, or another part.'));
            help.appendChild(make('p', '', 'Only collected audio is saved. The part count does not confirm that the whole track is here.'));
            help.appendChild(make('p', '', hasMP3() ? 'Files save as 192 kbps MP3.' : 'MP3 export is unavailable, so files save as WAV. WAV does not improve the source quality.'));
            help.appendChild(make('p', '', 'The same join setting is used as before. If a join sounds cut off, the 10 ms trim can be changed in the script.'));
            details.appendChild(help); details.addEventListener('toggle', function() { detailsOpen = details.open; }); panel.appendChild(details);
            var status = make('div', 'sd-status', statusMessage); status.dataset.kind = statusKind;
            status.setAttribute('role', 'status'); status.setAttribute('aria-live', 'polite'); panel.appendChild(status);
        }
        if (focused) {
            var target = panel.querySelector('[data-focus="' + focused + '"]');
            if (target && !target.disabled) target.focus({preventScroll:true});
        }
    }

    function parseSegmentURL(url) {
        var clean = String(url || '').split('?')[0].split('#')[0];
        var match = clean.match(/(.*\/)segment-(\d+)\.mp3$/i);
        return match ? {base:match[1], num:parseInt(match[2], 10)} : null;
    }
    function isSegmentURL(url) { return /segment-\d+\.mp3(?:[?#]|$)/i.test(String(url || '')); }
    function getAudioContext() {
        if (!audioContext) {
            var Ctor = window.AudioContext || window.webkitAudioContext;
            if (!Ctor) throw new Error('Audio processing is not supported in this browser.');
            audioContext = new Ctor();
        }
        return audioContext;
    }
    function decodeAudio(ctx, buffer) {
        return new Promise(function(resolve, reject) {
            try {
                var result = ctx.decodeAudioData(buffer, resolve, reject);
                if (result && typeof result.then === 'function') result.then(resolve, reject);
            } catch (e) { reject(e); }
        });
    }
    async function decodeAndJoin(base) {
        var segments = trackGroups[base] || {};
        var nums = Object.keys(segments).map(Number).filter(Number.isFinite).sort(function(a,b) { return a-b; });
        if (!nums.length) throw new Error('Play this stem first, then try again.');
        var raw = nums.map(function(n) { return segments[n].slice(0); });
        var ctx = getAudioContext();
        if (ctx.state === 'suspended') { try { await ctx.resume(); } catch (e) {} }
        var decoded = await Promise.all(raw.map(function(b) { return decodeAudio(ctx,b); }));
        var sampleRate = decoded[0].sampleRate;
        if (decoded.some(function(b) { return b.sampleRate !== sampleRate; })) throw new Error('These audio parts could not be joined.');
        var channels = decoded.some(function(b) { return b.numberOfChannels > 1; }) ? 2 : 1;
        var trimFrames = Math.round(sampleRate * JOIN_TRIM_MS / 1000);
        var totalFrames = 0;
        var pieces = decoded.map(function(b, i) {
            var start = i === 0 ? 0 : Math.min(trimFrames, b.length);
            var length = b.length-start; totalFrames += length;
            return {buffer:b, start:start, length:length};
        });
        var pcm = Array.from({length:channels}, function() { return new Float32Array(totalFrames); });
        var writeAt = 0;
        pieces.forEach(function(piece) {
            for (var ch = 0; ch < channels; ch++) {
                var source = piece.buffer.getChannelData(Math.min(ch,piece.buffer.numberOfChannels-1));
                pcm[ch].set(source.subarray(piece.start,piece.start+piece.length),writeAt);
            }
            writeAt += piece.length;
        });
        return {pcm:pcm, sampleRate:sampleRate, channels:channels};
    }
    function floatToInt16(data) {
        var out = new Int16Array(data.length);
        for (var i=0;i<data.length;i++) {
            var value = Math.max(-1,Math.min(1,data[i])); out[i] = value < 0 ? value*32768 : value*32767;
        }
        return out;
    }
    function encodeMP3(joined) {
        var lib = getLame(); if (!lib || !lib.Mp3Encoder) return null;
        var encoder = new lib.Mp3Encoder(joined.channels,joined.sampleRate,MP3_BITRATE_KBPS);
        var left = floatToInt16(joined.pcm[0]);
        var right = joined.channels > 1 ? floatToInt16(joined.pcm[1]) : null;
        var chunks = [];
        for (var i=0;i<left.length;i+=1152) {
            var end = Math.min(i+1152,left.length);
            var data = right ? encoder.encodeBuffer(left.subarray(i,end),right.subarray(i,end)) : encoder.encodeBuffer(left.subarray(i,end));
            if (data.length) chunks.push(new Int8Array(data));
        }
        var last = encoder.flush(); if (last.length) chunks.push(new Int8Array(last));
        return new Blob(chunks,{type:'audio/mpeg'});
    }
    function encodeWAV(joined) {
        var channels = joined.channels, frames = joined.pcm[0].length;
        var blockAlign = channels*2, dataSize = frames*blockAlign;
        if (dataSize > 0xffffffff-36) throw new Error('This audio is too large for one WAV file.');
        var buffer = new ArrayBuffer(44+dataSize), view = new DataView(buffer);
        function ascii(offset,text) { for (var i=0;i<text.length;i++) view.setUint8(offset+i,text.charCodeAt(i)); }
        ascii(0,'RIFF');view.setUint32(4,36+dataSize,true);ascii(8,'WAVE');ascii(12,'fmt ');
        view.setUint32(16,16,true);view.setUint16(20,1,true);view.setUint16(22,channels,true);
        view.setUint32(24,joined.sampleRate,true);view.setUint32(28,joined.sampleRate*blockAlign,true);
        view.setUint16(32,blockAlign,true);view.setUint16(34,16,true);ascii(36,'data');view.setUint32(40,dataSize,true);
        var offset=44;
        for (var i=0;i<frames;i++) for (var ch=0;ch<channels;ch++) {
            var value=Math.max(-1,Math.min(1,joined.pcm[ch][i]));
            view.setInt16(offset,value<0?value*32768:value*32767,true);offset+=2;
        }
        return new Blob([buffer],{type:'audio/wav'});
    }
    function saveBlob(blob,filename) {
        var href=URL.createObjectURL(blob), a=make('a');a.href=href;a.download=filename;
        document.body.appendChild(a);a.click();a.remove();
        setTimeout(function() { URL.revokeObjectURL(href); },60000);
    }
    async function exportTrack(idx) {
        var base=trackOrder[idx]; if (!base || !trackGroups[base]) return false;
        activeIndex=idx;setStatus('Preparing Stem '+(idx+1)+'…');
        await new Promise(function(resolve) { setTimeout(resolve,30); });
        try {
            var joined=await decodeAndJoin(base), blob=encodeMP3(joined), ext='mp3';
            if (!blob) { blob=encodeWAV(joined);ext='wav'; }
            saveBlob(blob,'stem_'+(idx+1)+'.'+ext);
            setStatus('Download started for Stem '+(idx+1)+'.');return true;
        } catch (err) {
            console.error('[Stem Downloader]',err);
            setStatus('Couldn’t save Stem '+(idx+1)+'. '+(err.message || 'Try playing it again.'),'error');return false;
        }
    }
    async function downloadTrack(idx) {
        if (busy) return;busy=true;
        try { await exportTrack(idx); }
        finally { busy=false;activeIndex=-1;refreshPanel(); }
    }
    async function downloadAll() {
        if (busy) return;busy=true;var count=trackOrder.length, saved=0;
        try {
            for (var i=0;i<count;i++) if (await exportTrack(i)) saved++;
            setStatus(saved===count ? 'Downloads started. Allow multiple downloads if your browser asks.' : saved+' of '+count+' downloads started. Try the others again.', saved===count ? '' : 'error');
        } finally { busy=false;activeIndex=-1;refreshPanel(); }
    }
    function captureSegment(url,getBuffer) {
        var parsed=parseSegmentURL(url);if (!parsed) return;
        var token=generation;
        getBuffer().then(function(buffer) {
            if (token!==generation || !buffer || !buffer.byteLength) return;
            if (!trackGroups[parsed.base]) {trackGroups[parsed.base]=Object.create(null);trackOrder.push(parsed.base);}
            trackGroups[parsed.base][parsed.num]=buffer;queueRefresh();
        }).catch(function() {});
    }
    var origFetch=window.fetch;
    if (origFetch) window.fetch=function() {
        var args=arguments,input=args[0];
        var url=typeof input==='string'?input:((input && input.url) || String(input || ''));
        return origFetch.apply(this,args).then(function(response) {
            if (response.ok && isSegmentURL(url)) {
                try { var clone=response.clone();captureSegment(url,function() {return clone.arrayBuffer();}); } catch (e) {}
            }
            return response;
        });
    };
    var origOpen=XMLHttpRequest.prototype.open,origSend=XMLHttpRequest.prototype.send;
    XMLHttpRequest.prototype.open=function(method,url) {this._stemUrl=String(url || '');return origOpen.apply(this,arguments);};
    XMLHttpRequest.prototype.send=function() {
        var self=this,url=self._stemUrl;
        if (isSegmentURL(url)) {
            try { if (self.responseType==='' || self.responseType==='text') self.responseType='arraybuffer'; } catch (e) {}
            self.addEventListener('load',function() {
                if (self.status<200 || self.status>=300) return;
                var response=self.response;
                if (response instanceof ArrayBuffer) captureSegment(url,function() {return Promise.resolve(response.slice(0));});
                else if (response instanceof Blob) captureSegment(url,function() {return response.arrayBuffer();});
            },{once:true});
        }
        return origSend.apply(this,arguments);
    };
    if (document.readyState==='loading') document.addEventListener('DOMContentLoaded',refreshPanel,{once:true});
    else refreshPanel();
    setInterval(function() { if (document.body && (!panel || !panel.isConnected)) refreshPanel(); },1000);
})();

```
</details>

thats about it gng
