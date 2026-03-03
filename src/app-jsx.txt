import { useState, useCallback } from "react";

// ─── Config (injected at build time via Vite env vars) ────────
const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;
const TEMPLATE_ID = import.meta.env.VITE_TEMPLATE_ID;
const SCOPES = [
  "https://www.googleapis.com/auth/presentations",
  "https://www.googleapis.com/auth/drive.file",
  "https://www.googleapis.com/auth/drive.readonly",
].join(" ");

// ─── Brand tokens ─────────────────────────────────────────────
const CORAL  = "#FF492C";
const DARK   = "#1A1A2E";
const GRAY   = "#F4F4F6";
const BORDER = "#E2E2E8";
const TEXT   = "#2D2D3A";
const MUTED  = "#6B6B80";

// ─── Shared UI ────────────────────────────────────────────────
const Label = ({ children, sub }) => (
  <div style={{ marginBottom: 6 }}>
    <div style={{ fontWeight: 600, fontSize: 13, color: TEXT }}>{children}</div>
    {sub && <div style={{ fontSize: 11, color: MUTED, marginTop: 2 }}>{sub}</div>}
  </div>
);

const inputBase = {
  width: "100%", boxSizing: "border-box", padding: "10px 14px",
  border: `1.5px solid ${BORDER}`, borderRadius: 8, fontSize: 14,
  color: TEXT, background: "#fff", outline: "none", fontFamily: "inherit",
  transition: "border 0.15s",
};
const focusOn  = e => (e.target.style.border = `1.5px solid ${CORAL}`);
const focusOff = e => (e.target.style.border = `1.5px solid ${BORDER}`);

const Input    = (p) => <input    {...p} style={{ ...inputBase, ...p.style }} onFocus={focusOn} onBlur={focusOff} />;
const Textarea = (p) => <textarea {...p} style={{ ...inputBase, resize: "vertical", lineHeight: 1.5, minHeight: 90, ...p.style }} onFocus={focusOn} onBlur={focusOff} />;

const Select = ({ children, ...p }) => (
  <select {...p} style={{
    ...inputBase, appearance: "none",
    backgroundImage: `url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%236B6B80' d='M6 8L1 3h10z'/%3E%3C/svg%3E")`,
    backgroundRepeat: "no-repeat", backgroundPosition: "right 14px center", ...p.style,
  }}>{children}</select>
);

const Btn = ({ children, onClick, disabled, variant = "primary", style = {} }) => {
  const variants = {
    primary: { background: disabled ? "#ccc" : CORAL, color: "#fff" },
    outline:  { background: "#fff", color: CORAL, border: `1.5px solid ${CORAL}` },
    ghost:    { background: GRAY, color: TEXT },
  };
  return (
    <button onClick={disabled ? undefined : onClick} style={{
      padding: "11px 24px", borderRadius: 8, fontWeight: 700, fontSize: 14,
      cursor: disabled ? "not-allowed" : "pointer", border: "none",
      transition: "all 0.15s", fontFamily: "inherit",
      ...variants[variant], ...style,
    }}>{children}</button>
  );
};

const Tag = ({ label, onRemove }) => (
  <span style={{
    display: "inline-flex", alignItems: "center", gap: 6,
    background: "#FFF0EE", color: CORAL, borderRadius: 6,
    padding: "4px 10px", fontSize: 13, fontWeight: 600,
  }}>
    {label}
    <span onClick={onRemove} style={{ cursor: "pointer", fontSize: 16, lineHeight: 1 }}>×</span>
  </span>
);

const ProgressBar = ({ steps, current }) => (
  <div style={{ marginBottom: 16 }}>
    <div style={{ display: "flex", borderRadius: 8, overflow: "hidden", height: 6 }}>
      {steps.map((_, i) => (
        <div key={i} style={{
          flex: 1,
          background: i < current ? CORAL : i === current ? `${CORAL}66` : BORDER,
          transition: "background 0.3s",
        }} />
      ))}
    </div>
    <div style={{ fontSize: 12, color: MUTED, marginTop: 6 }}>
      {steps[current]} ({current + 1}/{steps.length})
    </div>
  </div>
);

const LogConsole = ({ entries }) => (
  <div style={{
    background: DARK, borderRadius: 10, padding: "14px 18px",
    fontFamily: "monospace", fontSize: 12, color: "#A8FFB0",
  }}>
    {entries.map((l, i) => <div key={i} style={{ marginBottom: 3 }}>{l}</div>)}
  </div>
);

const ErrorBox = ({ msg }) => msg ? (
  <div style={{
    background: "#FFF1F0", border: "1px solid #FECACA", borderRadius: 10,
    padding: "12px 16px", marginBottom: 16, fontSize: 13, color: "#DC2626",
  }}>❌ {msg}</div>
) : null;

const SuccessBox = ({ children }) => (
  <div style={{
    background: "#F0FDF4", border: "1px solid #BBF7D0", borderRadius: 10,
    padding: "16px 20px", marginBottom: 20,
  }}>{children}</div>
);

const OpenBtn = ({ id, label }) => (
  <a href={`https://docs.google.com/presentation/d/${id}/edit`}
    target="_blank" rel="noreferrer"
    style={{
      display: "inline-flex", alignItems: "center", gap: 8,
      background: CORAL, color: "#fff", borderRadius: 8,
      padding: "10px 20px", fontWeight: 700, fontSize: 14, textDecoration: "none",
    }}>
    {label} ↗
  </a>
);

// ─── Google OAuth hook ────────────────────────────────────────
const useGoogleAuth = () => {
  const [token, setToken] = useState(null);
  const [user,  setUser]  = useState(null);
  const [loading, setLoading] = useState(false);

  const signIn = useCallback(async () => {
    setLoading(true);
    try {
      // google script loaded in index.html
      const client = window.google.accounts.oauth2.initTokenClient({
        client_id: CLIENT_ID,
        scope: SCOPES,
        callback: async (resp) => {
          if (resp.error) { setLoading(false); return; }
          setToken(resp.access_token);
          try {
            const r = await fetch("https://www.googleapis.com/oauth2/v3/userinfo", {
              headers: { Authorization: `Bearer ${resp.access_token}` },
            });
            setUser(await r.json());
          } catch {}
          setLoading(false);
        },
      });
      client.requestAccessToken();
    } catch { setLoading(false); }
  }, []);

  const signOut = useCallback(() => {
    if (token && window.google) window.google.accounts.oauth2.revoke(token);
    setToken(null);
    setUser(null);
  }, [token]);

  return { token, user, loading, signIn, signOut };
};

// ─── Google Slides API helpers ────────────────────────────────
const slidesApi = (token) => {
  const headers = { Authorization: `Bearer ${token}`, "Content-Type": "application/json" };

  const get = (url) => fetch(url, { headers }).then(async r => {
    if (!r.ok) { const e = await r.json(); throw new Error(e.error?.message || r.status); }
    return r.json();
  });

  const post = (url, body) => fetch(url, { method: "POST", headers, body: JSON.stringify(body) }).then(async r => {
    if (!r.ok) { const e = await r.json(); throw new Error(e.error?.message || r.status); }
    return r.json();
  });

  return {
    getPresentation: (id) => get(`https://slides.googleapis.com/v1/presentations/${id}`),
    copyFile: (id, name) => post(`https://www.googleapis.com/drive/v3/files/${id}/copy`, { name }),
    batchUpdate: (id, requests) => post(`https://slides.googleapis.com/v1/presentations/${id}:batchUpdate`, { requests }),
    deleteAllSlides: async (presId, pres) => {
      const ids = (pres.slides || []).map(s => s.objectId);
      if (!ids.length) return;
      await post(`https://slides.googleapis.com/v1/presentations/${presId}:batchUpdate`, {
        requests: ids.map(id => ({ deleteObject: { objectId: id } })),
      });
    },
  };
};

// ─── Template layout inspector ────────────────────────────────
const inspectTemplate = (pres) => {
  const layouts = {};
  (pres.layouts || []).forEach(l => {
    const name = l.layoutProperties?.displayName || l.objectId;
    layouts[name] = {
      objectId: l.objectId,
      placeholders: (l.placeholders || []).map(p => ({ idx: p.idx, type: p.type, objectId: p.objectId })),
    };
  });
  return { layouts };
};

// ─── Slide request builder ────────────────────────────────────
const EMU = { W: 9144000, H: 5143500 }; // 10×7.5 in standard widescreen

const rgbHex = (hex) => {
  const n = parseInt(hex.replace("#", ""), 16);
  return { red: ((n >> 16) & 255) / 255, green: ((n >> 8) & 255) / 255, blue: (n & 255) / 255 };
};

const textBox = (pageId, id, text, x, y, w, h, opts = {}) => {
  const { fontSize = 16, bold = false, colorHex = TEXT, italic = false } = opts;
  return [
    { createShape: { objectId: id, shapeType: "TEXT_BOX", elementProperties: {
      pageObjectId: pageId,
      size: { width: { magnitude: w, unit: "EMU" }, height: { magnitude: h, unit: "EMU" } },
      transform: { scaleX: 1, scaleY: 1, translateX: x, translateY: y, unit: "EMU" },
    }}},
    { insertText: { objectId: id, text } },
    { updateTextStyle: { objectId: id, textRange: { type: "ALL" }, fields: "bold,italic,fontSize,foregroundColor", style: {
      bold, italic,
      fontSize: { magnitude: fontSize, unit: "PT" },
      foregroundColor: { opaqueColor: { rgbColor: rgbHex(colorHex) } },
    }}},
    { updateParagraphStyle: { objectId: id, textRange: { type: "ALL" }, fields: "lineSpacing,spaceAbove,spaceBelow", style: {
      lineSpacing: 140,
      spaceAbove: { magnitude: 0, unit: "PT" },
      spaceBelow: { magnitude: 6, unit: "PT" },
    }}},
  ];
};

const buildSlideRequests = (slides, template) => {
  const layoutNames = Object.keys(template.layouts);
  const pickLayout = (hint = "") => {
    const h = hint.toLowerCase();
    const match = layoutNames.find(n => n.toLowerCase().includes(h));
    return (match ? template.layouts[match] : template.layouts[layoutNames[0]])?.objectId;
  };

  const reqs = [];
  const ts = Date.now();

  slides.forEach((slide, idx) => {
    const sid  = `sl_${idx}_${ts}`;
    const layoutId = pickLayout(slide.layoutHint);

    reqs.push({ createSlide: { objectId: sid, insertionIndex: idx, slideLayoutReference: { layoutId } } });

    // Background
    const bgColor = slide.isTitle ? DARK : "#FFFFFF";
    reqs.push({ updatePageProperties: { objectId: sid, fields: "pageBackgroundFill", pageProperties: {
      pageBackgroundFill: { solidFill: { color: { rgbColor: rgbHex(bgColor) } } },
    }}});

    // Coral accent bar (top)
    const barId = `bar_${idx}_${ts}`;
    reqs.push({ createShape: { objectId: barId, shapeType: "RECTANGLE", elementProperties: {
      pageObjectId: sid,
      size: { width: { magnitude: EMU.W, unit: "EMU" }, height: { magnitude: 45000, unit: "EMU" } },
      transform: { scaleX: 1, scaleY: 1, translateX: 0, translateY: 0, unit: "EMU" },
    }}});
    reqs.push({ updateShapeProperties: { objectId: barId, fields: "shapeBackgroundFill,outline", shapeProperties: {
      shapeBackgroundFill: { solidFill: { color: { rgbColor: rgbHex(CORAL) } } },
      outline: { propertyState: "NOT_RENDERED" },
    }}});

    // Title
    const titleColor = slide.isTitle ? "#FFFFFF" : DARK;
    const titleY     = slide.isTitle ? EMU.H * 0.28 : 200000;
    const titleSize  = slide.isTitle ? 44 : 30;
    reqs.push(...textBox(sid, `ttl_${idx}_${ts}`, slide.title || "",
      457200, titleY, EMU.W - 914400, 1200000,
      { fontSize: titleSize, bold: true, colorHex: titleColor }
    ));

    // Body
    if (slide.body) {
      const bodyColor = slide.isTitle ? "#CCCCDD" : TEXT;
      const bodyY     = slide.isTitle ? EMU.H * 0.55 : 1600000;
      reqs.push(...textBox(sid, `bdy_${idx}_${ts}`, slide.body,
        457200, bodyY, EMU.W - 914400, 3000000,
        { fontSize: slide.isTitle ? 18 : 15, colorHex: bodyColor }
      ));
    }

    // Slide number (non-title slides)
    if (!slide.isTitle) {
      reqs.push(...textBox(sid, `num_${idx}_${ts}`, String(idx),
        EMU.W - 600000, EMU.H - 400000, 500000, 300000,
        { fontSize: 10, colorHex: MUTED }
      ));
    }
  });

  return reqs;
};

// ─── Shared form helpers ──────────────────────────────────────
const G2_PLANS = ["Essentials", "Advanced", "Professional", "Enterprise", "Ultimate"];
const G2_GOALS = [
  "Drive pipeline / demand gen", "Improve win rates",
  "Competitive displacement", "Customer retention & expansion",
  "Build brand credibility", "Analyst & PR support",
];

const useLog = () => {
  const [log, setLog] = useState([]);
  const add = (msg) => setLog(l => [...l, msg]);
  const clear = () => setLog([]);
  return { log, add, clear };
};

// ─── Auth Banner ──────────────────────────────────────────────
const AuthBanner = ({ user, loading, onSignIn, onSignOut }) => (
  <div style={{
    display: "flex", alignItems: "center", justifyContent: "space-between",
    padding: "12px 20px", marginBottom: 24, borderRadius: 10,
    background: user ? "#F0FDF4" : "#FFF8F7",
    border: `1px solid ${user ? "#BBF7D0" : "#FDDDD8"}`,
  }}>
    <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
      <div style={{ fontSize: 18 }}>{user ? "✅" : "🔗"}</div>
      <div>
        <div style={{ fontWeight: 600, fontSize: 13, color: user ? "#15803D" : "#7A2718" }}>
          {user ? `Connected as ${user.email}` : "Google account not connected"}
        </div>
        <div style={{ fontSize: 11, color: MUTED }}>
          {user ? "Decks will be created in your Google Drive" : "Required to create and save Google Slides"}
        </div>
      </div>
    </div>
    {user
      ? <Btn variant="ghost" onClick={onSignOut} style={{ fontSize: 12, padding: "6px 14px" }}>Disconnect</Btn>
      : <Btn variant="outline" onClick={onSignIn} disabled={loading} style={{ fontSize: 13, padding: "8px 18px" }}>
          {loading ? "Connecting…" : "Connect Google Account"}
        </Btn>
    }
  </div>
);

// ─── Reskin Tab ───────────────────────────────────────────────
const RESKIN_STEPS = ["Reading source deck", "Mapping to G2 brand", "Copying template", "Applying changes", "Done ✓"];

const ReskinTab = ({ token }) => {
  const [url,   setUrl]   = useState("");
  const [notes, setNotes] = useState("");
  const [step,  setStep]  = useState(-1);
  const [result, setResult] = useState(null);
  const [error,  setError]  = useState(null);
  const { log, add, clear } = useLog();

  const busy = step >= 0 && step < RESKIN_STEPS.length - 1;
  const done = step === RESKIN_STEPS.length - 1;
  const validUrl = url.includes("docs.google.com/presentation");
  const srcId = url.match(/\/presentation\/d\/([a-zA-Z0-9_-]+)/)?.[1];

  const reset = () => { setStep(-1); clear(); setResult(null); setError(null); };

  const run = async () => {
    if (!token || !srcId) return;
    reset();
    const api = slidesApi(token);
    try {
      // 0 — read source
      setStep(0); add("Reading source presentation…");
      const src = await api.getPresentation(srcId);
      const slideList = (src.slides || []).map((s, i) => {
        const title = s.pageElements
          ?.find(e => e.shape?.placeholder?.type === "TITLE")
          ?.shape?.text?.textElements?.map(t => t.textRun?.content || "").join("") || `Slide ${i + 1}`;
        return `Slide ${i + 1}: "${title.trim()}"`;
      });
      add(`Found ${slideList.length} slides.`);

      // 1 — Claude mapping
      setStep(1); add("Mapping slides to G2 brand template via Claude…");
      const claudeRes = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          messages: [{ role: "user", content: `You are a G2 brand designer. Map each slide to the G2 brand system.

Slides:
${slideList.join("\n")}

Context: ${notes || "none"}

G2 Brand: Coral #FF492C, Navy #1A1A2E, clean bold sans-serif typography.
Layout options: title, section, data, comparison, quote, cta, blank

Return ONLY a JSON array — no markdown, no explanation:
[{"slideNum":1,"originalTitle":"...","suggestedTitle":"...","layoutHint":"title","body":"key points for this slide","flag":"describe any mapping complication or null"}]` }]
        }),
      });
      const cd = await claudeRes.json();
      const raw = cd.content?.map(b => b.text || "").join("") || "[]";
      let mapping = [];
      try { mapping = JSON.parse(raw.match(/\[[\s\S]*\]/)?.[0] || "[]"); } catch {}
      const flags = mapping.filter(m => m.flag).map(m => `Slide ${m.slideNum} ("${m.originalTitle}"): ${m.flag}`);
      add(`Mapped ${mapping.length} slides. ${flags.length} flag(s).`);

      // 2 — copy template
      setStep(2); add("Copying G2 master template from Drive…");
      const copy = await api.copyFile(TEMPLATE_ID, `[G2 Reskin] ${src.title || "Untitled"}`);
      add(`Created: "${copy.name}"`);

      // 3 — write slides
      setStep(3); add("Applying brand layout and styles…");
      const tmplPres = await api.getPresentation(copy.id);
      const tmpl     = inspectTemplate(tmplPres);
      await api.deleteAllSlides(copy.id, tmplPres);

      const slides = mapping.map((m, i) => ({
        title:      m.suggestedTitle || m.originalTitle,
        body:       m.body || "",
        layoutHint: m.layoutHint || "blank",
        isTitle:    i === 0,
      }));
      const reqs = buildSlideRequests(slides, tmpl);
      for (let i = 0; i < reqs.length; i += 50)
        await api.batchUpdate(copy.id, reqs.slice(i, i + 50));

      add("Brand styles applied.");
      setStep(RESKIN_STEPS.length - 1);
      setResult({ id: copy.id, name: copy.name, flags });
      add("✅ Reskin complete!");
    } catch (e) { setError(e.message); add(`❌ ${e.message}`); }
  };

  return (
    <div>
      {!token && <div style={{ padding: "14px 18px", background: GRAY, borderRadius: 10, fontSize: 13, color: MUTED, marginBottom: 20 }}>Connect your Google account to enable reskinning.</div>}

      <div style={{ marginBottom: 16 }}>
        <Label sub="Must be a Google Slides URL you have view/edit access to">Source Google Slides URL</Label>
        <Input placeholder="https://docs.google.com/presentation/d/…" value={url} onChange={e => setUrl(e.target.value)} />
        {url && !validUrl && <div style={{ color: "#DC2626", fontSize: 12, marginTop: 5 }}>Please enter a valid Google Slides URL.</div>}
      </div>

      <div style={{ marginBottom: 24 }}>
        <Label sub="Optional — describe the deck's audience, purpose, or any slide-specific notes">Additional Context</Label>
        <Textarea placeholder="e.g. 12-slide sales deck for mid-market prospects. Slide 5 has a complex comparison table…" value={notes} onChange={e => setNotes(e.target.value)} style={{ minHeight: 72 }} />
      </div>

      {log.length > 0 && (
        <div style={{ marginBottom: 20 }}>
          <ProgressBar steps={RESKIN_STEPS} current={Math.max(step, 0)} />
          <LogConsole entries={log} />
        </div>
      )}

      {result && (
        <SuccessBox>
          <div style={{ fontWeight: 700, color: "#15803D", marginBottom: 6 }}>✅ Reskin Complete</div>
          {result.flags.length > 0 && (
            <div style={{ marginBottom: 12, padding: "10px 14px", background: "#FFFBEB", borderRadius: 8, border: "1px solid #FDE68A" }}>
              <div style={{ fontWeight: 600, fontSize: 12, color: "#92400E", marginBottom: 6 }}>⚠️ Flags to Review Before Sending</div>
              {result.flags.map((f, i) => <div key={i} style={{ fontSize: 12, color: "#78350F", marginBottom: 2 }}>• {f}</div>)}
            </div>
          )}
          <OpenBtn id={result.id} label="Open Reskinned Deck" />
        </SuccessBox>
      )}

      <ErrorBox msg={error} />

      <div style={{ display: "flex", gap: 12 }}>
        <Btn onClick={run} disabled={!token || !validUrl || busy}>
          {busy ? "Working…" : "Analyze & Reskin →"}
        </Btn>
        {done && <Btn variant="ghost" onClick={reset}>Start Over</Btn>}
      </div>
    </div>
  );
};

// ─── Generate Tab ─────────────────────────────────────────────
const GEN_STEPS = ["Pulling G2 data", "Building deck plan", "Copying template", "Writing slides", "Done ✓"];

const GenerateTab = ({ token }) => {
  const [company,   setCompany]   = useState("");
  const [plan,      setPlan]      = useState("");
  const [goal,      setGoal]      = useState("");
  const [compInput, setCompInput] = useState("");
  const [competitors, setCompetitors] = useState([]);
  const [notes, setNotes] = useState("");
  const [step,  setStep]  = useState(-1);
  const [result, setResult] = useState(null);
  const [error,  setError]  = useState(null);
  const { log, add, clear } = useLog();

  const busy = step >= 0 && step < GEN_STEPS.length - 1;
  const done = step === GEN_STEPS.length - 1;
  const canRun = token && company.trim() && plan && goal && !busy;

  const reset = () => { setStep(-1); clear(); setResult(null); setError(null); };

  const addComp = () => {
    const v = compInput.trim();
    if (v && !competitors.includes(v)) setCompetitors(c => [...c, v]);
    setCompInput("");
  };

  const run = async () => {
    reset();
    const api = slidesApi(token);
    try {
      // 0 — Claude + G2 MCP
      setStep(0); add(`Pulling G2 data for ${company}…`);
      const claudeRes = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          messages: [{ role: "user", content: `You are a G2 sales expert. Generate a complete branded slide deck for a sales rep presenting G2 to ${company}.

Company: ${company}
G2 Plan: ${plan}
Primary Goal: ${goal}
Competitors: ${competitors.join(", ") || "auto-identify top 3-5 in category"}
Rep Notes: ${notes || "none"}

Return ONLY a JSON array of slides — no markdown, no preamble:
[
  {
    "slideNum": 1,
    "title": "slide title",
    "body": "line 1\nline 2\nline 3",
    "layoutHint": "title|section|data|comparison|quote|cta|blank",
    "isTitle": true,
    "speakerNotes": "what the rep should say"
  }
]

Required slides (11 total):
1. Title: "${company} × G2" (isTitle: true)
2. Why G2 Matters — market credibility in ${company}'s category
3. ${company}'s Current G2 Presence — reviews, rating, badges, category rank (use realistic G2 metric structures)
4. Competitive Comparison — ${company} vs ${competitors.slice(0,3).join(", ") || "top competitors"} (review count, avg rating, market presence, momentum score)
5. Review Velocity & Trend
6. How Buyers Use G2 in This Category — intent data angle
7. G2 ${plan} Plan — features mapped to goal: "${goal}"
8. ROI & Expected Impact — benchmarks from similar customers
9. G2 Customer Success Story — similar company in same category
10. Recommended Action Plan & Next Steps
11. Q&A / Thank You

Make body content specific and data-rich. Use realistic G2 metrics (star ratings out of 5, review counts, market presence scores 0-100, momentum scores, etc).` }],
          mcp_servers: [{ type: "url", url: "https://mcp.g2.com/mcp", name: "g2-mcp" }],
        }),
      });
      const cd = await claudeRes.json();
      const raw = cd.content?.filter(b => b.type === "text").map(b => b.text).join("") || "[]";
      const g2Hits = cd.content?.filter(b => b.type === "mcp_tool_result").length || 0;
      add(`G2 MCP returned ${g2Hits} data point(s).`);

      setStep(1); add("Parsing deck plan…");
      let slides = [];
      try { slides = JSON.parse(raw.match(/\[[\s\S]*\]/)?.[0] || "[]"); } catch {}
      if (!slides.length) throw new Error("Claude did not return valid slide JSON. Try again.");
      add(`Deck plan ready: ${slides.length} slides.`);

      // 2 — copy template
      setStep(2); add("Copying G2 master template…");
      const name = `G2 Sales Deck — ${company} — ${new Date().toLocaleDateString("en-US", { month: "short", day: "numeric", year: "numeric" })}`;
      const copy = await api.copyFile(TEMPLATE_ID, name);
      add(`Created: "${copy.name}"`);

      // 3 — write slides
      setStep(3); add(`Writing ${slides.length} slides to Google Slides…`);
      const tmplPres = await api.getPresentation(copy.id);
      const tmpl     = inspectTemplate(tmplPres);
      await api.deleteAllSlides(copy.id, tmplPres);

      const reqs = buildSlideRequests(slides, tmpl);
      for (let i = 0; i < reqs.length; i += 50)
        await api.batchUpdate(copy.id, reqs.slice(i, i + 50));

      setStep(GEN_STEPS.length - 1);
      setResult({ id: copy.id, name: copy.name, count: slides.length });
      add(`✅ ${slides.length}-slide deck created!`);
    } catch (e) { setError(e.message); add(`❌ ${e.message}`); }
  };

  return (
    <div>
      {!token && <div style={{ padding: "14px 18px", background: GRAY, borderRadius: 10, fontSize: 13, color: MUTED, marginBottom: 20 }}>Connect your Google account to enable deck creation.</div>}

      <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 16, marginBottom: 16 }}>
        <div>
          <Label>Prospect Company *</Label>
          <Input placeholder="e.g. Salesforce, Gong, Notion…" value={company} onChange={e => setCompany(e.target.value)} />
        </div>
        <div>
          <Label>Recommended G2 Plan *</Label>
          <Select value={plan} onChange={e => setPlan(e.target.value)}>
            <option value="">Select a plan…</option>
            {G2_PLANS.map(p => <option key={p}>{p}</option>)}
          </Select>
        </div>
      </div>

      <div style={{ marginBottom: 16 }}>
        <Label>Primary Goal with G2 *</Label>
        <Select value={goal} onChange={e => setGoal(e.target.value)}>
          <option value="">Select primary goal…</option>
          {G2_GOALS.map(g => <option key={g}>{g}</option>)}
        </Select>
      </div>

      <div style={{ marginBottom: 16 }}>
        <Label sub="Press Enter or click Add — leave blank to auto-identify from G2 data">Primary Competitors</Label>
        <div style={{ display: "flex", gap: 8, marginBottom: 8 }}>
          <Input placeholder="Competitor name…" value={compInput}
            onChange={e => setCompInput(e.target.value)}
            onKeyDown={e => { if (e.key === "Enter") { e.preventDefault(); addComp(); } }}
            style={{ flex: 1 }} />
          <Btn variant="outline" onClick={addComp} style={{ whiteSpace: "nowrap", padding: "10px 16px" }}>+ Add</Btn>
        </div>
        {competitors.length > 0 && (
          <div style={{ display: "flex", flexWrap: "wrap", gap: 8 }}>
            {competitors.map(c => <Tag key={c} label={c} onRemove={() => setCompetitors(cs => cs.filter(x => x !== c))} />)}
          </div>
        )}
        {!competitors.length && <div style={{ fontSize: 12, color: MUTED }}>Leave blank to auto-identify from G2.</div>}
      </div>

      <div style={{ marginBottom: 24 }}>
        <Label sub="Deal context, pain points, messaging angles, objections to address">Rep Notes & Deal Context</Label>
        <Textarea placeholder="e.g. Losing deals to Competitor X on brand credibility. Champion is VP Marketing, renewal Q3…" value={notes} onChange={e => setNotes(e.target.value)} />
      </div>

      {log.length > 0 && (
        <div style={{ marginBottom: 20 }}>
          <ProgressBar steps={GEN_STEPS} current={Math.max(step, 0)} />
          <LogConsole entries={log} />
        </div>
      )}

      {result && (
        <SuccessBox>
          <div style={{ fontWeight: 700, color: "#15803D", marginBottom: 4 }}>✅ {result.count}-Slide Deck Created</div>
          <div style={{ fontSize: 13, color: MUTED, marginBottom: 14 }}>{result.name}</div>
          <OpenBtn id={result.id} label="Open in Google Slides" />
        </SuccessBox>
      )}

      <ErrorBox msg={error} />

      <div style={{ display: "flex", gap: 12 }}>
        <Btn onClick={run} disabled={!canRun}>
          {busy ? "Building…" : `Generate ${company || "Prospect"} × G2 Deck →`}
        </Btn>
        {done && <Btn variant="ghost" onClick={reset}>New Deck</Btn>}
      </div>
    </div>
  );
};

// ─── App root ─────────────────────────────────────────────────
export default function App() {
  const [tab, setTab] = useState("reskin");
  const { token, user, loading, signIn, signOut } = useGoogleAuth();

  const tabs = [
    { id: "reskin",   label: "🎨  Reskin Existing Deck", sub: "Map any deck to G2 brand" },
    { id: "generate", label: "📊  Generate G2 Sales Deck", sub: "Build from live G2 data" },
  ];

  return (
    <div style={{ minHeight: "100vh", background: GRAY, fontFamily: "'Inter', system-ui, sans-serif", padding: "32px 20px" }}>
      <div style={{ maxWidth: 780, margin: "0 auto" }}>

        <div style={{ marginBottom: 28, display: "flex", alignItems: "center", gap: 16 }}>
          <div style={{ width: 44, height: 44, background: CORAL, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center" }}>
            <span style={{ color: "#fff", fontWeight: 900, fontSize: 18 }}>G2</span>
          </div>
          <div>
            <div style={{ fontWeight: 800, fontSize: 22, color: DARK, letterSpacing: "-0.5px" }}>G2 Slide Studio</div>
            <div style={{ fontSize: 13, color: MUTED }}>Brand-compliant presentations, powered by G2 data</div>
          </div>
        </div>

        <AuthBanner user={user} loading={loading} onSignIn={signIn} onSignOut={signOut} />

        <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 12, marginBottom: 24 }}>
          {tabs.map(t => (
            <button key={t.id} onClick={() => setTab(t.id)} style={{
              padding: "16px 20px", borderRadius: 12, border: "none", cursor: "pointer",
              textAlign: "left", fontFamily: "inherit", transition: "all 0.15s",
              background: tab === t.id ? CORAL : "#fff",
              color: tab === t.id ? "#fff" : TEXT,
              boxShadow: tab === t.id ? `0 4px 16px ${CORAL}44` : "0 1px 4px rgba(0,0,0,0.08)",
              transform: tab === t.id ? "translateY(-1px)" : "none",
            }}>
              <div style={{ fontWeight: 700, fontSize: 14 }}>{t.label}</div>
              <div style={{ fontSize: 12, marginTop: 3, opacity: tab === t.id ? 0.85 : 0.6 }}>{t.sub}</div>
            </button>
          ))}
        </div>

        <div style={{ background: "#fff", borderRadius: 16, border: `1.5px solid ${BORDER}`, padding: 32, boxShadow: "0 2px 12px rgba(0,0,0,0.06)" }}>
          {tab === "reskin" ? <ReskinTab token={token} /> : <GenerateTab token={token} />}
        </div>

        <div style={{ marginTop: 16, textAlign: "center", fontSize: 11, color: MUTED }}>
          G2 Brand Library · G2 Approved Templates · Google Slides API · G2 MCP
        </div>
      </div>
    </div>
  );
}
