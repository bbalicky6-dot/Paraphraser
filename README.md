
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Paraphrase & Convert — Real Working App</title>
<style>
  * { box-sizing: border-box; margin: 0; font-family: system-ui, sans-serif; }
  body { background: #eef2f7; padding: 2rem; display: flex; justify-content: center; }
  .app { max-width: 1100px; width: 100%; background: white; border-radius: 1.5rem;
         padding: 2rem; box-shadow: 0 20px 40px rgba(0,20,40,.15); }
  h1 { color: #0b2b4a; margin-bottom: 1rem; }
  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1.5rem; }
  @media (max-width: 700px) { .grid { grid-template-columns: 1fr; } }
  .card { border: 1px solid #d0dfee; border-radius: 1rem; padding: 1.2rem; }
  h2 { color: #1e3f6e; font-size: 1.1rem; margin-bottom: .8rem; }
  textarea, input[type=file] { width: 100%; padding: .8rem; border: 1px solid #c8dcf0;
    border-radius: .6rem; font-family: inherit; margin-bottom: .8rem; }
  textarea { min-height: 140px; resize: vertical; }
  button { background: #1e3f6e; color: white; border: none; padding: .6rem 1.2rem;
    border-radius: .6rem; cursor: pointer; font-weight: 600; margin: .2rem .2rem .2rem 0; }
  button:hover { background: #0f2d4e; }
  button:disabled { opacity: .5; cursor: not-allowed; }
  .out { background: #f2f8ff; border: 1px solid #c8dcf0; border-radius: .6rem;
    padding: .8rem; min-height: 100px; white-space: pre-wrap; margin-top: .5rem;
    max-height: 250px; overflow-y: auto; font-size: .9rem; }
  .status { font-size: .85rem; color: #4d6d8b; margin-top: .5rem; }
</style>
</head>
<body>
<div class="app">
  <h1>✨ Paraphrase & Convert (Real)</h1>

  <div class="grid">
    <!-- PARAPHRASE -->
    <div class="card">
      <h2>📝 Paraphrase / Humanize</h2>
      <textarea id="src" placeholder="Paste text here..."></textarea>
      <div>
        <button id="para">🔄 Paraphrase</button>
        <button id="hum">🧑 Humanize</button>
        <button id="clr">🗑 Clear</button>
      </div>
      <div class="out" id="out">Output appears here...</div>
    </div>

    <!-- CONVERT -->
    <div class="card">
      <h2>📎 File Conversion</h2>
      <input type="file" id="file" accept=".pdf,.docx,.xlsx,.doc,.xls,.txt">
      <div>
        <button id="p2w">📄➡️📝 PDF→Word</button>
        <button id="p2x">📄➡️📊 PDF→Excel</button>
        <button id="w2p">📝➡️📄 Word→PDF</button>
      </div>
      <div class="out" id="cout">No file selected.</div>
      <div class="status" id="status"></div>
    </div>
  </div>
</div>

<!-- REAL LIBRARIES (loaded from CDN) -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/mammoth/1.6.0/mammoth.browser.min.js"></script>

<script>
/* ============ PARAPHRASER (rule-based, context-preserving) ============ */
const SYNONYMS = {
  "important":"significant","significant":"important","big":"large","large":"big",
  "small":"tiny","tiny":"small","fast":"quick","quick":"fast","begin":"start",
  "start":"begin","end":"finish","finish":"end","show":"demonstrate","demonstrate":"show",
  "help":"assist","assist":"help","use":"utilize","utilize":"use","make":"create",
  "create":"make","need":"require","require":"need","get":"obtain","obtain":"get",
  "buy":"purchase","purchase":"buy","sell":"vend","change":"modify","modify":"change",
  "think":"believe","believe":"think","say":"state","state":"say","tell":"inform",
  "inform":"tell","ask":"inquire","inquire":"ask","answer":"respond","respond":"answer",
  "however":"nevertheless","therefore":"thus","moreover":"furthermore","also":"additionally",
  "but":"however","so":"therefore","because":"since","since":"because","many":"numerous",
  "numerous":"many","some":"several","several":"some","all":"every","every":"all",
  "good":"excellent","excellent":"good","bad":"poor","poor":"bad","very":"extremely",
  "extremely":"very","really":"truly","truly":"really","often":"frequently",
  "frequently":"often","always":"constantly","constantly":"always","never":"never"
};

const CONTRACTIONS = {
  "do not":"don't","does not":"doesn't","did not":"didn't","is not":"isn't",
  "are not":"aren't","was not":"wasn't","were not":"weren't","have not":"haven't",
  "has not":"hasn't","had not":"hadn't","cannot":"can't","will not":"won't",
  "would not":"wouldn't","should not":"shouldn't","could not":"couldn't",
  "it is":"it's","that is":"that's","there is":"there's","they are":"they're",
  "we are":"we're","you are":"you're","I am":"I'm","I have":"I've","I will":"I'll",
  "I would":"I'd","he is":"he's","she is":"she's","we will":"we'll","they will":"they'll"
};

function preserveCase(original, replacement) {
  if (original[0] === original[0].toUpperCase() && original[0] !== original[0].toLowerCase())
    return replacement[0].toUpperCase() + replacement.slice(1);
  return replacement;
}

function paraphrase(text, humanize=false) {
  if (!text.trim()) return "Please enter some text.";
  let result = text;

  // Apply synonym replacement
  const keys = Object.keys(SYNONYMS).sort((a,b)=>b.length-a.length);
  for (const key of keys) {
    const re = new RegExp(`\\b${key}\\b`, "gi");
    result = result.replace(re, m => preserveCase(m, SYNONYMS[key]));
  }

  if (humanize) {
    // Apply contractions
    for (const [phrase, c] of Object.entries(CONTRACTIONS)) {
      const re = new RegExp(`\\b${phrase}\\b`, "gi");
      result = result.replace(re, m => preserveCase(m, c));
    }
    // Remove stiff academic connectives
    result = result.replace(/\bFurthermore\b/gi, "Also")
                   .replace(/\bMoreover\b/gi, "Plus")
                   .replace(/\bNevertheless\b/gi, "Still")
                   .replace(/\bConsequently\b/gi, "So");
  }

  // Cleanup
  result = result.replace(/\s+/g," ").trim();
  result = result.replace(/(^\s*[a-z])|([.!?]\s+[a-z])/g, m => m.toUpperCase());
  if (!/[.!?]$/.test(result)) result += ".";
  return result;
}

/* ============ DOM: PARAPHRASE ============ */
const src = document.getElementById("src");
const out = document.getElementById("out");

document.getElementById("para").onclick = () => {
  out.textContent = paraphrase(src.value, false);
};
document.getElementById("hum").onclick = () => {
  out.textContent = paraphrase(src.value, true);
};
document.getElementById("clr").onclick = () => {
  src.value = ""; out.textContent = "Output appears here...";
};

/* ============ FILE CONVERSION (REAL) ============ */
const fileInput = document.getElementById("file");
const cout = document.getElementById("cout");
const status = document.getElementById("status");
let currentFile = null;

fileInput.onchange = e => {
  currentFile = e.target.files[0];
  if (currentFile) {
    cout.textContent = `Selected: ${currentFile.name} (${(currentFile.size/1024).toFixed(1)} KB)`;
  }
};

function setStatus(msg) { status.textContent = msg; }
function setOut(msg) { cout.textContent = msg; }

function downloadBlob(blob, name) {
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = name; a.click();
  URL.revokeObjectURL(url);
}

/* --- PDF → Word (extracts text, writes .docx via HTML blob trick) --- */
document.getElementById("p2w").onclick = async () => {
  if (!currentFile || !currentFile.name.toLowerCase().endsWith(".pdf"))
    return setStatus("❌ Please select a PDF file.");
  setStatus("⏳ Extracting text from PDF...");
  try {
    const buf = await currentFile.arrayBuffer();
    const pdf = await pdfjsLib.getDocument({ data: buf }).promise;
    let fullText = "";
    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();
      fullText += content.items.map(it => it.str).join(" ") + "\n\n";
    }
    // Build a Word-compatible HTML file (.doc)
    const html = `<html xmlns:o='urn:schemas-microsoft-com:office:office'
      xmlns:w='urn:schemas-microsoft-com:office:word' xmlns='http://www.w3.org/TR/REC-html40'>
      <head><meta charset='utf-8'></head><body><pre>${fullText
        .replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;")}</pre></body></html>`;
    const blob = new Blob(["\ufeff", html], { type: "application/msword" });
    downloadBlob(blob, currentFile.name.replace(/\.pdf$/i, ".doc"));
    setStatus("✅ PDF extracted → .doc downloaded.");
  } catch (err) {
    setStatus("❌ Error: " + err.message);
  }
};

/* --- PDF → Excel (extracts text lines into a spreadsheet) --- */
document.getElementById("p2x").onclick = async () => {
  if (!currentFile || !currentFile.name.toLowerCase().endsWith(".pdf"))
    return setStatus("❌ Please select a PDF file.");
  setStatus("⏳ Extracting text for spreadsheet...");
  try {
    const buf = await currentFile.arrayBuffer();
    const pdf = await pdfjsLib.getDocument({ data: buf }).promise;
    const rows = [];
    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();
      const lines = {};
      content.items.forEach(it => {
        const y = Math.round(it.transform[5]);
        if (!lines[y]) lines[y] = [];
        lines[y].push(it.str);
      });
      Object.keys(lines).sort((a,b)=>b-a).forEach(y => rows.push(lines[y]));
    }
    const wb = XLSX.utils.book_new();
    const ws = XLSX.utils.aoa_to_sheet(rows);
    XLSX.utils.book_append_sheet(wb, ws, "PDF Data");
    XLSX.writeFile(wb, currentFile.name.replace(/\.pdf$/i, ".xlsx"));
    setStatus("✅ PDF text → .xlsx downloaded.");
  } catch (err) {
    setStatus("❌ Error: " + err.message);
  }
};

/* --- Word → PDF (uses mammoth to extract text, jsPDF to render) --- */
document.getElementById("w2p").onclick = async () => {
  const name = currentFile?.name.toLowerCase() || "";
  if (!currentFile || !(name.endsWith(".docx") || name.endsWith(".doc")))
    return setStatus("❌ Please select a Word file (.docx/.doc).");
  setStatus("⏳ Reading Word document...");
  try {
    const buf = await currentFile.arrayBuffer();
    const result = await mammoth.extractRawText({ arrayBuffer: buf });
    const text = result.value;
    const { jsPDF } = window.jspdf;
    const doc = new jsPDF();
    const lines = doc.splitTextToSize(text, 180);
    let y = 20;
    lines.forEach(line => {
      if (y > 280) { doc.addPage(); y = 20; }
      doc.text(line, 15, y);
      y += 7;
    });
    doc.save(currentFile.name.replace(/\.(docx?|DOCX?)$/, ".pdf"));
    setStatus("✅ Word → PDF downloaded.");
  } catch (err) {
    setStatus("❌ Error: " + err.message);
  }
};
</script>
</body>
</html>
