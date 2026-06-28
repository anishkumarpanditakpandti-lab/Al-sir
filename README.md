<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>AI Chat Pro</title>

  <!-- Google Identity Services -->
  <script src="https://accounts.google.com/gsi/client" async defer></script>

  <!-- React + Babel -->
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: 'Segoe UI', system-ui, sans-serif; }
    @keyframes pulse { 0%,80%,100%{opacity:0.3;transform:scale(0.8)} 40%{opacity:1;transform:scale(1)} }
    @keyframes spin { to { transform: rotate(360deg); } }
    @keyframes fadeIn { from{opacity:0;transform:translateY(8px)} to{opacity:1;transform:translateY(0)} }
    ::-webkit-scrollbar { width: 6px; }
    ::-webkit-scrollbar-track { background: transparent; }
    ::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.15); border-radius: 3px; }
  </style>
</head>
<body>
  <div id="root"></div>

  <script type="text/babel">
    const { useState, useRef, useEffect, useCallback } = React;

    // ─── CONFIGURATION ────────────────────────────────────────────────────────
    // 1. Go to https://console.cloud.google.com/
    // 2. Create a project → APIs & Services → Credentials → Create OAuth 2.0 Client ID
    // 3. Add your deployed domain to "Authorized JavaScript origins"
    // 4. Paste your Client ID below:
    const GOOGLE_CLIENT_ID = "YOUR_GOOGLE_CLIENT_ID_HERE.apps.googleusercontent.com";

    // 5. Add your Anthropic API key below:
    //    Get one at https://console.anthropic.com/
    //    WARNING: For production, proxy this through your own backend!
    const ANTHROPIC_API_KEY = "YOUR_ANTHROPIC_API_KEY_HERE";
    // ─────────────────────────────────────────────────────────────────────────

    const PERSONALITIES = [
      { id: "deeplearning", name: "Deep Learning Expert", emoji: "🧠", color: "#00d4ff", prompt: "You are a deep learning and AI expert. Explain concepts clearly with examples. Specialize in neural networks, ML, and AI topics." },
      { id: "tutor",       name: "Friendly Tutor",       emoji: "👨‍🏫", color: "#00e676", prompt: "You are a friendly, patient tutor. Break down complex topics into simple steps. Use analogies and encouragement." },
      { id: "researcher",  name: "AI Researcher",        emoji: "🔬", color: "#ff6d00", prompt: "You are a cutting-edge AI researcher. Discuss latest papers, trends, and technical details with precision." },
      { id: "coder",       name: "Code Assistant",       emoji: "💻", color: "#e040fb", prompt: "You are a coding assistant specialized in Python, TensorFlow, PyTorch, and ML code. Always provide working examples." },
    ];

    // ─── GOOGLE LOGO SVG ─────────────────────────────────────────────────────
    function GoogleLogo() {
      return (
        <svg width="20" height="20" viewBox="0 0 48 48">
          <path fill="#EA4335" d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"/>
          <path fill="#4285F4" d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"/>
          <path fill="#FBBC05" d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"/>
          <path fill="#34A853" d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.18 1.48-4.97 2.31-8.16 2.31-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"/>
        </svg>
      );
    }

    // ─── LOGIN PAGE ──────────────────────────────────────────────────────────
    function LoginPage({ onLogin, darkMode, setDarkMode }) {
      const [guestName, setGuestName] = useState("");
      const [error, setError] = useState("");
      const [googleReady, setGoogleReady] = useState(false);
      const googleBtnRef = useRef(null);

      const t = {
        card:    darkMode ? "rgba(255,255,255,0.05)" : "rgba(255,255,255,0.95)",
        border:  darkMode ? "rgba(255,255,255,0.1)"  : "rgba(0,0,0,0.12)",
        text:    darkMode ? "#e8f4ff" : "#0a1628",
        subtext: darkMode ? "#5a7a9a" : "#6a7a9a",
        input:   darkMode ? "rgba(255,255,255,0.07)" : "rgba(0,0,0,0.04)",
        bg:      darkMode ? "linear-gradient(135deg,#0a0a1a,#0d1b2a)" : "linear-gradient(135deg,#f0f4ff,#e8f0ff)",
      };

      // Initialize Google Sign-In
      useEffect(() => {
        const init = () => {
          if (!window.google || GOOGLE_CLIENT_ID === "YOUR_GOOGLE_CLIENT_ID_HERE.apps.googleusercontent.com") return;
          try {
            window.google.accounts.id.initialize({
              client_id: GOOGLE_CLIENT_ID,
              callback: (response) => {
                // Decode JWT
                const payload = JSON.parse(atob(response.credential.split('.')[1]));
                onLogin({
                  name: payload.name,
                  email: payload.email,
                  avatar: payload.picture || payload.name[0].toUpperCase(),
                  avatarIsUrl: !!payload.picture,
                  avatarColor: "#4285F4",
                  provider: "google",
                });
              },
              auto_select: false,
            });
            if (googleBtnRef.current) {
              window.google.accounts.id.renderButton(googleBtnRef.current, {
                theme: darkMode ? "filled_black" : "outline",
                size: "large",
                width: 320,
                text: "continue_with",
              });
            }
            setGoogleReady(true);
          } catch(e) { console.warn("Google init failed:", e); }
        };

        if (window.google) { init(); }
        else {
          const interval = setInterval(() => { if (window.google) { clearInterval(interval); init(); } }, 200);
          return () => clearInterval(interval);
        }
      }, [darkMode]);

      const handleGuest = () => {
        if (!guestName.trim()) { setError("Please enter your name."); return; }
        onLogin({ name: guestName.trim(), email: "guest", avatar: guestName[0].toUpperCase(), avatarColor: "#7b2fff", provider: "guest" });
      };

      const isConfigured = GOOGLE_CLIENT_ID !== "YOUR_GOOGLE_CLIENT_ID_HERE.apps.googleusercontent.com";

      return (
        <div style={{ position:"fixed", inset:0, background:t.bg, display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", padding:"20px" }}>
          <button onClick={() => setDarkMode(!darkMode)} style={{ position:"absolute", top:"16px", right:"16px", background:t.card, border:`1px solid ${t.border}`, borderRadius:"8px", padding:"6px 10px", cursor:"pointer", fontSize:"16px" }}>
            {darkMode ? "☀️" : "🌙"}
          </button>

          <div style={{ width:"100%", maxWidth:"380px", background:t.card, border:`1px solid ${t.border}`, borderRadius:"24px", padding:"36px 28px", backdropFilter:"blur(20px)", boxShadow: darkMode ? "0 20px 60px rgba(0,0,0,0.5)" : "0 20px 60px rgba(0,0,0,0.1)" }}>

            {/* Logo */}
            <div style={{ textAlign:"center", marginBottom:"28px" }}>
              <div style={{ width:"64px", height:"64px", margin:"0 auto 14px", background:"linear-gradient(135deg,#00d4ff,#7b2fff)", borderRadius:"18px", display:"flex", alignItems:"center", justifyContent:"center", fontSize:"32px" }}>🧠</div>
              <h1 style={{ margin:"0 0 6px", fontSize:"24px", fontWeight:"800", background:"linear-gradient(90deg,#00d4ff,#7b2fff)", WebkitBackgroundClip:"text", WebkitTextFillColor:"transparent" }}>AI Chat Pro</h1>
              <p style={{ margin:0, color:t.subtext, fontSize:"13px" }}>Your Deep Learning Assistant</p>
            </div>

            {/* Google Sign-In */}
            {isConfigured ? (
              <div ref={googleBtnRef} style={{ display:"flex", justifyContent:"center", marginBottom:"20px", minHeight:"44px" }} />
            ) : (
              <div style={{ background:"rgba(255,165,0,0.1)", border:"1px solid rgba(255,165,0,0.3)", borderRadius:"12px", padding:"12px", marginBottom:"20px", fontSize:"12px", color:"#ffaa00", lineHeight:"1.6" }}>
                ⚙️ <strong>Setup required:</strong> Open this HTML file and replace <code>YOUR_GOOGLE_CLIENT_ID_HERE</code> with your Google OAuth Client ID to enable real Google login.
                <br/><br/>
                <a href="https://console.cloud.google.com/" target="_blank" style={{ color:"#4285F4" }}>→ Get Client ID at console.cloud.google.com</a>
              </div>
            )}

            {/* Divider */}
            <div style={{ display:"flex", alignItems:"center", gap:"10px", marginBottom:"20px" }}>
              <div style={{ flex:1, height:"1px", background:t.border }} />
              <span style={{ color:t.subtext, fontSize:"12px" }}>or continue as guest</span>
              <div style={{ flex:1, height:"1px", background:t.border }} />
            </div>

            {/* Guest login */}
            <input
              placeholder="Your name"
              value={guestName}
              onChange={e => { setGuestName(e.target.value); setError(""); }}
              onKeyDown={e => e.key === "Enter" && handleGuest()}
              style={{ width:"100%", padding:"12px 14px", marginBottom:"10px", borderRadius:"10px", border:`1px solid ${t.border}`, background:t.input, color:t.text, fontSize:"14px", outline:"none", fontFamily:"inherit" }}
            />
            {error && <div style={{ color:"#ff5555", fontSize:"12px", marginBottom:"8px" }}>⚠️ {error}</div>}
            <button onClick={handleGuest} style={{ width:"100%", padding:"13px", background:"linear-gradient(135deg,#7b2fff,#00d4ff)", border:"none", borderRadius:"12px", color:"#fff", fontWeight:"700", fontSize:"14px", cursor:"pointer" }}>
              Continue as Guest →
            </button>

            <p style={{ textAlign:"center", color:t.subtext, fontSize:"11px", marginTop:"16px" }}>By continuing you agree to our Terms & Privacy Policy</p>
          </div>
        </div>
      );
    }

    // ─── MAIN APP ─────────────────────────────────────────────────────────────
    function App() {
      const [user, setUser]                   = useState(null);
      const [darkMode, setDarkMode]           = useState(true);
      const [personality, setPersonality]     = useState(PERSONALITIES[0]);
      const [sessions, setSessions]           = useState(() => { try { return JSON.parse(localStorage.getItem("chatSessions") || "{}"); } catch { return {}; } });
      const [currentSession, setCurrentSession] = useState("default");
      const [input, setInput]                 = useState("");
      const [loading, setLoading]             = useState(false);
      const [listening, setListening]         = useState(false);
      const [showPersonalities, setShowPersonalities] = useState(false);
      const [showHistory, setShowHistory]     = useState(false);
      const bottomRef  = useRef(null);
      const recognitionRef = useRef(null);

      useEffect(() => { bottomRef.current?.scrollIntoView({ behavior: "smooth" }); }, [sessions, currentSession, loading]);

      const getMessages = () => sessions[currentSession] || [
        { role: "assistant", content: `Hi ${user?.name}! I'm your ${personality.name}. Ask me anything! ${personality.emoji}` }
      ];

      const saveSession = (newSessions) => {
        setSessions(newSessions);
        try { localStorage.setItem("chatSessions", JSON.stringify(newSessions)); } catch {}
      };

      const sendMessage = async () => {
        const text = input.trim();
        if (!text || loading) return;
        const msgs = getMessages();
        const updated = [...msgs, { role: "user", content: text }];
        const newSessions = { ...sessions, [currentSession]: updated };
        saveSession(newSessions);
        setInput(""); setLoading(true);

        const apiKey = ANTHROPIC_API_KEY !== "YOUR_ANTHROPIC_API_KEY_HERE" ? ANTHROPIC_API_KEY : null;

        try {
          if (!apiKey) throw new Error("No API key configured");
          const response = await fetch("https://api.anthropic.com/v1/messages", {
            method: "POST",
            headers: { "Content-Type": "application/json", "x-api-key": apiKey, "anthropic-version": "2023-06-01", "anthropic-dangerous-direct-browser-access": "true" },
            body: JSON.stringify({ model: "claude-sonnet-4-6", max_tokens: 1000, system: personality.prompt, messages: updated.map(m => ({ role: m.role, content: m.content })) }),
          });
          const data = await response.json();
          if (data.error) throw new Error(data.error.message);
          const reply = data.content?.map(c => c.text).join("") || "No response.";
          saveSession({ ...newSessions, [currentSession]: [...updated, { role: "assistant", content: reply }] });
        } catch (err) {
          const errMsg = err.message.includes("No API key")
            ? "⚠️ Add your Anthropic API key in the HTML file to enable AI responses."
            : `⚠️ Error: ${err.message}`;
          saveSession({ ...newSessions, [currentSession]: [...updated, { role: "assistant", content: errMsg }] });
        } finally { setLoading(false); }
      };

      const startVoice = () => {
        const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
        if (!SR) { alert("Voice not supported in this browser. Try Chrome."); return; }
        const rec = new SR();
        rec.lang = "en-US"; rec.interimResults = false;
        rec.onresult = e => { setInput(e.results[0][0].transcript); setListening(false); };
        rec.onerror = () => setListening(false);
        rec.onend   = () => setListening(false);
        recognitionRef.current = rec; rec.start(); setListening(true);
      };

      const stopVoice  = () => { recognitionRef.current?.stop(); setListening(false); };
      const newChat    = () => { setCurrentSession("chat_" + Date.now()); setShowHistory(false); };
      const clearAll   = () => { saveSession({}); setCurrentSession("default"); setShowHistory(false); };
      const handleKey  = e => { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); sendMessage(); } };
      const handleLogout = () => {
        if (window.google) try { window.google.accounts.id.disableAutoSelect(); } catch {}
        setUser(null);
      };

      if (!user) return <LoginPage onLogin={setUser} darkMode={darkMode} setDarkMode={setDarkMode} />;

      const th = {
        surface:  darkMode ? "rgba(255,255,255,0.04)" : "rgba(0,0,0,0.04)",
        border:   darkMode ? "rgba(255,255,255,0.1)"  : "rgba(0,0,0,0.12)",
        text:     darkMode ? "#e8f4ff" : "#0a1628",
        subtext:  darkMode ? "#5a7a9a" : "#6a7a9a",
        inputBg:  darkMode ? "rgba(255,255,255,0.06)" : "rgba(255,255,255,0.9)",
        bubbleAI: darkMode ? "rgba(255,255,255,0.07)" : "rgba(0,0,0,0.06)",
        bg:       darkMode ? "linear-gradient(135deg,#0a0a1a,#0d1b2a)" : "linear-gradient(135deg,#f0f4ff,#e8f0ff)",
      };

      const messages    = getMessages();
      const sessionKeys = Object.keys(sessions);

      return (
        <div style={{ position:"fixed", inset:0, background:th.bg, display:"flex", flexDirection:"column", alignItems:"center", padding:"12px", overflow:"hidden" }}>

          {/* ── Header ── */}
          <div style={{ width:"100%", maxWidth:"720px", marginBottom:"10px", flexShrink:0 }}>
            <div style={{ display:"flex", alignItems:"center", justifyContent:"space-between" }}>
              {/* Brand */}
              <div style={{ display:"flex", alignItems:"center", gap:"10px" }}>
                <div style={{ width:"36px", height:"36px", background:`linear-gradient(135deg,${personality.color},#7b2fff)`, borderRadius:"10px", display:"flex", alignItems:"center", justifyContent:"center", fontSize:"18px" }}>{personality.emoji}</div>
                <div>
                  <div style={{ fontSize:"16px", fontWeight:"700", background:`linear-gradient(90deg,${personality.color},#7b2fff)`, WebkitBackgroundClip:"text", WebkitTextFillColor:"transparent" }}>AI Chat Pro</div>
                  <div style={{ fontSize:"11px", color:th.subtext }}>{personality.name}</div>
                </div>
              </div>
              {/* Controls */}
              <div style={{ display:"flex", gap:"6px", alignItems:"center" }}>
                {/* Avatar */}
                {user.avatarIsUrl
                  ? <img src={user.avatar} alt={user.name} title={`${user.name} · Click to sign out`} onClick={handleLogout} style={{ width:"32px", height:"32px", borderRadius:"50%", cursor:"pointer", border:`2px solid ${th.border}` }} />
                  : <div title={`${user.name} · Click to sign out`} onClick={handleLogout} style={{ width:"32px", height:"32px", borderRadius:"50%", background:user.avatarColor||"#7b2fff", display:"flex", alignItems:"center", justifyContent:"center", fontSize:"14px", fontWeight:"700", color:"#fff", cursor:"pointer" }}>{user.avatar}</div>
                }
                <button onClick={() => { setShowHistory(!showHistory); setShowPersonalities(false); }} style={{ background:th.surface, border:`1px solid ${th.border}`, borderRadius:"8px", padding:"6px 10px", cursor:"pointer", fontSize:"16px" }}>🗂️</button>
                <button onClick={() => { setShowPersonalities(!showPersonalities); setShowHistory(false); }} style={{ background:th.surface, border:`1px solid ${th.border}`, borderRadius:"8px", padding:"6px 10px", cursor:"pointer", fontSize:"16px" }}>🤖</button>
                <button onClick={() => setDarkMode(!darkMode)} style={{ background:th.surface, border:`1px solid ${th.border}`, borderRadius:"8px", padding:"6px 10px", cursor:"pointer", fontSize:"16px" }}>{darkMode ? "☀️" : "🌙"}</button>
                <button onClick={newChat} style={{ background:`linear-gradient(135deg,${personality.color},#7b2fff)`, border:"none", borderRadius:"8px", padding:"6px 10px", cursor:"pointer", fontSize:"12px", color:"#fff", fontWeight:"600" }}>+ New</button>
              </div>
            </div>

            {/* Personalities dropdown */}
            {showPersonalities && (
              <div style={{ marginTop:"10px", background:th.surface, border:`1px solid ${th.border}`, borderRadius:"12px", padding:"12px", display:"flex", flexDirection:"column", gap:"8px" }}>
                <div style={{ fontSize:"12px", color:th.subtext }}>Choose AI Personality:</div>
                {PERSONALITIES.map(p => (
                  <button key={p.id} onClick={() => { setPersonality(p); setShowPersonalities(false); }}
                    style={{ display:"flex", alignItems:"center", gap:"10px", background:personality.id===p.id ? `${p.color}22` : "transparent", border:`1px solid ${personality.id===p.id ? p.color : th.border}`, borderRadius:"8px", padding:"8px 12px", cursor:"pointer", textAlign:"left" }}>
                    <span style={{ fontSize:"20px" }}>{p.emoji}</span>
                    <div>
                      <div style={{ fontSize:"13px", fontWeight:"600", color:p.color }}>{p.name}</div>
                      <div style={{ fontSize:"11px", color:th.subtext }}>{p.prompt.slice(0,55)}…</div>
                    </div>
                  </button>
                ))}
              </div>
            )}

            {/* History dropdown */}
            {showHistory && (
              <div style={{ marginTop:"10px", background:th.surface, border:`1px solid ${th.border}`, borderRadius:"12px", padding:"12px" }}>
                <div style={{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:"8px" }}>
                  <div style={{ fontSize:"12px", color:th.subtext }}>Saved Chats ({sessionKeys.length})</div>
                  <button onClick={clearAll} style={{ background:"none", border:"none", color:"#ff5555", fontSize:"12px", cursor:"pointer" }}>Clear All</button>
                </div>
                {sessionKeys.length === 0 && <div style={{ fontSize:"13px", color:th.subtext }}>No saved chats yet.</div>}
                {sessionKeys.map(key => {
                  const msgs = sessions[key];
                  const preview = msgs?.find(m => m.role==="user")?.content || "Empty chat";
                  return (
                    <button key={key} onClick={() => { setCurrentSession(key); setShowHistory(false); }}
                      style={{ display:"block", width:"100%", textAlign:"left", background:currentSession===key ? `${personality.color}22` : "transparent", border:`1px solid ${currentSession===key ? personality.color : th.border}`, borderRadius:"8px", padding:"8px 12px", cursor:"pointer", marginBottom:"6px" }}>
                      <div style={{ fontSize:"12px", color:th.text, overflow:"hidden", textOverflow:"ellipsis", whiteSpace:"nowrap" }}>💬 {preview.slice(0,40)}…</div>
                      <div style={{ fontSize:"10px", color:th.subtext }}>{msgs?.length||0} messages</div>
                    </button>
                  );
                })}
              </div>
            )}
          </div>

          {/* ── Chat window ── */}
          <div style={{ width:"100%", maxWidth:"720px", background:th.surface, border:`1px solid ${th.border}`, borderRadius:"16px", padding:"16px", overflowY:"auto", flex:1, minHeight:0, display:"flex", flexDirection:"column", gap:"12px", backdropFilter:"blur(10px)" }}>
            {messages.map((msg, i) => (
              <div key={i} style={{ display:"flex", justifyContent:msg.role==="user"?"flex-end":"flex-start", animation:"fadeIn 0.2s ease" }}>
                {msg.role==="assistant" && (
                  <div style={{ width:"26px", height:"26px", background:`linear-gradient(135deg,${personality.color},#7b2fff)`, borderRadius:"50%", display:"flex", alignItems:"center", justifyContent:"center", fontSize:"12px", marginRight:"8px", flexShrink:0, marginTop:"2px" }}>{personality.emoji}</div>
                )}
                <div style={{ maxWidth:"75%", padding:"10px 14px", borderRadius:msg.role==="user"?"18px 18px 4px 18px":"18px 18px 18px 4px", background:msg.role==="user"?`linear-gradient(135deg,#7b2fff,${personality.color})`:th.bubbleAI, border:msg.role==="user"?"none":`1px solid ${th.border}`, color:msg.role==="user"?"#fff":th.text, fontSize:"14px", lineHeight:"1.6", whiteSpace:"pre-wrap", wordBreak:"break-word" }}>{msg.content}</div>
                {msg.role==="user" && (
                  user.avatarIsUrl
                    ? <img src={user.avatar} alt="" style={{ width:"26px", height:"26px", borderRadius:"50%", marginLeft:"8px", flexShrink:0, marginTop:"2px" }} />
                    : <div style={{ width:"26px", height:"26px", borderRadius:"50%", background:user.avatarColor||"#7b2fff", display:"flex", alignItems:"center", justifyContent:"center", fontSize:"12px", fontWeight:"700", color:"#fff", marginLeft:"8px", flexShrink:0, marginTop:"2px" }}>{user.avatar}</div>
                )}
              </div>
            ))}
            {loading && (
              <div style={{ display:"flex", alignItems:"center", gap:"8px" }}>
                <div style={{ width:"26px", height:"26px", background:`linear-gradient(135deg,${personality.color},#7b2fff)`, borderRadius:"50%", display:"flex", alignItems:"center", justifyContent:"center", fontSize:"12px" }}>{personality.emoji}</div>
                <div style={{ padding:"10px 14px", background:th.bubbleAI, border:`1px solid ${th.border}`, borderRadius:"18px 18px 18px 4px", display:"flex", gap:"5px", alignItems:"center" }}>
                  {[0,1,2].map(d => <div key={d} style={{ width:"6px", height:"6px", borderRadius:"50%", background:personality.color, animation:`pulse 1.2s ease-in-out ${d*0.2}s infinite` }} />)}
                </div>
              </div>
            )}
            <div ref={bottomRef} />
          </div>

          {/* ── Suggestions ── */}
          <div style={{ width:"100%", maxWidth:"720px", display:"flex", gap:"6px", flexWrap:"wrap", marginTop:"8px", flexShrink:0 }}>
            {["What is deep learning?", "Write Python CNN code", "Explain transformers", "What is backpropagation?"].map(s => (
              <button key={s} onClick={() => setInput(s)} style={{ background:th.surface, border:`1px solid ${personality.color}44`, color:personality.color, borderRadius:"20px", padding:"4px 10px", fontSize:"11px", cursor:"pointer" }}>{s}</button>
            ))}
          </div>

          {/* ── Input bar ── */}
          <div style={{ width:"100%", maxWidth:"720px", display:"flex", gap:"8px", alignItems:"flex-end", marginTop:"8px", flexShrink:0 }}>
            <button onClick={listening ? stopVoice : startVoice} style={{ padding:"12px", borderRadius:"12px", background:listening?"linear-gradient(135deg,#ff4444,#ff0000)":th.surface, border:`1px solid ${listening?"#ff4444":th.border}`, cursor:"pointer", fontSize:"18px" }}>{listening ? "🔴" : "🎤"}</button>
            <textarea value={input} onChange={e => setInput(e.target.value)} onKeyDown={handleKey} placeholder={listening ? "Listening… 🎤" : "Type or speak…"} rows={1}
              style={{ flex:1, padding:"12px 14px", borderRadius:"12px", border:`1px solid ${th.border}`, background:th.inputBg, color:th.text, fontSize:"14px", outline:"none", resize:"none", fontFamily:"inherit", lineHeight:"1.5" }} />
            <button onClick={sendMessage} disabled={loading || !input.trim()} style={{ padding:"12px 18px", borderRadius:"12px", border:`1px solid ${th.border}`, background:loading||!input.trim() ? th.surface : `linear-gradient(135deg,#7b2fff,${personality.color})`, color:loading||!input.trim() ? th.subtext : "#fff", fontWeight:"600", fontSize:"14px", cursor:loading||!input.trim()?"not-allowed":"pointer" }}>
              {loading ? "…" : "Send ↑"}
            </button>
          </div>

          <p style={{ color:th.subtext, fontSize:"11px", marginTop:"6px", flexShrink:0 }}>Click avatar to sign out · 🎤 Voice · 🤖 Personalities · 🗂️ History</p>
        </div>
      );
    }

    ReactDOM.createRoot(document.getElementById("root")).render(<App />);
  </script>
</body>
</html>
