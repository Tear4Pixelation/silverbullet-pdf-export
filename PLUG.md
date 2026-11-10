---
name: Library/Tear4Pixelation/PDF Export
description: "Export the current page (or selection) as a styled PDF via the browser's print dialog."
tags: meta/library
---
# PDF Export
Adds the command **Export: PDF** and a **PDF** option to `Export: Page Or Selection`.

It renders the page with SilverBullet's own renderer (transclusions and Lua expressions are expanded), applies a print stylesheet, turns `> **note**`-style admonitions into callout boxes and opens the print dialog. Pick **Save as PDF** as the printer.

Code blocks that a plug renders as a widget (for example ` ```excalidraw ` or ` ```canvas `) are rendered by that plug and printed as the widget, not as code. Widgets run read-only: their syscalls go to SilverBullet, but writes and navigation are ignored. The export waits until the widgets stop resizing (at most 10 seconds) before it prints.

```space-lua
-- priority: 5
local pdfExport = {}

pdfExport.css = [==[
@page { margin: 16mm 14mm; }
:root {
  --font-body: Inter, system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  --accent: #464cfc;
  --note: #00a3ff;
  --warning: #ff9100;
  --danger: #e5484d;
  --success: #30a46c;
}
html { font-size: 18px; }
body { font-family: var(--font-body); color: #111; background: #fff; margin: 0; }
h1, h2, h3, h4, h5, h6 { margin: 0 0 0.4rem; break-after: avoid; }
p { white-space: pre-line; }
pre, code { font-family: var(--font-mono); }
pre { background: #f2f2f2; border-radius: 8px; padding: 16px 20px; white-space: pre-wrap; }
:not(pre) > code { background: #f2f2f2; padding: 0.15em 0.35em; border-radius: 4px; }
ul, ol { margin: 0.5rem 0 0.75rem; padding-left: 1.25rem; }
li { margin: 0.35rem 0; }
li:has(> input[type="checkbox"]), li:has(> .sb-task-state), li:has(> .sb-task) { list-style: none; }
table { border-collapse: collapse; margin: 0.75rem 0; }
th, td { text-align: left; vertical-align: top; padding: 0.45rem 1rem; border-bottom: 1px solid rgba(0,0,0,.15); }
th { font-weight: 600; }
hr { border: none; border-top: 1px solid rgba(0,0,0,.15); margin: 1.25rem 0; }
img { max-width: 100%; }
blockquote {
  margin: 1rem 0; padding: 0.5rem 1rem;
  border-left: 4px solid #ccc; background: #f9f9f9; color: #666; font-style: italic;
}
blockquote > :first-child { margin-top: 0; }
blockquote > :last-child { margin-bottom: 0; }

/* Plug-rendered code widgets */
.sb-pdf-widget { display: block; width: 100%; border: 0; margin: 0.75rem 0; break-inside: avoid; }

/* SilverBullet inline syntax */
a { color: var(--accent); }
a.wiki-link { text-decoration: none; border-bottom: 1px solid color-mix(in srgb, var(--accent) 35%, transparent); }
a.sb-hashtag {
  display: inline-block; padding: 0 0.4em; border-radius: 0.6em;
  background: #e8e9ff; color: var(--accent); font-size: 0.9em; text-decoration: none;
}
.sb-highlight { background: #fff3a3; padding: 0 0.1em; border-radius: 2px; }
.sb-attribute {
  display: inline-block; padding: 0 0.35em; border: 1px solid rgba(0,0,0,.15);
  border-radius: 4px; font-size: 0.85em; color: #555;
}
.sb-task-mark { display: none; }
.sb-task-state {
  display: inline-block; padding: 0 0.4em; border: 1px solid currentColor; border-radius: 4px;
  color: #6b6b6b; font-size: 0.75em; font-weight: 600; vertical-align: 0.1em;
}
.sb-named-anchor, .sb-frontmatter { display: none; }

/* Admonitions */
.sb-admonition {
  --color: var(--note);
  margin: 1rem 0; padding: 0.6rem 1rem 0.7rem;
  border-left: 4px solid var(--color); border-radius: 6px;
  background: color-mix(in srgb, var(--color) 9%, white);
  break-inside: avoid;
}
.sb-admonition[data-admonition="warning" i] { --color: var(--warning); }
.sb-admonition[data-admonition="danger" i] { --color: var(--danger); }
.sb-admonition[data-admonition="success" i] { --color: var(--success); }
.sb-admonition-title { display: flex; align-items: center; gap: 0.45rem; font-weight: 600; margin-bottom: 0.35rem; }
.sb-admonition-title::before {
  content: ""; flex: none; width: 0.55em; height: 0.55em; border-radius: 50%; background: var(--color);
}
.sb-admonition-title.untitled { color: var(--color); }
.sb-admonition > :not(.sb-admonition-title):first-of-type { margin-top: 0; }
.sb-admonition > :last-child { margin-bottom: 0; }
.sb-admonition :not(pre) > code, .sb-admonition pre { background: color-mix(in srgb, var(--color) 6%, white); }
]==]

-- Runs inside the print frame: mounts plug widgets, converts admonitions, then prints.
pdfExport.script = [==[
const WIDGET_TOKEN = /^PDFWIDGET(\d+)END$/;
const WIDGET_SHELL = '<!DOCTYPE html><html data-theme="light"><head><meta charset="utf-8">'
  + '<style>html, body { margin: 0; background: transparent; } @media print { button { display: none !important; } }</style>'
  + '</head><body></body></html>';
// Widgets get a read-only view of SilverBullet: no writes, no navigation, no UI prompts.
const BLOCKED_SYSCALLS = /^(space\.(write|delete)|editor\.(replace|set|insert|navigate|open|flash|prompt|confirm|filterBox|save|reload|moveCursor|dispatch))/;
const PRINT_PIXEL_RATIO = 3;
const SETTLE_MS = 1500;
const MAX_WAIT_MS = 10000;

let lastResize = Date.now();

function hostSyscall(name, args) {
  if (name === "system.getMode") return Promise.resolve("ro");
  if (BLOCKED_SYSCALLS.test(name)) return Promise.resolve(null);
  const host = window.parent && window.parent.client;
  if (!host) return Promise.reject(new Error("SilverBullet client not reachable"));
  return host.clientSystem.system.localSyscall(name, args);
}

function mountWidget(placeholder, widget) {
  const frame = document.createElement("iframe");
  frame.className = "sb-pdf-widget";
  frame.style.height = (widget.height || 400) + "px";
  if (widget.url) frame.src = widget.url;
  else frame.srcdoc = WIDGET_SHELL;
  placeholder.replaceWith(frame);

  return new Promise((resolve) => {
    frame.addEventListener("load", () => {
      if (!widget.url) {
        const frameWindow = frame.contentWindow;
        // SilverBullet widget frames only get `syscall`. Plugs (e.g. Excalidraw) check for a
        // `silverbullet` global to detect document-editor frames, so it must stay undefined.
        frameWindow.syscall = (name, ...args) => hostSyscall(name, args);
        // Canvas-based widgets (Excalidraw) size their bitmap by devicePixelRatio; raise it for print resolution.
        const printPixelRatio = Math.max(PRINT_PIXEL_RATIO, window.devicePixelRatio || 1);
        Object.defineProperty(frameWindow, "devicePixelRatio", { get: () => printPixelRatio, configurable: true });
        frameWindow.document.body.innerHTML = widget.html || "";
        if (widget.script) {
          const scriptElement = frameWindow.document.createElement("script");
          scriptElement.textContent = widget.script;
          frameWindow.document.body.appendChild(scriptElement);
        }
      }
      resolve(frame);
    }, { once: true });
  });
}

function fitToContent(frame) {
  try {
    const contentHeight = frame.contentDocument.documentElement.scrollHeight;
    if (contentHeight > 0) frame.style.height = contentHeight + "px";
  } catch (error) {
    // Cross-origin (url widgets): keep the reported height.
  }
}

async function mountWidgets(root) {
  const dataElement = document.getElementById("pdf-widgets");
  const widgets = dataElement ? JSON.parse(dataElement.textContent) : [];
  if (!widgets.length) return;

  const mounting = [];
  for (const block of Array.from(root.querySelectorAll("pre"))) {
    const match = block.textContent.trim().match(WIDGET_TOKEN);
    const widget = match && widgets[Number(match[1]) - 1];
    if (widget) mounting.push(mountWidget(block, widget));
  }
  const frames = await Promise.all(mounting);

  window.addEventListener("message", (event) => {
    if (!event.data || event.data.type !== "setHeight") return;
    const frame = frames.find((candidate) => candidate.contentWindow === event.source);
    if (frame) {
      frame.style.height = event.data.height + "px";
      lastResize = Date.now();
    }
  });

  // Wait until the widgets stop resizing, then size each frame to its content.
  const started = Date.now();
  lastResize = started;
  while (Date.now() - lastResize < SETTLE_MS && Date.now() - started < MAX_WAIT_MS) {
    await new Promise((resolve) => setTimeout(resolve, 250));
  }
  frames.forEach(fitToContent);
}

// The renderer links each task to its source position ("Page@123"); that's noise on paper.
function removeTaskReferences(root) {
  for (const link of Array.from(root.querySelectorAll("li a"))) {
    const item = link.closest("li");
    const isTask = item.querySelector('input[type="checkbox"], .sb-task-state, .sb-task');
    if (isTask && /@\d+$/.test(link.textContent.trim())) link.remove();
  }
}

function convertAdmonitions(root) {
  for (const quote of Array.from(root.querySelectorAll("blockquote"))) {
    const firstParagraph = quote.firstElementChild;
    if (!firstParagraph || firstParagraph.tagName !== "P") continue;

    // Skip leading whitespace text
    let marker = firstParagraph.firstChild;
    while (marker && marker.nodeType === Node.TEXT_NODE && !marker.textContent.trim()) marker = marker.nextSibling;
    if (!marker) continue;

    let admonitionType = null;
    if (marker.nodeName === "STRONG" && /^[\w-]+$/.test(marker.textContent.trim())) {
      admonitionType = marker.textContent.trim().toLowerCase();
      marker.remove();
    } else if (marker.nodeType === Node.TEXT_NODE) {
      const githubMatch = marker.textContent.match(/^\s*\[!([\w-]+)\]/);
      if (!githubMatch) continue;
      admonitionType = githubMatch[1].toLowerCase();
      marker.textContent = marker.textContent.slice(githubMatch[0].length);
    } else {
      continue;
    }

    // The rest of the first line becomes the title.
    const title = document.createElement("div");
    title.className = "sb-admonition-title";
    while (firstParagraph.firstChild) {
      const node = firstParagraph.firstChild;
      if (node.nodeName === "BR") { node.remove(); break; }
      if (node.nodeType === Node.TEXT_NODE && node.textContent.includes("\n")) {
        const breakIndex = node.textContent.indexOf("\n");
        title.append(node.textContent.slice(0, breakIndex));
        node.textContent = node.textContent.slice(breakIndex + 1);
        break;
      }
      title.append(node);
    }
    if (!title.textContent.trim()) {
      title.textContent = admonitionType.charAt(0).toUpperCase() + admonitionType.slice(1);
      title.classList.add("untitled");
    }
    if (!firstParagraph.textContent.trim() && !firstParagraph.querySelector("img")) firstParagraph.remove();

    const box = document.createElement("div");
    box.className = "sb-admonition";
    box.dataset.admonition = admonitionType;
    box.append(title, ...Array.from(quote.childNodes));
    quote.replaceWith(box);
  }
}

window.addEventListener("load", async () => {
  try {
    await mountWidgets(document.body);
  } catch (error) {
    console.error("PDF export: widget rendering failed", error);
  }
  removeTaskReferences(document.body);
  convertAdmonitions(document.body);
  if (document.fonts) await document.fonts.ready;
  window.addEventListener("afterprint", () => setTimeout(() => window.frameElement && window.frameElement.remove(), 100));
  setTimeout(() => window.print(), 50);
});
]==]

local function escapeHtml(text)
  text = string.gsub(text, "&", "&amp;")
  text = string.gsub(text, "<", "&lt;")
  text = string.gsub(text, ">", "&gt;")
  return text
end

local function stripFrontmatter(text)
  local body = string.match(text, "^%-%-%-[ \t]*\n.-\n%-%-%-[ \t]*\n(.*)$")
  return body or text
end

local function nodeText(node)
  if node.text then return node.text end
  local parts = {}
  for _, child in ipairs(node.children or {}) do
    table.insert(parts, nodeText(child))
  end
  return table.concat(parts)
end

local function findChild(node, childType)
  for _, child in ipairs(node.children or {}) do
    if child.type == childType then return child end
  end
  return nil
end

-- Asks the plugs to render fenced code blocks they registered a widget for.
-- Each rendered block's body is swapped for a token the print frame replaces with the widget.
local function renderCodeWidgets(text, pageName)
  local tree = markdown.parseMarkdown(text)
  local widgets = {}

  local function visit(node)
    for _, child in ipairs(node.children or {}) do
      if child.type == "FencedCode" then
        local info = findChild(child, "CodeInfo")
        local codeText = findChild(child, "CodeText")
        local language = info and string.match(nodeText(info), "^%s*([^%s]+)")
        if language and codeText then
          local body = nodeText(codeText)
          local ok, content = pcall(function()
            return codeWidget.render(language, body, pageName)
          end)
          if ok and content and (content.html or content.url) then
            table.insert(widgets, {
              html = content.html,
              script = content.script,
              url = content.url,
              height = content.height
            })
            codeText.children = { { text = "PDFWIDGET" .. #widgets .. "END" } }
          end
        end
      else
        visit(child)
      end
    end
  end

  visit(tree)
  if #widgets == 0 then return text, widgets end
  return markdown.renderParseTree(tree), widgets
end

function pdfExport.run(text, pageName)
  local source, widgets = renderCodeWidgets(stripFrontmatter(text), pageName)
  local html = markdown.markdownToHtml(source, { expand = true })
  local shortName = string.match(pageName, "([^/]+)$") or pageName
  -- "<" inside the JSON would let widget markup close the script tag early.
  local widgetJson = string.gsub(js.stringify(js.tojs(widgets)), "<", "\\u003c")

  local documentHtml = "<!DOCTYPE html><html data-theme=\"light\"><head><meta charset=\"utf-8\">"
    .. "<title>" .. escapeHtml(shortName) .. "</title>"
    .. "<style>" .. pdfExport.css .. "</style>"
    .. "<script type=\"application/json\" id=\"pdf-widgets\">" .. widgetJson .. "</script>"
    .. "<script>" .. pdfExport.script .. "</script>"
    .. "</head><body>" .. html .. "</body></html>"

  local document = js.window.document
  local frame = document.createElement("iframe")
  -- Off screen but full width, so widgets lay out at page width before printing.
  frame.setAttribute("style", "position:fixed;left:-10000px;top:0;width:182mm;height:260mm;border:0")
  frame.setAttribute("srcdoc", documentHtml)
  document.body.appendChild(frame)
end

local function currentTextAndPage()
  local selection = editor.getSelection()
  local text = editor.getText()
  if selection and selection.text and selection.text ~= "" then
    text = selection.text
  end
  return text, editor.getCurrentPage()
end

command.define {
  name = "Export: PDF",
  run = function()
    local text, pageName = currentTextAndPage()
    pdfExport.run(text, pageName)
  end
}

service.define {
  selector = "export",
  match = {
    name = "PDF",
    description = "Styled print view, choose 'Save as PDF' as the printer"
  },
  run = function(data)
    pdfExport.run(data.text, data.pageMeta.name)
  end
}
```
