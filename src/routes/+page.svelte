<script>
  import TrustBar from "$lib/TrustBar.svelte";
  import ToolHeader from "$lib/ToolHeader.svelte";
  import ToolFooter from "$lib/ToolFooter.svelte";

  let input = $state("");
  let output = $state("");
  let copied = $state(false);
  let stats = $state(null);

  const TRACKING_PARAMS = [
    /* UTM */
    "utm_source", "utm_medium", "utm_campaign", "utm_term", "utm_content", "utm_id", "utm_cid",
    /* Facebook */
    "fbclid", "fb_action_ids", "fb_action_types", "fb_ref", "fb_source",
    /* Google */
    "gclid", "gclsrc", "dclid", "gbraid", "wbraid", "gad_source",
    /* Microsoft */
    "msclkid",
    /* HubSpot */
    "hsa_cam", "hsa_grp", "hsa_mt", "hsa_src", "hsa_ad", "hsa_acc", "hsa_net", "hsa_ver", "hsa_la", "hsa_ol", "hsa_kw",
    /* Mailchimp */
    "mc_cid", "mc_eid",
    /* Adobe / Marketo */
    "s_kwcid", "mkt_tok",
    /* Misc tracking */
    "ref", "referrer", "affiliate", "affiliate_id", "partner_id", "click_id",
    "campaign_id", "ad_id", "adgroup_id", "adset_id",
    "_ga", "_gl", "_hsenc", "_hsmi", "_openstat",
    "igshid", "si", "feature", "app", "src",
    /* Twitter/X */
    "twclid",
    /* TikTok */
    "ttclid",
    /* Pinterest */
    "epik",
    /* Snapchat */
    "sccid", "ScCid",
  ];

  const TRACKING_SET = new Set(TRACKING_PARAMS.map((p) => p.toLowerCase()));

  function clean() {
    if (input.trim() === "") {
      output = "";
      stats = null;
      return;
    }

    const lines = input.split("\n").filter((l) => l.trim());
    const cleaned = [];
    let totalRemoved = 0;
    let totalUrls = 0;

    for (const line of lines) {
      const trimmed = line.trim();
      try {
        const url = new URL(trimmed);
        totalUrls++;
        const params = new URLSearchParams(url.search);
        let removed = 0;

        const keysToDelete = [];
        for (const [key] of params) {
          if (TRACKING_SET.has(key.toLowerCase())) {
            keysToDelete.push(key);
          }
        }

        for (const key of keysToDelete) {
          params.delete(key);
          removed++;
        }

        url.search = params.toString();

        /* Also strip hash fragments that are just tracking */
        if (url.hash && url.hash.length < 3) {
          url.hash = "";
        }

        cleaned.push(url.toString());
        totalRemoved += removed;
      } catch {
        /* Not a valid URL, pass through */
        cleaned.push(trimmed);
      }
    }

    output = cleaned.join("\n");
    stats = { urls: totalUrls, removed: totalRemoved };
  }

  function handleInput() {
    clean();
  }

  function copyOutput() {
    if (!output) return;
    navigator.clipboard.writeText(output).then(() => {
      copied = true;
      setTimeout(() => (copied = false), 1500);
    });
  }

  function clear() {
    input = "";
    output = "";
    stats = null;
    copied = false;
  }

  function loadSample() {
    input = `https://example.com/products/shoes?utm_source=facebook&utm_medium=cpc&utm_campaign=summer_sale&fbclid=IwAR3xYz123abc
https://shop.example.com/item/12345?gclid=CjwKCAjw&utm_source=google&utm_medium=paid&ref=banner_ad&_ga=2.123456.789
https://news.example.com/article/headline?utm_source=newsletter&utm_medium=email&utm_campaign=weekly_digest&mc_cid=abc123&mc_eid=def456`;
    clean();
  }

  let inputLineCount = $derived(
    input.trim() ? input.split("\n").filter((l) => l.trim()).length : 0
  );
</script>

<TrustBar sourceUrl="https://github.com/user/localurl" />

<div class="page-shell">
  <ToolHeader
    title="Link Cleaner"
    description="Strip tracking parameters from URLs — nothing leaves your browser."
  />

  <div class="tool-area">
    <div class="panel">
      <label class="tool-label" for="input">dirty urls</label>
      <textarea
        id="input"
        class="tool-textarea"
        bind:value={input}
        oninput={handleInput}
        placeholder="Paste URLs with tracking params (one per line)..."
        spellcheck="false"
      ></textarea>
    </div>

    <div class="panel">
      <label class="tool-label" for="output">clean urls</label>
      <textarea
        id="output"
        class="tool-textarea"
        class:output-valid={stats && stats.removed > 0}
        value={output}
        readonly
        placeholder="Cleaned URLs appear here..."
        spellcheck="false"
      ></textarea>
    </div>
  </div>

  <div class="controls">
    <button class="btn btn-primary" onclick={clean}>Clean</button>
    <button class="btn" onclick={copyOutput} disabled={!output}>
      {copied ? "✓ Copied" : "Copy"}
    </button>
    <button class="btn" onclick={clear}>Clear</button>
    <button class="btn btn-sample" onclick={loadSample}>Sample</button>

    <div class="spacer"></div>

    {#if stats}
      {#if stats.removed > 0}
        <span class="pill pill-ok">
          <span class="pill-dot"></span>
          {stats.removed} tracker{stats.removed !== 1 ? "s" : ""} stripped
        </span>
      {:else if stats.urls > 0}
        <span class="pill pill-ok">
          <span class="pill-dot"></span>
          already clean
        </span>
      {/if}
    {/if}
  </div>

  <div class="tracker-info">
    <details>
      <summary class="tracker-summary">What gets removed?</summary>
      <div class="tracker-list">
        <span class="tracker-group">
          <strong>Ad platforms:</strong> utm_*, fbclid, gclid, msclkid, twclid, ttclid, dclid, gbraid, wbraid
        </span>
        <span class="tracker-group">
          <strong>Email tools:</strong> mc_cid, mc_eid, _hsenc, _hsmi, mkt_tok
        </span>
        <span class="tracker-group">
          <strong>Analytics:</strong> _ga, _gl, s_kwcid, _openstat, hsa_*
        </span>
        <span class="tracker-group">
          <strong>Social:</strong> igshid, si, epik, sccid, fb_ref, fb_source
        </span>
        <span class="tracker-group">
          <strong>Affiliates:</strong> ref, affiliate_id, partner_id, click_id, campaign_id
        </span>
      </div>
    </details>
  </div>

  <ToolFooter />
</div>

<style>
  .panel {
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .btn-sample {
    font-size: 12px;
    padding: 8px 12px;
    color: var(--text-muted);
  }

  .spacer {
    margin-left: auto;
  }

  .tracker-info {
    margin-top: 16px;
  }

  .tracker-summary {
    font-family: var(--font-mono);
    font-size: 12px;
    color: var(--text-muted);
    cursor: pointer;
    padding: 8px 0;
  }

  .tracker-summary:hover {
    color: var(--accent);
  }

  .tracker-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding: 12px 14px;
    background: var(--bg-input);
    border-radius: var(--radius-sm);
    margin-top: 8px;
  }

  .tracker-group {
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--text-muted);
    line-height: 1.5;
  }

  .tracker-group strong {
    color: var(--text-secondary);
  }

  button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  @media (max-width: 640px) {
    .controls {
      flex-wrap: wrap;
    }
  }
</style>
