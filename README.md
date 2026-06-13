# n8n-youtube-animation-automation
Koleksi workflow n8n untuk YouTube Shorts &amp; AI Animation Automation

clipper-v4-final.json
clipper-v3-final.json
clipper-simple.json
dazn-fury-hall.json
n8n-clipper-final.json :

{ "name": "AI Clipper — YouTube to Shorts", "nodes": [ { "name": "Telegram Trigger", "type": "n8n-nodes-base.telegramTrigger", "parameters": { "updates": ["message"], "additionalFields": {} }, "credentials": { "telegramApi": { "id": "1", "name": "Telegram Bot" } }, "position": [100, 300] }, { "name": "Extract YouTube URL", "type": "n8n-nodes-base.set", "parameters": { "values": { "string": [{ "name": "youtubeUrl", "value": "={{$json.message.text}}" }] } }, "position": [300, 300] }, { "name": "Log to Google Sheets", "type": "n8n-nodes-base.googleSheets", "parameters": { "operation": "append", "sheetId": "YOUR_SHEET_ID", "range": "Queue!A:D", "dataMode": "autoMapInputData", "options": {} }, "position": [500, 300] }, { "name": "Download Video (yt-dlp)", "type": "n8n-nodes-base.executeCommand", "parameters": { "command": "yt-dlp -f 'bestvideo[height<=720]+bestaudio/best[height<=720]' --merge-output-format mp4 -o '/root/clipper/downloads/%(id)s.%(ext)s' '={{$json.youtubeUrl}}' && echo 'DONE:' $(ls /root/clipper/downloads/*.mp4 | tail -1)" }, "position": [700, 300] }, { "name": "Extract Audio for Transcription", "type": "n8n-nodes-base.executeCommand", "parameters": { "command": "VIDEO=$(ls /root/clipper/downloads/*.mp4 | tail -1); ffmpeg -i \"$VIDEO\" -vn -ar 16000 -ac 1 -b:a 64k /root/clipper/downloads/audio.mp3 -y && echo $VIDEO" }, "position": [900, 300] }, { "name": "Groq Whisper Transcribe", "type": "n8n-nodes-base.httpRequest", "parameters": { "method": "POST", "url": "https://api.groq.com/openai/v1/audio/transcriptions", "authentication": "genericCredentialType", "genericAuthType": "httpHeaderAuth", "sendHeaders": true, "headerParameters": { "parameters": [{ "name": "Authorization", "value": "Bearer YOUR_GROQ_API_KEY" }] }, "sendBody": true, "contentType": "multipart-form-data", "bodyParameters": { "parameters": [ { "name": "file", "value": "={{ $binary.data }}", "parameterType": "formBinaryData" }, { "name": "model", "value": "whisper-large-v3" }, { "name": "response_format", "value": "verbose_json" }, { "name": "timestamp_granularities[]", "value": "segment" } ] } }, "position": [1100, 300] }, { "name": "Gemini — Find Viral Clips", "type": "n8n-nodes-base.httpRequest", "parameters": { "method": "POST", "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=YOUR_GEMINI_API_KEY", "sendBody": true, "bodyParameters": { "parameters": [{ "name": "contents", "value": "=[{\"parts\":[{\"text\": \"Kamu adalah ahli viral content. Analisis transkrip ini dan temukan 5-8 momen paling viral untuk dijadikan video pendek 30-90 detik. Untuk setiap clip, berikan: start_time (detik), end_time (detik), hook (kalimat pembuka 5 kata pertama yang bikin orang berhenti scroll), viral_score (1-10), caption_tiktok (dengan 5 hashtag). Transkrip: \" + $json.text + \". Balas HANYA JSON array.\"}]}]" }] } }, "position": [1300, 300] }, { "name": "Parse Clips JSON", "type": "n8n-nodes-base.code", "parameters": { "jsCode": "const raw = $input.first().json.candidates[0].content.parts[0].text;\nconst clean = raw.replace(/```json|```/g,'').trim();\nconst clips = JSON.parse(clean);\nreturn clips.map((c,i) => ({ json: { ...c, index: i, videoFile: $('Download Video (yt-dlp)').first().json.stdout.replace('DONE: ','').trim() } }));" }, "position": [1500, 300] }, { "name": "FFmpeg Cut + Crop + Caption", "type": "n8n-nodes-base.executeCommand", "parameters": { "command": "START={{$json.start_time}}; END={{$json.end_time}}; INPUT='{{$json.videoFile}}'; OUT='/root/clipper/clips/clip_{{$json.index}}.mp4'; DUR=$((END-START)); ffmpeg -ss $START -t $DUR -i \"$INPUT\" -vf \"crop=ih*9/16:ih,scale=1080:1920,drawtext=text='{{$json.hook}}':fontsize=52:fontcolor=white:x=(w-text_w)/2:y=h*0.15:box=1:boxcolor=black@0.5:boxborderw=15\" -c:v libx264 -preset fast -crf 23 -c:a aac -b:a 128k -movflags +faststart \"$OUT\" -y && echo \"CLIP_DONE:$OUT\"" }, "position": [1700, 300] }, { "name": "Send to Telegram", "type": "n8n-nodes-base.telegram", "parameters": { "operation": "sendDocument", "chatId": "YOUR_CHAT_ID", "binaryData": true, "binaryProperty": "data", "additionalFields": { "caption": "🎬 Clip {{$json.index+1}}\n⚡ Hook: {{$json.hook}}\n📊 Viral Score: {{$json.viral_score}}/10\n\n📱 Caption:\n{{$json.caption_tiktok}}" } }, "position": [1900, 300] }, { "name": "Update Sheet — Done", "type": "n8n-nodes-base.googleSheets", "parameters": { "operation": "update", "sheetId": "YOUR_SHEET_ID", "range": "Queue!E:E", "dataMode": "autoMapInputData" }, "position": [2100, 300] } ], "connections": { "Telegram Trigger": { "main": [[{"node": "Extract YouTube URL", "type": "main", "index": 0}]] }, "Extract YouTube URL": { "main": [[{"node": "Log to Google Sheets", "type": "main", "index": 0}]] }, "Log to Google Sheets": { "main": [[{"node": "Download Video (yt-dlp)", "type": "main", "index": 0}]] }, "Download Video (yt-dlp)": { "main": [[{"node": "Extract Audio for Transcription", "type": "main", "index": 0}]] }, "Extract Audio for Transcription": { "main": [[{"node": "Groq Whisper Transcribe", "type": "main", "index": 0}]] }, "Groq Whisper Transcribe": { "main": [[{"node": "Gemini — Find Viral Clips", "type": "main", "index": 0}]] }, "Gemini — Find Viral Clips": { "main": [[{"node": "Parse Clips JSON", "type": "main",  "index": 0}]] }, "Parse Clips JSON": { "main": [[{"node": "FFmpeg Cut + Crop + Caption", "type": "main", "index": 0}]] }, "FFmpeg Cut + Crop + Caption": { "main": [[{"node": "Send to Telegram", "type": "main", "index": 0}]] }, "Send to Telegram": { "main": [[{"node": "Update Sheet — Done", "type": "main", "index": 0}]] } } } 

set up guide : 


<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AI Clipper v2 — Auto Source + Full Pipeline</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@300;400;500;600;700&family=JetBrains+Mono:wght@400;500;700&display=swap');
  :root {
    --bg:#0a0a0f;--surface:#111118;--surface2:#1a1a24;--border:#2a2a3a;
    --accent:#7c3aed;--accent2:#06b6d4;--accent3:#f59e0b;
    --green:#10b981;--red:#ef4444;--text:#e8e8f0;--muted:#6b7280;
    --mono:'JetBrains Mono',monospace;--sans:'Space Grotesk',sans-serif;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  body{background:var(--bg);color:var(--text);font-family:var(--sans);min-height:100vh;overflow-x:hidden}

  /* HERO */
  .hero{position:relative;padding:44px 20px 36px;text-align:center;overflow:hidden}
  .hero::before{content:'';position:absolute;top:-60px;left:50%;transform:translateX(-50%);width:700px;height:320px;background:radial-gradient(ellipse,rgba(124,58,237,0.16) 0%,transparent 70%);pointer-events:none}
  .hero-tag{display:inline-block;background:rgba(124,58,237,0.15);border:1px solid rgba(124,58,237,0.4);color:#a78bfa;font-size:11px;font-weight:600;letter-spacing:2px;text-transform:uppercase;padding:5px 14px;border-radius:20px;margin-bottom:18px}
  .hero h1{font-size:clamp(24px,6vw,50px);font-weight:700;line-height:1.1;margin-bottom:10px;letter-spacing:-0.02em}
  .hero h1 span{background:linear-gradient(135deg,#7c3aed,#06b6d4);-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}
  .hero p{color:var(--muted);font-size:14px;max-width:500px;margin:0 auto 24px;line-height:1.6}
  .hero-stats{display:flex;gap:10px;justify-content:center;flex-wrap:wrap}
  .stat-pill{background:var(--surface);border:1px solid var(--border);border-radius:8px;padding:10px 16px;text-align:center}
  .stat-pill .num{font-size:20px;font-weight:700;color:var(--accent2);font-family:var(--mono)}
  .stat-pill .lbl{font-size:10px;color:var(--muted);letter-spacing:1px;text-transform:uppercase;margin-top:2px}

  /* VERSION BADGE */
  .v2-badge{display:inline-flex;align-items:center;gap:6px;background:linear-gradient(135deg,rgba(124,58,237,0.2),rgba(6,182,212,0.2));border:1px solid rgba(124,58,237,0.4);border-radius:6px;padding:4px 12px;font-size:11px;font-weight:600;color:#a78bfa;margin-bottom:8px}

  /* TABS */
  .tabs-wrap{position:sticky;top:0;z-index:100;background:rgba(10,10,15,0.94);backdrop-filter:blur(12px);border-bottom:1px solid var(--border);padding:0 16px;overflow-x:auto;scrollbar-width:none}
  .tabs-wrap::-webkit-scrollbar{display:none}
  .tabs{display:flex;gap:0;min-width:max-content}
  .tab-btn{background:none;border:none;color:var(--muted);font-family:var(--sans);font-size:13px;font-weight:500;padding:13px 16px;cursor:pointer;border-bottom:2px solid transparent;transition:all 0.2s;white-space:nowrap}
  .tab-btn.active{color:var(--text);border-bottom-color:var(--accent)}
  .tab-btn:hover:not(.active){color:var(--text)}

  /* CONTENT */
  .content{padding:22px 16px 60px;max-width:720px;margin:0 auto}
  .tab-panel{display:none}.tab-panel.active{display:block}

  /* SECTION TITLE */
  .section-title{font-size:11px;font-weight:600;letter-spacing:2px;text-transform:uppercase;color:var(--accent);margin-bottom:14px;margin-top:28px}
  .section-title:first-child{margin-top:0}

  /* CARD */
  .card{background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:16px;margin-bottom:10px}
  .card-header{display:flex;align-items:center;gap:12px;margin-bottom:12px}
  .card-icon{width:34px;height:34px;border-radius:8px;display:flex;align-items:center;justify-content:center;font-size:15px;flex-shrink:0}
  .card-title{font-size:14px;font-weight:600}
  .card-sub{font-size:11px;color:var(--muted);margin-top:2px}

  /* PIPELINE */
  .pipeline{display:flex;flex-direction:column;gap:0;margin:10px 0}
  .pipe-node{background:var(--surface2);border:1px solid var(--border);border-radius:10px;padding:12px 14px;display:flex;align-items:center;gap:12px;position:relative}
  .pipe-node.source{border-color:rgba(16,185,129,0.4);background:rgba(16,185,129,0.05)}
  .pipe-node.process{border-color:rgba(124,58,237,0.3);background:rgba(124,58,237,0.04)}
  .pipe-node.output{border-color:rgba(6,182,212,0.4);background:rgba(6,182,212,0.05)}
  .pipe-arrow{text-align:center;color:var(--border);font-size:12px;padding:4px 0;line-height:1}
  .pipe-emoji{font-size:16px;flex-shrink:0}
  .pipe-info{flex:1}
  .pipe-name{font-size:13px;font-weight:600;color:var(--text);margin-bottom:2px}
  .pipe-detail{font-size:11px;color:var(--muted);line-height:1.5}
  .pipe-badge{font-size:10px;padding:2px 7px;border-radius:4px;font-weight:600;white-space:nowrap;flex-shrink:0}
  .pb-free{background:rgba(16,185,129,0.15);color:#10b981}
  .pb-api{background:rgba(245,158,11,0.15);color:#f59e0b}
  .pb-new{background:rgba(124,58,237,0.2);color:#a78bfa;border:1px solid rgba(124,58,237,0.3)}

  /* FLOW STEP */
  .flow-step{display:flex;gap:14px;position:relative}
  .flow-line{display:flex;flex-direction:column;align-items:center;flex-shrink:0}
  .step-dot{width:28px;height:28px;border-radius:50%;background:var(--surface2);border:2px solid var(--accent);display:flex;align-items:center;justify-content:center;font-size:11px;font-weight:700;font-family:var(--mono);color:var(--accent);flex-shrink:0;z-index:1}
  .connector{width:2px;background:linear-gradient(to bottom,var(--accent),var(--border));flex:1;min-height:16px;margin:4px 0}
  .flow-body{padding-bottom:18px;flex:1}
  .flow-title{font-size:14px;font-weight:600;margin-bottom:5px;padding-top:4px}
  .flow-desc{font-size:13px;color:#9090a8;line-height:1.6}

  /* CODE */
  .code-block{background:#0a0a12;border:1px solid var(--border);border-radius:8px;padding:14px;font-family:var(--mono);font-size:12px;color:#a0e0b0;overflow-x:auto;margin:10px 0;line-height:1.9;position:relative;white-space:pre}
  .comment{color:#3d4a5c}.keyword{color:#7c3aed}.string{color:#f59e0b}.cmd{color:#06b6d4}.hi{color:#10b981;font-weight:600}
  .copy-btn{position:absolute;top:8px;right:8px;background:var(--surface2);border:1px solid var(--border);color:var(--muted);font-size:10px;font-family:var(--sans);padding:4px 8px;border-radius:4px;cursor:pointer;transition:all 0.2s;z-index:2}
  .copy-btn:hover{color:var(--text);border-color:var(--accent)}

  /* BADGE */
  .badge{display:inline-block;font-size:10px;font-weight:600;padding:2px 8px;border-radius:4px;letter-spacing:0.5px;margin-left:5px;vertical-align:middle}
  .badge-free{background:rgba(16,185,129,0.15);color:#10b981;border:1px solid rgba(16,185,129,0.3)}
  .badge-new{background:rgba(124,58,237,0.15);color:#a78bfa;border:1px solid rgba(124,58,237,0.3)}
  .badge-key{background:rgba(245,158,11,0.15);color:#f59e0b;border:1px solid rgba(245,158,11,0.3)}

  /* ALERT */
  .alert{border-radius:8px;padding:13px 15px;font-size:13px;line-height:1.6;margin-bottom:10px;display:flex;gap:10px}
  .alert-tip{background:rgba(124,58,237,0.08);border:1px solid rgba(124,58,237,0.25);color:#c4b5fd}
  .alert-warn{background:rgba(245,158,11,0.08);border:1px solid rgba(245,158,11,0.25);color:#fbbf24}
  .alert-success{background:rgba(16,185,129,0.08);border:1px solid rgba(16,185,129,0.25);color:#34d399}
  .alert-info{background:rgba(6,182,212,0.08);border:1px solid rgba(6,182,212,0.25);color:#67e8f9}
  .alert-icon{font-size:15px;flex-shrink:0;margin-top:1px}

  /* TABLE */
  .spec-row{display:flex;justify-content:space-between;align-items:flex-start;font-size:12px;padding:7px 0;border-bottom:1px solid rgba(255,255,255,0.05)}
  .spec-row:last-child{border-bottom:none}
  .spec-label{color:var(--muted);flex-shrink:0;min-width:110px}
  .spec-val{color:var(--text);font-family:var(--mono);font-size:11px;text-align:right}

  /* TOGGLE */
  .toggle-section{margin-bottom:8px}
  .toggle-header{background:var(--surface);border:1px solid var(--border);border-radius:10px;padding:13px 15px;cursor:pointer;display:flex;align-items:center;justify-content:space-between;font-size:13px;font-weight:600;transition:border-color 0.2s;user-select:none}
  .toggle-header:hover{border-color:var(--accent)}
  .toggle-header.open{border-bottom-left-radius:0;border-bottom-right-radius:0;border-bottom-color:transparent}
  .toggle-arrow{transition:transform 0.2s;font-size:11px;color:var(--muted)}
  .toggle-header.open .toggle-arrow{transform:rotate(180deg)}
  .toggle-body{display:none;background:var(--surface);border:1px solid var(--border);border-top:none;border-bottom-left-radius:10px;border-bottom-right-radius:10px;padding:14px}
  .toggle-body.open{display:block}

  /* CHANNEL GRID */
  .channel-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin:10px 0}
  .channel-item{background:var(--surface2);border:1px solid var(--border);border-radius:8px;padding:10px 12px}
  .ch-name{font-size:12px;font-weight:600;margin-bottom:3px}
  .ch-id{font-size:10px;color:var(--muted);font-family:var(--mono);word-break:break-all;line-height:1.4}

  /* PROMPT */
  .prompt-box{background:#09090f;border:1px solid rgba(124,58,237,0.3);border-left:3px solid var(--accent);border-radius:8px;padding:13px 15px;font-size:12px;color:#c0b8e8;line-height:1.8;margin:10px 0;font-family:var(--mono)}

  /* DIVIDER */
  .divider{height:1px;background:var(--border);margin:22px 0}

  /* CHECKLIST */
  .checklist{list-style:none}
  .checklist li{display:flex;gap:10px;align-items:flex-start;padding:7px 0;font-size:13px;border-bottom:1px solid rgba(255,255,255,0.04);line-height:1.5;color:#b0b0c0}
  .checklist li:last-child{border-bottom:none}
  .check-icon{color:var(--green);flex-shrink:0;margin-top:1px}

  /* TAG GROUP */
  .tag-group{display:flex;flex-wrap:wrap;gap:6px;margin:8px 0}
  .tag{background:var(--surface2);border:1px solid var(--border);border-radius:6px;padding:4px 10px;font-size:11px;color:#9090a8}
  .tag.active{border-color:rgba(124,58,237,0.5);color:#c4b5fd;background:rgba(124,58,237,0.1)}

  ::-webkit-scrollbar{width:4px;height:4px}
  ::-webkit-scrollbar-track{background:transparent}
  ::-webkit-scrollbar-thumb{background:var(--border);border-radius:2px}
</style>
</head>
<body>

<div class="hero">
  <div class="v2-badge">✨ VERSION 2.0 — AUTO SOURCE ENGINE</div>
  <div class="hero-tag">⚡ Full Autopilot · Smartphone Only · $0</div>
  <h1>Cari Sendiri.<br><span>Clip Sendiri. Viral Sendiri.</span></h1>
  <p>n8n otomatis memantau YouTube, memilih video terbaik, memotong klip viral, dan mengantre upload — semua tanpa kamu sentuh.</p>
  <div class="hero-stats">
    <div class="stat-pill"><div class="num">100+</div><div class="lbl">Clips/Hari</div></div>
    <div class="stat-pill"><div class="num">50+</div><div class="lbl">Channel Dipantau</div></div>
    <div class="stat-pill"><div class="num">0</div><div class="lbl">Input Manual</div></div>
    <div class="stat-pill"><div class="num">$0</div><div class="lbl">Biaya/Bulan</div></div>
  </div>
</div>

<div class="tabs-wrap">
  <div class="tabs">
    <button class="tab-btn active" onclick="switchTab('pipeline')">🗺 Full Pipeline</button>
    <button class="tab-btn" onclick="switchTab('source')">📡 Auto Source</button>
    <button class="tab-btn" onclick="switchTab('setup')">⚙️ Setup</button>
    <button class="tab-btn" onclick="switchTab('workflow')">🤖 n8n JSON</button>
    <button class="tab-btn" onclick="switchTab('channels')">📋 Channel List</button>
    <button class="tab-btn" onclick="switchTab('viral')">🔥 Viral Formula</button>
  </div>
</div>

<div class="content">

<!-- ===== FULL PIPELINE ===== -->
<div class="tab-panel active" id="tab-pipeline">

  <div class="section-title">Arsitektur Lengkap v2 — Full Autopilot</div>

  <div class="pipeline">
    <div class="pipe-node source">
      <div class="pipe-emoji">📡</div>
      <div class="pipe-info">
        <div class="pipe-name">AUTO SOURCE — RSS Feed Monitor</div>
        <div class="pipe-detail">n8n polling 50+ channel YouTube setiap 30 menit via RSS resmi YouTube. Tanpa API key.</div>
      </div>
      <span class="pipe-badge pb-new">NEW v2</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node source">
      <div class="pipe-emoji">🔍</div>
      <div class="pipe-info">
        <div class="pipe-name">AUTO SOURCE — yt-dlp Keyword Search</div>
        <div class="pipe-detail">Cari video berdasarkan keyword trending tiap 2 jam. Filter: min 100K views, durasi 5-40 menit.</div>
      </div>
      <span class="pipe-badge pb-new">NEW v2</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node process">
      <div class="pipe-emoji">🧹</div>
      <div class="pipe-info">
        <div class="pipe-name">DEDUP FILTER — Google Sheets Check</div>
        <div class="pipe-detail">Cek apakah URL sudah pernah diproses. Skip jika duplikat. Log ke Sheet.</div>
      </div>
      <span class="pipe-badge pb-free">FREE</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node process">
      <div class="pipe-emoji">⬇️</div>
      <div class="pipe-info">
        <div class="pipe-name">yt-dlp — Download Video 720p</div>
        <div class="pipe-detail">Download otomatis ke ~/clipper/downloads/ di Termux</div>
      </div>
      <span class="pipe-badge pb-free">FREE</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node process">
      <div class="pipe-emoji">🎙</div>
      <div class="pipe-info">
        <div class="pipe-name">Groq Whisper — Transkripsi + Timestamps</div>
        <div class="pipe-detail">Speech-to-text dengan timestamps per segmen. Gratis 14.400 menit/hari.</div>
      </div>
      <span class="pipe-badge pb-api">API FREE</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node process">
      <div class="pipe-emoji">🧠</div>
      <div class="pipe-info">
        <div class="pipe-name">Gemini Flash — AI Viral Clip Picker</div>
        <div class="pipe-detail">Analisis transkrip → pilih 5-8 momen viral terbaik + hook + caption. 1.500 req/hari gratis.</div>
      </div>
      <span class="pipe-badge pb-api">API FREE</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node process">
      <div class="pipe-emoji">✂️</div>
      <div class="pipe-info">
        <div class="pipe-name">FFmpeg — Cut + Crop 9:16 + Burn Caption</div>
        <div class="pipe-detail">Potong video, crop portrait, burn subtitle hook, encode MP4 siap upload.</div>
      </div>
      <span class="pipe-badge pb-free">FREE</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node output">
      <div class="pipe-emoji">📬</div>
      <div class="pipe-info">
        <div class="pipe-name">Telegram Bot — Kirim Hasil + Caption</div>
        <div class="pipe-detail">Clip + caption TikTok/Reels/Shorts dikirim ke HP kamu. Approve/skip via bot.</div>
      </div>
      <span class="pipe-badge pb-free">FREE</span>
    </div>
    <div class="pipe-arrow">▼</div>
    <div class="pipe-node output">
      <div class="pipe-emoji">📊</div>
      <div class="pipe-info">
        <div class="pipe-name">Google Sheets — Log & Stats</div>
        <div class="pipe-detail">Catat semua video sumber, jumlah clip, status upload, viral score.</div>
      </div>
      <span class="pipe-badge pb-free">FREE</span>
    </div>
  </div>

  <div class="alert alert-success">
    <span class="alert-icon">✅</span>
    <span><strong>Zero input manual.</strong> Kamu tinggal buka Telegram, terima clip, copy caption, upload ke 3 platform. Semua proses sebelumnya berjalan otomatis 24 jam.</span>
  </div>

  <div class="divider"></div>
  <div class="section-title">Kapasitas Harian (Realistis)</div>
  <div class="card">
    <div class="spec-row"><span class="spec-label">RSS channels dipantau</span><span class="spec-val">50 channels × update 2x/hari = ~10 video baru</span></div>
    <div class="spec-row"><span class="spec-label">yt-dlp keyword search</span><span class="spec-val">10 keyword × 5 video = ~50 video kandidat</span></div>
    <div class="spec-row"><span class="spec-label">Setelah dedup filter</span><span class="spec-val">~15-20 video unik lolos per hari</span></div>
    <div class="spec-row"><span class="spec-label">Clips per video</span><span class="spec-val">5-8 clips (pilihan Gemini AI)</span></div>
    <div class="spec-row"><span class="spec-label">Total clips/hari</span><span class="spec-val" style="color:var(--green);font-weight:700">75–160 clips siap upload</span></div>
    <div class="spec-row"><span class="spec-label">Groq limit check</span><span class="spec-val">15 video × 20 menit = 300 menit (dari 14.400 limit) ✅</span></div>
    <div class="spec-row"><span class="spec-label">Gemini limit check</span><span class="spec-val">15 request (dari 1.500 limit) ✅</span></div>
  </div>

</div>

<!-- ===== AUTO SOURCE ===== -->
<div class="tab-panel" id="tab-source">

  <div class="section-title">Metode 1 — YouTube RSS Feed (Utama)</div>

  <div class="alert alert-info">
    <span class="alert-icon">📡</span>
    <span>YouTube punya RSS endpoint resmi untuk setiap channel. <strong>Tidak butuh API key.</strong> n8n RSS node polling setiap 30 menit — kalau ada video baru, langsung trigger pipeline.</span>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">1</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Dapatkan Channel ID</div>
      <div class="flow-desc">Buka channel YouTube target → lihat URL. Format: <code style="background:#1a1a24;padding:1px 6px;border-radius:3px;font-size:11px">youtube.com/channel/UC...</code><br>Atau pakai cara cepat:</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button><span class="comment"># Cara 1: dari URL channel langsung</span>
<span class="comment"># https://youtube.com/channel/UCxxxxxx → ambil UCxxxxxx</span>

<span class="comment"># Cara 2: dari @handle pakai yt-dlp</span>
yt-dlp --print channel_id <span class="string">"https://youtube.com/@NamaChannel"</span></div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">2</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Format RSS URL YouTube</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button><span class="comment"># Template RSS URL resmi YouTube</span>
https://www.youtube.com/feeds/videos.xml?channel_id=<span class="string">CHANNEL_ID_DISINI</span>

<span class="comment"># Contoh channel besar:</span>
https://www.youtube.com/feeds/videos.xml?channel_id=<span class="hi">UCVHFbw7woebKtfvug_tExUA</span>
<span class="comment"># (Deddy Corbuzier)</span>

<span class="comment"># Test langsung di browser HP kamu — harus muncul XML</span></div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">3</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Setup di n8n — RSS Feed Read Node</div>
      <div class="flow-desc">Di n8n: Add Node → <strong>RSS Feed Read</strong> → masukkan URL RSS → set interval polling 30 menit. Bisa tambah multiple RSS URL untuk monitor banyak channel sekaligus.</div>
      <div class="alert alert-tip" style="margin-top:10px">
        <span class="alert-icon">💡</span>
        <span>n8n RSS node bisa handle <strong>multiple URL</strong> sekaligus. Masukkan 50 channel RSS URL dalam satu node, semua dipantau paralel.</span>
      </div>
    </div>
  </div>

  <div class="divider"></div>
  <div class="section-title">Metode 2 — yt-dlp Keyword Search (Pendukung)</div>

  <div class="alert alert-info">
    <span class="alert-icon">🔍</span>
    <span>yt-dlp bisa search YouTube langsung tanpa API key. Hasil difilter otomatis berdasarkan views, durasi, dan tanggal upload.</span>
  </div>

  <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button><span class="comment"># Search YouTube via yt-dlp, ambil 5 video terbaik</span>
yt-dlp \
  --default-search "ytsearch5:" \
  --get-id \
  --get-title \
  --get-duration \
  --match-filter "duration > 300 & duration < 2400 & view_count > 100000" \
  --no-download \
  <span class="string">"ytsearch5:motivasi sukses viral 2025"</span>

<span class="comment"># Hasilnya: daftar video ID + judul + durasi</span>
<span class="comment"># Kirim ke Google Sheet antrian → proses otomatis</span></div>

  <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button><span class="comment"># Script lengkap: search multi-keyword → simpan ke file</span>
<span class="keyword">KEYWORDS</span>=(<span class="string">"motivasi viral"</span> <span class="string">"tips sukses"</span> <span class="string">"fakta unik"</span> <span class="string">"tutorial singkat"</span>)

<span class="keyword">for</span> kw <span class="keyword">in</span> "${KEYWORDS[@]}"; <span class="keyword">do</span>
  echo "Searching: $kw"
  yt-dlp \
    --default-search "ytsearch5:" \
    --get-url \
    --match-filter "duration > 300 & view_count > 50000" \
    --no-download \
    <span class="string">"ytsearch5:$kw"</span> >> ~/clipper/queue/urls.txt
<span class="keyword">done</span>

sort -u ~/clipper/queue/urls.txt -o ~/clipper/queue/urls.txt
echo "Total URL unik: $(wc -l < ~/clipper/queue/urls.txt)"</div>

  <div class="divider"></div>
  <div class="section-title">Filter Cerdas — Video yang Layak Diclip</div>
  <div class="card">
    <div class="spec-row"><span class="spec-label">Durasi video</span><span class="spec-val">5–40 menit (cukup panjang untuk banyak clip)</span></div>
    <div class="spec-row"><span class="spec-label">Minimum views</span><span class="spec-val">> 50.000 (sudah terbukti menarik)</span></div>
    <div class="spec-row"><span class="spec-label">Umur video</span><span class="spec-val">Upload < 30 hari (konten fresh)</span></div>
    <div class="spec-row"><span class="spec-label">Bahasa</span><span class="spec-val">Indonesia / English (Groq support keduanya)</span></div>
    <div class="spec-row"><span class="spec-label">Sudah diproses?</span><span class="spec-val">Cek Google Sheet — skip jika duplikat</span></div>
    <div class="spec-row"><span class="spec-label">Ada subtitle asli?</span><span class="spec-val">Bonus — bisa skip Groq, hemat quota</span></div>
  </div>

</div>

<!-- ===== SETUP ===== -->
<div class="tab-panel" id="tab-setup">

  <div class="section-title">Setup Lengkap — Urutan Wajib</div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">1</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Install Termux + Semua Dependensi</div>
      <div class="flow-desc">Download Termux dari F-Droid (bukan Play Store). Jalankan satu per satu:</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button><span class="comment"># Step 1: Update</span>
pkg update && pkg upgrade -y

<span class="comment"># Step 2: Install semua yang dibutuhkan</span>
pkg install python nodejs ffmpeg git curl wget nano -y

<span class="comment"># Step 3: Install yt-dlp</span>
pip install yt-dlp

<span class="comment"># Step 4: Install n8n</span>
npm install -g n8n

<span class="comment"># Step 5: Verifikasi semua OK</span>
yt-dlp --version
ffmpeg -version | head -1
node --version
n8n --version</div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">2</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Buat Folder Struktur Kerja</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button">mkdir -p ~/clipper/{downloads,clips,queue,done,logs}
echo "Folder siap!" && ls ~/clipper</div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">3</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Daftar API Keys (Semua Gratis)</div>
      <div class="card" style="margin-top:8px">
        <div class="spec-row"><span class="spec-label">🔑 Groq API</span><span class="spec-val">console.groq.com → API Keys → Create</span></div>
        <div class="spec-row"><span class="spec-label">🔑 Gemini API</span><span class="spec-val">aistudio.google.com → Get API Key</span></div>
        <div class="spec-row"><span class="spec-label">🤖 Telegram Bot</span><span class="spec-val">Chat @BotFather → /newbot → simpan token</span></div>
        <div class="spec-row"><span class="spec-label">📊 Google Sheets</span><span class="spec-val">Buat sheet baru → share ke anyone with link</span></div>
        <div class="spec-row"><span class="spec-label">🔐 Google Service Account</span><span class="spec-val">console.cloud.google.com → IAM → Service Accounts</span></div>
      </div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">4</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Buat File Keyword List</div>
      <div class="flow-desc">File ini dipakai yt-dlp search otomatis. Edit sesuai niche kamu:</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button">cat > ~/clipper/keywords.txt << 'EOF'
motivasi sukses viral
tips produktivitas 2025
fakta unik mengejutkan
cara cepat belajar
rahasia sukses pengusaha
mindset kaya
tips hidup hemat
motivasi pagi
kisah inspiratif nyata
cara mengatasi overthinking
EOF
echo "Keywords siap: $(wc -l < ~/clipper/keywords.txt) keyword"</div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">5</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Jalankan n8n + Cloudflare Tunnel</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button><span class="comment"># Terminal 1: Jalankan n8n</span>
n8n start

<span class="comment"># Terminal 2 (buka tab baru Termux): Tunnel</span>
pkg install cloudflared -y
cloudflared tunnel --url http://localhost:5678

<span class="comment"># Salin URL tunnel → masuk n8n Settings → Webhook URL</span>
<span class="comment"># Buka n8n di browser: http://localhost:5678</span></div>
      <div class="alert alert-tip" style="margin-top:10px">
        <span class="alert-icon">💡</span>
        <span>Termux bisa buka banyak tab. Geser dari kiri atau tekan <strong>Vol+ Q</strong> untuk buka tab baru.</span>
      </div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">6</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Import n8n Workflow JSON</div>
      <div class="flow-desc">Buka n8n di browser HP → klik <strong>☰ → Import from File/JSON</strong> → paste JSON dari tab <strong>n8n JSON</strong> → Save → Activate.</div>
    </div>
  </div>

  <div class="flow-step">
    <div class="flow-line"><div class="step-dot">7</div><div class="connector"></div></div>
    <div class="flow-body">
      <div class="flow-title">Setup Google Sheets Queue</div>
      <div class="flow-desc">Buat spreadsheet baru dengan nama <strong>"AI Clipper Queue"</strong>. Sheet pertama nama <strong>"Queue"</strong>, buat header di baris 1:</div>
      <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button">A1: youtube_url
B1: status
C1: clips_count
D1: processed_at
E1: source
F1: video_title
G1: error_log</div>
    </div>
  </div>

</div>

<!-- ===== N8N WORKFLOW JSON ===== -->
<div class="tab-panel" id="tab-workflow">

  <div class="section-title">n8n Workflow JSON — Full v2 (Copy & Import)</div>

  <div class="alert alert-tip">
    <span class="alert-icon">📋</span>
    <span>Copy seluruh JSON di bawah → n8n → <strong>☰ Import → Paste JSON → Save</strong>. Ganti semua <code style="background:#1a1a24;padding:1px 5px;border-radius:3px">YOUR_*</code> dengan nilai kamu.</span>
  </div>

  <div class="code-block" style="max-height:340px;overflow-y:auto;font-size:11px"><button class="copy-btn" onclick="copyCode(this)">copy</button>{
  "name": "AI Clipper v2 — Auto Source + Full Pipeline",
  "nodes": [

    {
      "name": "⏰ Schedule — Cek RSS tiap 30 menit",
      "type": "n8n-nodes-base.scheduleTrigger",
      "parameters": {
        "rule": { "interval": [{ "field": "minutes", "minutesInterval": 30 }] }
      },
      "position": [100, 200]
    },

    {
      "name": "📡 RSS — Monitor 50 Channel",
      "type": "n8n-nodes-base.rssFeedRead",
      "parameters": {
        "url": "https://www.youtube.com/feeds/videos.xml?channel_id=UCVHFbw7woebKtfvug_tExUA"
      },
      "position": [300, 200],
      "notes": "Tambah lebih banyak channel di sini — lihat tab Channel List"
    },

    {
      "name": "⏰ Schedule — Search Keyword tiap 2 jam",
      "type": "n8n-nodes-base.scheduleTrigger",
      "parameters": {
        "rule": { "interval": [{ "field": "hours", "hoursInterval": 2 }] }
      },
      "position": [100, 400]
    },

    {
      "name": "🔍 yt-dlp Keyword Search",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "cat ~/clipper/keywords.txt | while read kw; do yt-dlp --default-search 'ytsearch3:' --get-url --match-filter 'duration > 300 & duration < 2400 & view_count > 50000' --no-download \"ytsearch3:$kw\" 2>/dev/null; done | sort -u"
      },
      "position": [300, 400]
    },

    {
      "name": "🧹 Format URL dari yt-dlp",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "const raw = $input.first().json.stdout || '';\nconst urls = raw.split('\\n').filter(u => u.includes('youtube.com/watch'));\nreturn urls.map(url => ({ json: { youtube_url: url.trim(), source: 'keyword_search' } }));"
      },
      "position": [500, 400]
    },

    {
      "name": "🔀 Merge RSS + Keyword Results",
      "type": "n8n-nodes-base.merge",
      "parameters": { "mode": "append" },
      "position": [700, 300]
    },

    {
      "name": "📊 Cek Duplikat di Google Sheets",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "read",
        "sheetId": "YOUR_GOOGLE_SHEET_ID",
        "range": "Queue!A:A"
      },
      "credentials": { "googleSheetsOAuth2Api": { "id": "1", "name": "Google Sheets" } },
      "position": [900, 300]
    },

    {
      "name": "🚫 Filter Duplikat",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "const existing = $('📊 Cek Duplikat di Google Sheets').all().map(i => i.json.A).filter(Boolean);\nconst incoming = $input.all();\nreturn incoming.filter(item => {\n  const url = item.json.youtube_url || item.json.link || '';\n  return !existing.includes(url);\n}).map(item => ({\n  json: { youtube_url: item.json.youtube_url || item.json.link, source: item.json.source || 'rss' }\n}));"
      },
      "position": [1100, 300]
    },

    {
      "name": "📝 Log ke Sheet — Status: queued",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "append",
        "sheetId": "YOUR_GOOGLE_SHEET_ID",
        "range": "Queue!A:G",
        "dataMode": "autoMapInputData"
      },
      "position": [1300, 300]
    },

    {
      "name": "⬇️ Download Video yt-dlp",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "URL='{{$json.youtube_url}}'; ID=$(yt-dlp --get-id \"$URL\" 2>/dev/null); yt-dlp -f 'bestvideo[height<=720]+bestaudio/best[height<=720]' --merge-output-format mp4 -o \"/root/clipper/downloads/%(id)s.%(ext)s\" \"$URL\" 2>&1 | tail -3; echo \"FILE:/root/clipper/downloads/${ID}.mp4\""
      },
      "position": [1500, 300]
    },

    {
      "name": "🎙 Ekstrak Audio untuk Groq",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "VIDFILE=$(echo '{{$json.stdout}}' | grep 'FILE:' | sed 's/FILE://'); ffmpeg -i \"$VIDFILE\" -vn -ar 16000 -ac 1 -b:a 64k /root/clipper/downloads/audio_temp.mp3 -y 2>/dev/null && echo \"AUDIO_OK:$VIDFILE\""
      },
      "position": [1700, 300]
    },

    {
      "name": "🎙 Groq Whisper — Transkripsi",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "method": "POST",
        "url": "https://api.groq.com/openai/v1/audio/transcriptions",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [{ "name": "Authorization", "value": "Bearer YOUR_GROQ_API_KEY" }]
        },
        "sendBody": true,
        "contentType": "multipart-form-data",
        "bodyParameters": {
          "parameters": [
            { "name": "file", "value": "={{ $binary.data }}", "parameterType": "formBinaryData" },
            { "name": "model", "value": "whisper-large-v3" },
            { "name": "response_format", "value": "verbose_json" },
            { "name": "timestamp_granularities[]", "value": "segment" },
            { "name": "language", "value": "id" }
          ]
        }
      },
      "position": [1900, 300]
    },

    {
      "name": "🧠 Gemini Flash — Pilih Momen Viral",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "method": "POST",
        "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=YOUR_GEMINI_API_KEY",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [{
            "name": "contents",
            "value": "=[{\"parts\":[{\"text\":\"Kamu adalah ahli viral short-form content. Analisis transkrip berikut dan pilih 5-8 momen terbaik untuk video pendek 30-90 detik. Kriteria: fakta mengejutkan, konflik emosional, solusi praktis, quote powerful, momen AHA. Untuk setiap clip berikan JSON: {start_time, end_time, hook, why_viral, viral_score, caption_tiktok, caption_reels, caption_shorts}. Transkrip: \" + $json.text + \". BALAS HANYA JSON ARRAY, tanpa teks lain.\"}]}]"
          }]
        }
      },
      "position": [2100, 300]
    },

    {
      "name": "🔧 Parse Clips dari Gemini",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "const raw = $input.first().json.candidates[0].content.parts[0].text;\nconst clean = raw.replace(/```json|```/g,'').trim();\nlet clips;\ntry { clips = JSON.parse(clean); } catch(e) { clips = []; }\nconst videoFile = $('🎙 Ekstrak Audio untuk Groq').first().json.stdout.replace('AUDIO_OK:','').trim();\nreturn clips.map((c,i) => ({ json: { ...c, index: i, videoFile, processed: new Date().toISOString() } }));"
      },
      "position": [2300, 300]
    },

    {
      "name": "✂️ FFmpeg — Cut + Crop 9:16 + Caption",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "START={{$json.start_time}}; END={{$json.end_time}}; VID='{{$json.videoFile}}'; HOOK='{{$json.hook}}'; OUT='/root/clipper/clips/clip_{{$json.index}}_$(date +%s).mp4'; DUR=$((END-START)); ffmpeg -ss $START -t $DUR -i \"$VID\" -vf \"crop=ih*9/16:ih,scale=1080:1920,drawtext=text='$HOOK':fontsize=50:fontcolor=white:x=(w-text_w)/2:y=h*0.12:box=1:boxcolor=black@0.55:boxborderw=16,fade=in:0:15\" -c:v libx264 -preset fast -crf 23 -c:a aac -b:a 128k -movflags +faststart \"$OUT\" -y 2>/dev/null && echo \"CLIP:$OUT\""
      },
      "position": [2500, 300]
    },

    {
      "name": "📱 Kirim ke Telegram",
      "type": "n8n-nodes-base.telegram",
      "parameters": {
        "operation": "sendDocument",
        "chatId": "YOUR_TELEGRAM_CHAT_ID",
        "binaryData": true,
        "binaryProperty": "data",
        "additionalFields": {
          "caption": "🎬 *Clip {{$json.index+1}}*\n\n⚡ *Hook:* {{$json.hook}}\n📊 *Viral Score:* {{$json.viral_score}}/10\n💡 *Kenapa viral:* {{$json.why_viral}}\n\n📱 *TikTok:*\n{{$json.caption_tiktok}}\n\n📸 *Reels:*\n{{$json.caption_reels}}\n\n▶️ *Shorts:*\n{{$json.caption_shorts}}"
        }
      },
      "position": [2700, 300]
    },

    {
      "name": "✅ Update Sheet — Status: done",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "update",
        "sheetId": "YOUR_GOOGLE_SHEET_ID",
        "range": "Queue!B:D",
        "dataMode": "autoMapInputData"
      },
      "position": [2900, 300]
    }

  ],
  "connections": {
    "⏰ Schedule — Cek RSS tiap 30 menit": { "main": [[{ "node": "📡 RSS — Monitor 50 Channel", "type": "main", "index": 0 }]] },
    "📡 RSS — Monitor 50 Channel": { "main": [[{ "node": "🔀 Merge RSS + Keyword Results", "type": "main", "index": 0 }]] },
    "⏰ Schedule — Search Keyword tiap 2 jam": { "main": [[{ "node": "🔍 yt-dlp Keyword Search", "type": "main", "index": 0 }]] },
    "🔍 yt-dlp Keyword Search": { "main": [[{ "node": "🧹 Format URL dari yt-dlp", "type": "main", "index": 0 }]] },
    "🧹 Format URL dari yt-dlp": { "main": [[{ "node": "🔀 Merge RSS + Keyword Results", "type": "main", "index": 1 }]] },
    "🔀 Merge RSS + Keyword Results": { "main": [[{ "node": "📊 Cek Duplikat di Google Sheets", "type": "main", "index": 0 }]] },
    "📊 Cek Duplikat di Google Sheets": { "main": [[{ "node": "🚫 Filter Duplikat", "type": "main", "index": 0 }]] },
    "🚫 Filter Duplikat": { "main": [[{ "node": "📝 Log ke Sheet — Status: queued", "type": "main", "index": 0 }]] },
    "📝 Log ke Sheet — Status: queued": { "main": [[{ "node": "⬇️ Download Video yt-dlp", "type": "main", "index": 0 }]] },
    "⬇️ Download Video yt-dlp": { "main": [[{ "node": "🎙 Ekstrak Audio untuk Groq", "type": "main", "index": 0 }]] },
    "🎙 Ekstrak Audio untuk Groq": { "main": [[{ "node": "🎙 Groq Whisper — Transkripsi", "type": "main", "index": 0 }]] },
    "🎙 Groq Whisper — Transkripsi": { "main": [[{ "node": "🧠 Gemini Flash — Pilih Momen Viral", "type": "main", "index": 0 }]] },
    "🧠 Gemini Flash — Pilih Momen Viral": { "main": [[{ "node": "🔧 Parse Clips dari Gemini", "type": "main", "index": 0 }]] },
    "🔧 Parse Clips dari Gemini": { "main": [[{ "node": "✂️ FFmpeg — Cut + Crop 9:16 + Caption", "type": "main", "index": 0 }]] },
    "✂️ FFmpeg — Cut + Crop 9:16 + Caption": { "main": [[{ "node": "📱 Kirim ke Telegram", "type": "main", "index": 0 }]] },
    "📱 Kirim ke Telegram": { "main": [[{ "node": "✅ Update Sheet — Status: done", "type": "main", "index": 0 }]] }
  }
}</div>

  <div class="section-title">Hal yang Wajib Diganti</div>
  <div class="card">
    <div class="spec-row"><span class="spec-label">YOUR_GROQ_API_KEY</span><span class="spec-val">Dari console.groq.com</span></div>
    <div class="spec-row"><span class="spec-label">YOUR_GEMINI_API_KEY</span><span class="spec-val">Dari aistudio.google.com</span></div>
    <div class="spec-row"><span class="spec-label">YOUR_GOOGLE_SHEET_ID</span><span class="spec-val">Dari URL spreadsheet kamu</span></div>
    <div class="spec-row"><span class="spec-label">YOUR_TELEGRAM_CHAT_ID</span><span class="spec-val">Kirim /start ke bot kamu → cek chat ID via @userinfobot</span></div>
    <div class="spec-row"><span class="spec-label">Channel RSS URL</span><span class="spec-val">Ganti channel_id dengan channel target kamu (lihat tab Channel List)</span></div>
  </div>

</div>

<!-- ===== CHANNEL LIST ===== -->
<div class="tab-panel" id="tab-channels">

  <div class="section-title">50 Channel Indonesia Terpopuler — Siap Pakai</div>
  <div class="alert alert-info">
    <span class="alert-icon">📋</span>
    <span>Copy channel ID ini langsung ke RSS URL: <code style="background:#1a1a24;padding:1px 5px;border-radius:3px;font-size:10px">youtube.com/feeds/videos.xml?channel_id=ID_DI_SINI</code></span>
  </div>

  <div class="section-title">🎤 Podcast / Motivasi / Bisnis</div>
  <div class="channel-grid">
    <div class="channel-item"><div class="ch-name">Deddy Corbuzier</div><div class="ch-id">UCVHFbw7woebKtfvug_tExUA</div></div>
    <div class="channel-item"><div class="ch-name">Ria Ricis</div><div class="ch-id">UCOOEKOJSQmkYeFEztwlAoIw</div></div>
    <div class="channel-item"><div class="ch-name">Raditya Dika</div><div class="ch-id">UCVPo3NzDXIHQnFHC6JTXGiQ</div></div>
    <div class="channel-item"><div class="ch-name">Young On Top</div><div class="ch-id">UCmGZ9fS5pIXrAGmx0b_YVWQ</div></div>
    <div class="channel-item"><div class="ch-name">Gita Wirjawan</div><div class="ch-id">UCyXnGwH0BLblbsRKEtcOdxQ</div></div>
    <div class="channel-item"><div class="ch-name">Helmy Yahya Bicara</div><div class="ch-id">UCWRqIDAfHAWV3KjrZn5Gq5g</div></div>
    <div class="channel-item"><div class="ch-name">Noice ID</div><div class="ch-id">UCgXWVMKx32Ev5Kl7spG7kMQ</div></div>
    <div class="channel-item"><div class="ch-name">Merry Riana</div><div class="ch-id">UCSlbwJcEITcBq0QOklbQFdA</div></div>
  </div>

  <div class="section-title">📚 Edukasi / Tips / How-To</div>
  <div class="channel-grid">
    <div class="channel-item"><div class="ch-name">Aditya Tyo</div><div class="ch-id">UCiRRKPq6K3mhqRNqMkb2Y0Q</div></div>
    <div class="channel-item"><div class="ch-name">Dea Sihotang</div><div class="ch-id">UC7iFbWhFSfqfgNefmqNhMjw</div></div>
    <div class="channel-item"><div class="ch-name">James & Joanna</div><div class="ch-id">UCfPOsqilKDPkQ1nfXaXE9pg</div></div>
    <div class="channel-item"><div class="ch-name">Jeremy Noven</div><div class="ch-id">UC5J1pGvlz4KW5DP-5R7WBOA</div></div>
    <div class="channel-item"><div class="ch-name">Luna Maya</div><div class="ch-id">UCzN6NpAHAC_8jCLSpGU4GzQ</div></div>
    <div class="channel-item"><div class="ch-name">Awkarin</div><div class="ch-id">UCBJk5sRAGaXB8wPdqqBv7gQ</div></div>
    <div class="channel-item"><div class="ch-name">Andovi da Lopez</div><div class="ch-id">UCz3xjdvpSCpiaCExixbAFNw</div></div>
    <div class="channel-item"><div class="ch-name">GadgetIn</div><div class="ch-id">UCkejXXBqaSFKt6MXHLA9VuA</div></div>
  </div>

  <div class="section-title">🌍 Channel Internasional Viral (English)</div>
  <div class="channel-grid">
    <div class="channel-item"><div class="ch-name">Andrew Huberman</div><div class="ch-id">UC2D2CMWXMOVWx7giW1n3LIg</div></div>
    <div class="channel-item"><div class="ch-name">Lex Fridman</div><div class="ch-id">UCSHZKyawb77ixDdsGog4iWA</div></div>
    <div class="channel-item"><div class="ch-name">Ali Abdaal</div><div class="ch-id">UCoOae5nYA7VqaXzerajD0lg</div></div>
    <div class="channel-item"><div class="ch-name">Jordan Peterson</div><div class="ch-id">UCL_f53ZEJxp8TtlOkHwMV9Q</div></div>
    <div class="channel-item"><div class="ch-name">Mark Manson</div><div class="ch-id">UCkyNQJNV4sCPcnXg2HzmWqQ</div></div>
    <div class="channel-item"><div class="ch-name">Simon Sinek</div><div class="ch-id">UCiHZIhEQmh2BkJCQjEiNKxA</div></div>
    <div class="channel-item"><div class="ch-name">Gary Vaynerchuk</div><div class="ch-id">UCctXZhXmG-kf3tlIXgVZUlw</div></div>
    <div class="channel-item"><div class="ch-name">Y Combinator</div><div class="ch-id">UCcefcZRL2oaA_uBNeo5UNqg</div></div>
  </div>

  <div class="section-title">Cara Generate RSS URL Massal</div>
  <div class="code-block"><button class="copy-btn" onclick="copyCode(this)">copy</button"><span class="comment"># Script: generate semua RSS URL sekaligus</span>
<span class="keyword">CHANNELS</span>=(
  <span class="string">"UCVHFbw7woebKtfvug_tExUA"</span>  <span class="comment"># Deddy Corbuzier</span>
  <span class="string">"UC2D2CMWXMOVWx7giW1n3LIg"</span>  <span class="comment"># Andrew Huberman</span>
  <span class="string">"UCSHZKyawb77ixDdsGog4iWA"</span>  <span class="comment"># Lex Fridman</span>
  <span class="string">"UCoOae5nYA7VqaXzerajD0lg"</span>  <span class="comment"># Ali Abdaal</span>
  <span class="comment"># ... tambah lebih banyak</span>
)

<span class="keyword">for</span> id <span class="keyword">in</span> "${CHANNELS[@]}"; <span class="keyword">do</span>
  echo "https://www.youtube.com/feeds/videos.xml?channel_id=$id"
<span class="keyword">done</span></div>

  <div class="alert alert-tip">
    <span class="alert-icon">💡</span>
    <span>Di n8n, node <strong>RSS Feed Read</strong> bisa diisi <strong>satu URL per node</strong>. Buat beberapa RSS node dan sambungkan semua ke satu Merge node. Atau gunakan Code node untuk loop semua URL sekaligus.</span>
  </div>

</div>

<!-- ===== VIRAL FORMULA ===== -->
<div class="tab-panel" id="tab-viral">

  <div class="section-title">Formula Hook yang Memaksa Orang Berhenti</div>

  <div class="card">
    <div class="card-header">
      <div class="card-icon" style="background:rgba(239,68,68,0.15)">🎯</div>
      <div><div class="card-title">5 Template Hook Terbukti Viral</div><div class="card-sub">Gunakan sebagai teks overlay di 3 detik pertama</div></div>
    </div>
    <div class="spec-row"><span class="spec-label">Kontra-Intuisi</span><span class="spec-val" style="font-size:10px">"Ternyata kamu SALAH soal [topik]..."</span></div>
    <div class="spec-row"><span class="spec-label">Angka Spesifik</span><span class="spec-val" style="font-size:10px">"7 detik ini ubah hidupmu selamanya"</span></div>
    <div class="spec-row"><span class="spec-label">Pertanyaan Tajam</span><span class="spec-val" style="font-size:10px">"Kenapa orang kaya tidak punya goals?"</span></div>
    <div class="spec-row"><span class="spec-label">Konflik / Drama</span><span class="spec-val" style="font-size:10px">"Dia kehilangan Rp1M karena hal ini..."</span></div>
    <div class="spec-row"><span class="spec-label">Secret / Exclusive</span><span class="spec-val" style="font-size:10px">"Yang orang sukses tidak pernah bilang"</span></div>
  </div>

  <div class="section-title">Prompt Gemini — Optimasi Viral Score</div>
  <div class="prompt-box">Kamu adalah viral content expert dengan track record 50+ video 10 juta views.

Tugas: Analisis transkrip video berikut. Temukan 5-8 momen dengan POTENSI VIRAL TERTINGGI untuk short video 30-90 detik.

KRITERIA WAJIB setiap momen:
1. Ada "pattern interrupt" — sesuatu yang mengejutkan atau kontra-intuisi
2. Bisa dipahami tanpa konteks video sebelumnya (standalone)
3. Mengandung emosi: kagum / takut / termotivasi / marah / tertawa
4. Ada satu kalimat yang bisa jadi quote / caption sendiri

OUTPUT FORMAT (JSON array only, no text):
[{
  "start_time": 45,
  "end_time": 112,
  "hook": "Kata pertama yang bikin scroll berhenti (max 8 kata)",
  "viral_score": 8,
  "emotion": "kagum",
  "standalone": true,
  "caption_tiktok": "Caption + 5 hashtag #fyp #viral",
  "caption_reels": "Caption berbeda untuk Instagram",
  "caption_shorts": "Caption untuk YouTube Shorts"
}]

TRANSKRIP:
[TRANSKRIP_DI_SINI]</div>

  <div class="section-title">Keyword Trending — Update Mingguan</div>
  <div class="tag-group">
    <span class="tag active">motivasi sukses</span>
    <span class="tag active">tips produktivitas</span>
    <span class="tag active">fakta unik</span>
    <span class="tag active">mindset kaya</span>
    <span class="tag active">cara belajar cepat</span>
    <span class="tag">psikologi manusia</span>
    <span class="tag">rahasia pengusaha</span>
    <span class="tag">tips karir</span>
    <span class="tag">self improvement</span>
    <span class="tag">hidup sehat</span>
    <span class="tag">investasi pemula</span>
    <span class="tag">passive income</span>
    <span class="tag">cara hemat uang</span>
    <span class="tag">digital marketing</span>
    <span class="tag">AI untuk kerja</span>
  </div>
  <div class="flow-desc" style="font-size:12px;margin-top:6px">Tambahkan keyword baru ke <code style="background:#1a1a24;padding:1px 5px;border-radius:3px">~/clipper/keywords.txt</code> — yt-dlp akan otomatis pakai di run berikutnya.</div>

  <div class="divider"></div>

  <div class="section-title">Checklist Clip Siap Upload</div>
  <ul class="checklist">
    <li><span class="check-icon">✓</span>Resolusi 1080×1920 (portrait 9:16)</li>
    <li><span class="check-icon">✓</span>Hook teks muncul di 0-3 detik pertama</li>
    <li><span class="check-icon">✓</span>Durasi 30-60 detik (sweet spot semua platform)</li>
    <li><span class="check-icon">✓</span>Audio jelas, tidak terpotong di awal/akhir</li>
    <li><span class="check-icon">✓</span>Caption sudah siap untuk 3 platform (dari Telegram)</li>
    <li><span class="check-icon">✓</span>File size di bawah 100MB untuk upload cepat</li>
    <li><span class="check-icon">✓</span>Ada CTA di 5 detik terakhir</li>
    <li><span class="check-icon">✓</span>Konten standalone — tidak butuh konteks video asli</li>
  </ul>

</div>

</div><!-- end content -->

<script>
function switchTab(id) {
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  document.querySelectorAll('.tab-panel').forEach(p => p.classList.remove('active'));
  document.getElementById('tab-' + id).classList.add('active');
  event.target.classList.add('active');
}

function copyCode(btn) {
  const block = btn.parentElement;
  const clone = block.cloneNode(true);
  clone.querySelectorAll('.copy-btn').forEach(b => b.remove());
  const text = clone.innerText || clone.textContent;
  navigator.clipboard.writeText(text.trim()).then(() => {
    btn.textContent = 'copied!';
    btn.style.color = '#10b981';
    setTimeout(() => { btn.textContent = 'copy'; btn.style.color = ''; }, 2000);
  }).catch(() => {
    btn.textContent = 'error';
    setTimeout(() => { btn.textContent = 'copy'; }, 2000);
  });
}

function toggleSection(header) {
  header.classList.toggle('open');
  header.nextElementSibling.classList.toggle('open');
}
</script>
</body>
</html>
