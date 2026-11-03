// ==UserScript==
// @name         Instagram Recent Sorter (Firefox Android)
// @namespace    instagram-recent-local
// @version      0.1
// @description  Firefox Android에서 Instagram 해시태그 검색 결과를 최신순으로 정렬합니다.
// @match        https://www.instagram.com/*
// @match        https://instagram.com/*
// @run-at       document-end
// @noframes
// @sandbox      JavaScript
// @grant        GM_info
// @homepageURL  https://github.com/hillofsky/instagram-recent-sorter-firefox-android
// @supportURL   https://github.com/hillofsky/instagram-recent-sorter-firefox-android/issues
// @updateURL    https://raw.githubusercontent.com/hillofsky/instagram-recent-sorter-firefox-android/main/InstagramRecentMobile.meta.js
// @downloadURL  https://raw.githubusercontent.com/hillofsky/instagram-recent-sorter-firefox-android/main/InstagramRecentMobile.user.js
// ==/UserScript==

/* B74 isolated execution indicator: runs BEFORE the original sorter. */
(function(){
 'use strict';
 const ID='ig-recent-b75-launcher', INFO='ig-recent-b75-diagnostic';
 function searchRoute(){return /^\/explore\/(?:tags|search)(?:\/|$)/.test(location.pathname)||/^\/popular\/[^/]+\/?$/.test(location.pathname);}
 function countLinks(){try{return document.querySelectorAll('a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]').length;}catch(_){return 0;}}
 function diagnostic(_reason){
  if(!document.body)return;
  let box=document.getElementById(INFO);
  if(!box){box=document.createElement('div');box.id=INFO;document.body.appendChild(box);}
  box.style.cssText='all:initial!important;position:fixed!important;right:12px!important;bottom:200px!important;z-index:2147483647!important;width:min(280px,85vw)!important;padding:14px!important;background:#262630!important;color:white!important;border-radius:12px!important;font:14px/1.5 system-ui,sans-serif!important;pointer-events:auto!important;';
  box.textContent='최신순 목록을 열지 못했습니다. 페이지를 새로고침해 주세요.';
  let button=document.createElement('button');button.textContent='닫기';button.type='button';button.style.cssText='display:block;padding:8px 16px;margin-top:10px;color:#111;background:#fff;border:0;border-radius:8px';button.addEventListener('click',()=>box.remove());box.appendChild(button);
 }
 window.__IG_RECENT_B78_DIAG__=diagnostic;
 // Only one entry button should be visible. The original count-bearing
 // .float button takes precedence once the sorter has initialized.
 function nativeVisible(){
  const root=document.getElementById('instagram-recent-local-root');
  const native=root && root.shadowRoot && root.shadowRoot.querySelector('button.float');
  if(!native || !native.isConnected || native.hidden)return false;
  try{
   const style=getComputedStyle(native);
   return style.display!=='none' && style.visibility!=='hidden';
  }catch(_){return true;}
 }
 function syncLauncher(){
  const btn=document.getElementById(ID);
  if(!btn)return;
  const visible=searchRoute() && !nativeVisible();
  btn.hidden=!visible;
  btn.style.setProperty('display', visible?'block':'none', 'important');
 }
 window.__IG_RECENT_B78_SYNC_LAUNCHER__=syncLauncher;
 function show(){
  if(!document.body)return;
  let b=document.getElementById(ID);
  if(!b){b=document.createElement('button');b.id=ID;b.type='button';document.body.appendChild(b);
   b.addEventListener('click',()=>{
    if(!searchRoute()){diagnostic('검색 화면이 아닙니다');return;}
    if(typeof window.__IG_RECENT_B78_OPEN__==='function'){
      try{if(window.__IG_RECENT_B78_OPEN__())return;}catch(_){/* diagnostic below */}
    }
    diagnostic('정렬 시작 실패 / 추가 진단 필요');
   });
  }
  b.style.cssText='all:initial!important;position:fixed!important;right:12px!important;bottom:calc(48px + env(safe-area-inset-bottom, 0px))!important;z-index:2147483647!important;display:block!important;visibility:visible!important;opacity:1!important;background:#24242b!important;color:#fff!important;border:2px solid #ffcb55!important;border-radius:99px!important;box-shadow:0 4px 15px #0008!important;padding:12px 16px!important;font:700 14px system-ui,sans-serif!important;pointer-events:auto!important;cursor:pointer!important;';
  b.textContent='최신순 보기';
  syncLauncher();
 }
 show();
 if(document.readyState==='loading')document.addEventListener('DOMContentLoaded',show,{once:true});
 // Bounded SPA and delayed-body checks; no traffic, storage or post data export.
 let n=0;const id=setInterval(()=>{show();if(++n>=20)clearInterval(id);},1200);
})();


/* Instagram Recent Sorter — pure, independently testable utilities.
   No API use, login/cookie access, storage, or network requests. */
(function (root) {
  'use strict';

  const ALPHABET = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_';
  // The epoch is used to DISPLAY an estimated creation date, not to order IDs.
  // Instagram Engineering ID design: time bits (41) + shard (13) + sequence (10).
  const IG_EPOCH_MS = 1314220021721n;
  const MAX_SHORTCODE_LENGTH = 11;

  function shortcodeToId(shortcode) {
    if (typeof shortcode !== 'string' || shortcode.length < 5 ||
        shortcode.length > MAX_SHORTCODE_LENGTH) {
      return null;
    }
    let id = 0n;
    for (const char of shortcode) {
      const index = ALPHABET.indexOf(char);
      if (index < 0) return null;
      id = (id << 6n) + BigInt(index);
    }
    return id > 0n && id <= 0xFFFFFFFFFFFFFFFFn ? id : null;
  }

  function decodePostUrl(rawUrl) {
    if (typeof rawUrl !== 'string') return null;
    let parsed;
    try {
      parsed = new URL(rawUrl, 'https://www.instagram.com');
    } catch (_error) {
      return null;
    }
    if (!['instagram.com', 'www.instagram.com'].includes(parsed.hostname) ||
        parsed.protocol !== 'https:') {
      return null;
    }
    const match = parsed.pathname.match(/^\/(p|reel|tv)\/([A-Za-z0-9_-]{5,11})(?:\/|$)/);
    if (!match) return null;
    const shortcode = match[2];
    const mediaId = shortcodeToId(shortcode);
    if (mediaId === null) return null;
    const ms = Number((mediaId >> 23n) + IG_EPOCH_MS);
    const now = Date.now();
    // Unexpected IDs should never get fabricated display times.
    const estimatedMs = Number.isFinite(ms) && ms >= Date.UTC(2011, 7, 24) &&
      ms <= now + 24 * 60 * 60 * 1000 ? ms : null;
    return {
      shortcode,
      mediaId,
      estimatedMs,
      type: match[1],
      url: `https://www.instagram.com/${match[1]}/${shortcode}/`
    };
  }

  function isSearchRoute(rawUrl) {
    let parsed;
    try { parsed = new URL(rawUrl); }
    catch (_error) { return false; }
    if (parsed.protocol !== 'https:' ||
        !['instagram.com', 'www.instagram.com'].includes(parsed.hostname)) return false;
    return /^\/explore\/(?:tags|search)(?:\/|$)/.test(parsed.pathname) ||
      /^\/popular\/[^/]+\/?$/.test(parsed.pathname);
  }

  function hashtagWithoutHashUrl(rawUrl) {
    if (!isSearchRoute(rawUrl)) return null;
    const parsed = new URL(rawUrl);
    if (!/^\/explore\/search\/keyword\/?$/.test(parsed.pathname)) return null;
    const q = parsed.searchParams.get('q');
    if (!q || !q.startsWith('#') || q.length < 2) return null;
    const result = new URL('https://www.instagram.com/explore/search/keyword/');
    result.searchParams.set('q', q.slice(1));
    return result.href;
  }

  function newerFirst(a, b) {
    if (a.mediaId > b.mediaId) return -1;
    if (a.mediaId < b.mediaId) return 1;
    return 0;
  }

  const api = { shortcodeToId, decodePostUrl, newerFirst, isSearchRoute, hashtagWithoutHashUrl, IG_EPOCH_MS };
  root.IGRecentCore = api;
  if (typeof module !== 'undefined' && module.exports) module.exports = api;
})(globalThis);


/* Instagram Recent Sorter B21
   Reads already-rendered links only. Does not intercept network, access cookies,
   automate scrolling, or alter Instagram's original post grid. */
(() => {
  'use strict';
  // Use a version-specific guard: an earlier, partially initialized sorter
  // must not prevent the mobile launcher from becoming available.
  if (window.top !== window || window.__IG_RECENT_B76_STARTED__) return;
  window.__IG_RECENT_B76_STARTED__ = true;

  const BUILD = '0.1';
  const core = globalThis.IGRecentCore;
  if (!core) return;
  const postMap = new Map();
  // Keep existing image elements mounted when more search results arrive.
  const cardMap = new Map();
  let b38ReusedCardCount = 0;
  let b38ThumbnailRefreshCount = 0;
  let lastUrl = location.href;
  let lastSearchKey = null;
  let scanTimer = null;
  let host = null;
  let shadow = null;
  let panel = null;
  let floating = null;
  let summary = null;
  let grid = null;
  let thumbnailColumns = 3; // Default: 3 columns on mobile and desktop
  let isOpen = false;
  let lastRenderCount = -1;
  let scanCount = 0;
  let lastMatchCount = 0;
  let lastScanSource = "none";
  let lastMissingThumbnailCount = 0;
  let lastScanAt = null;
  let verificationRun = false;
  let lastTotalCandidateCount = 0;
  let lastDocImageCount = 0;
  let lastMainRegionCount = 0;
  let manualOriginalScrollCount = 0;
  let lastThumbnailSourceUpdates = 0;
  let renderedThumbnailFailures = 0;
  let b37FallbackAttempts = 0;
  let b37FallbackRecovered = 0;
  let b37FallbackExhausted = 0;
  let b50MediaAttributeUpdates = 0;
  let b51ManualRetryClicks = 0;
  let b51ManualRetriedCards = 0;
  let b64AssociationCounts = {direct:0, single_tile:0, visual_overlap:0, none:0};
  let b67FallbackAttempts = 0;
  let b67FallbackRecovered = 0;
  let b67MissingZeroRect = 0;
  let b67MissingOffscreen = 0;
  let b67MissingVisible = 0;

  // Only treat an in-page dialog as a search surface when it has clear search
  // affordances. This supports Instagram search overlays on the home URL without
  // accidentally collecting thumbnails from the home feed behind the overlay.
  function getSearchDialog() {
    for (const dialog of document.querySelectorAll('[role="dialog"]')) {
      if (!dialog.isConnected || dialog.getAttribute('aria-hidden') === 'true') continue;
      if (getComputedStyle(dialog).display === 'none' ||
          getComputedStyle(dialog).visibility === 'hidden') continue;
      const label = dialog.getAttribute('aria-label') || '';
      if (/search|검색|hashtag|해시태그|results|검색 결과/i.test(label)) return dialog;
      for (const input of dialog.querySelectorAll('input, [role="searchbox"]')) {
        const attributes = [input.getAttribute('type'), input.getAttribute('placeholder'),
          input.getAttribute('aria-label'), input.getAttribute('role')].join(' ');
        if (/search|검색|hashtag|해시태그|searchbox/i.test(attributes)) return dialog;
      }
    }
    return null;
  }

  // Search may be rendered as a side drawer, not as a role=dialog.
  // Never ascend into <main> or a common ancestor containing <main>:
  // otherwise home-feed posts could be misclassified as search results.
  function getSearchSurface() {
    const dialog = getSearchDialog();
    if (dialog) return { root: dialog, kind: 'search-dialog' };
    const selectors = 'a[href*="/p/"], a[href*="/reel/"], a[href*="/tv/"]';
    for (const input of document.querySelectorAll('input, [role="searchbox"]')) {
      if (host?.contains(input) || input.closest('main, header')) continue;
      const attrs = [input.getAttribute('type'), input.getAttribute('placeholder'),
        input.getAttribute('aria-label'), input.getAttribute('role')].join(' ');
      if (!/search|검색|hashtag|해시태그|searchbox/i.test(attrs)) continue;
      if (getComputedStyle(input).display === 'none' ||
          getComputedStyle(input).visibility === 'hidden') continue;
      let ancestor = input.parentElement;
      for (let depth = 0; ancestor && ancestor !== document.body && depth < 12;
           depth++, ancestor = ancestor.parentElement) {
        if (ancestor.matches('main, header') || ancestor.querySelector('main')) break;
        // Require an actual post thumbnail within the same search pane.
        // A permanent global search input alone is not enough.
        const links = ancestor.querySelectorAll(selectors);
        if ([...links].some(a => a.querySelector('img'))) {
          return { root: ancestor, kind: 'search-sidebar' };
        }
      }
    }
    return null;
  }

  function isSearchLocation() {
    // Test fixture only. The production manifest matches HTTPS Instagram.
    if (location.protocol === 'file:' &&
        document.documentElement.dataset.igRecentFixture === 'true') return true;
    if (location.protocol === 'file:' &&
        document.documentElement.dataset.igRecentFixture === 'overlay') return !!getSearchSurface();
    if (core.isSearchRoute(location.href) || !!getSearchSurface()) return true;
    // B73: some mobile navigation keeps the /explore/ URL while showing
    // a grid. Do not mistake a home/profile page for search results.
    return /^\/explore\/?$/.test(location.pathname) &&
      !!document.querySelector('main a[href*="/p/"], main a[href*="/reel/"]');
  }

  function getSearchKey() {
    const surface = getSearchSurface();
    if (!surface) return location.href;
    const input = [...surface.root.querySelectorAll('input, [role="searchbox"]')]
      .find(node => /search|검색|hashtag|해시태그|searchbox/i.test([
        node.getAttribute('type'), node.getAttribute('placeholder'),
        node.getAttribute('aria-label'), node.getAttribute('role')].join(' ')));
    // Search text is only used transiently to reset mixed results. Never logged,
    // persisted, exported, or sent over the network.
    return `dialog:${location.pathname}:${input ? input.value : ''}`;
  }

  function collectVisibleLinks() {
    // B17: search results may be distributed across multiple main landmarks,
    // and some media links have no img descendants (lazy or video thumbnails).
    // On a dedicated search URL collect from the search document, not FIRST <main>.
    const selectors = 'a[href*="/p/"], a[href*="/reel/"], a[href*="/tv/"]';
    const surface = getSearchSurface();
    const dedicated = core.isSearchRoute(location.href) ||
      /^\/explore\/?$/.test(location.pathname) ||
      (location.protocol === 'file:' &&
       document.documentElement.dataset.igRecentFixture === 'true');
    const mainRoots = [...document.querySelectorAll('main, [role="main"]')];
    lastMainRegionCount = mainRoots.length;
    const groups = surface ? [[surface.kind, [surface.root]]] :
      dedicated ? [['search-document', [document]]] :
      mainRoots.length ? [['multi-main', mainRoots]] :
      [['document-fallback', [document]]];
    const candidates = new Set();
    let visibleImages = 0;
    for (const [kind, roots] of groups) {
      lastScanSource = kind;
      for (const root of roots) {
        visibleImages += [...root.querySelectorAll('img')].filter(img =>
          !host?.contains(img) && !img.closest('header, nav, footer')).length;
        for (const anchor of root.querySelectorAll(selectors)) {
          if (!anchor.isConnected || host?.contains(anchor) ||
              anchor.closest('header, nav, footer')) continue;
          // Post links are validated before collection; do not collect arbitrary URLs.
          if (!core.decodePostUrl(anchor.href)) continue;
          candidates.add(anchor);
        }
      }
    }
    lastTotalCandidateCount = candidates.size;
    lastDocImageCount = visibleImages;
    return candidates;
  }

  // B36: only reuse media URLs already present in the rendered DOM. Instagram
  // often places real thumbnails in srcset/picture/background while src is a
  // temporary placeholder. No API requests, scraping, or post-detail fetch.
  function thumbnailCandidate(url) {
    if (typeof url !== 'string') return null;
    const clean = url.trim().replace(/^['"]|['"]$/g, '');
    return /^(https:\/\/|blob:)/i.test(clean) ? clean : null;
  }

  function bestSrcset(srcset) {
    if (!srcset || typeof srcset !== 'string') return null;
    const parts = srcset.split(',').map(part => part.trim()).filter(Boolean);
    // Prefer last (normally the largest) valid entry; Instagram serves
    // equivalent media in multiple resolutions.
    for (let i = parts.length - 1; i >= 0; --i) {
      const url = parts[i].split(/\s+/)[0];
      if (thumbnailCandidate(url)) return thumbnailCandidate(url);
    }
    return null;
  }

  // B64: Instagram can use a transparent post <a> over an adjacent <img>.
  // Only associate media from the same single-post container or from a
  // *visually overlapping* element on the current search surface. Avoid
  // unrelated feed images, avatars, different tiles, and the local overlay.
  function b64ValidMedia(node) {
    return !!node && node.isConnected && !(host && host.contains(node)) &&
      !node.closest('header,nav,footer');
  }

  function b64IntersectionScore(anchorRect, imageRect) {
    if (anchorRect.width < 32 || anchorRect.height < 32 ||
        imageRect.width < 48 || imageRect.height < 48) return 0;
    const w = Math.max(0, Math.min(anchorRect.right, imageRect.right) -
      Math.max(anchorRect.left, imageRect.left));
    const h = Math.max(0, Math.min(anchorRect.bottom, imageRect.bottom) -
      Math.max(anchorRect.top, imageRect.top));
    const intersection = w * h;
    const anchorArea = anchorRect.width * anchorRect.height;
    const imageArea = imageRect.width * imageRect.height;
    if (!intersection || intersection / anchorArea < 0.30 ||
        intersection / imageArea < 0.30) return 0;
    return intersection / Math.max(anchorArea, imageArea);
  }

  function b64AssociatedMedia(anchor) {
    const hasSource = img => !![img.currentSrc, bestSrcset(img.getAttribute('srcset')),
      img.getAttribute('src'), img.getAttribute('data-src')].some(thumbnailCandidate);
    const direct = [...anchor.querySelectorAll('img')].filter(img => b64ValidMedia(img) && hasSource(img));
    const directVideo = [...anchor.querySelectorAll('video[poster]')].find(video =>
      thumbnailCandidate(video.getAttribute('poster'))) || null;
    if (direct.length || directVideo) {
      return {images: direct, video: directVideo, method: 'direct'};
    }
    // A card may have link and image as siblings, several nested wrappers
    // deep. Stop before crossing a container containing another post link.
    let parent = anchor.parentElement;
    const links = 'a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]';
    for (let depth = 0; parent && depth < 6 && parent !== document.body;
         depth++, parent = parent.parentElement) {
      if (host && host.contains(parent)) break;
      if (parent.querySelectorAll(links).length !== 1) break;
      const images = [...parent.querySelectorAll('img')].filter(img => b64ValidMedia(img) && hasSource(img));
      const video = [...parent.querySelectorAll('video[poster]')].find(el =>
        thumbnailCandidate(el.getAttribute('poster'))) || null;
      const backgrounds = [...parent.querySelectorAll('[style*=\"background\"]')].some(el =>
        /url\(/.test(el.style?.backgroundImage || ''));
      if (images.length || video || backgrounds) return {images, video, method: 'single_tile'};
    }
    // Sometimes links live in an overlay layer separate from a photo grid.
    // Only geometrically matching, rendered media can be attached; never
    // attach a random image just because it belongs to the same document.
    const box = anchor.getBoundingClientRect();
    if (box.width < 32 || box.height < 32) {
      return {images: [], video: null, method: 'none'};
    }
    const surface = getSearchSurface()?.root ||
      (core.isSearchRoute(location.href) ? document : document.querySelector('main'));
    if (!surface) return {images: [], video: null, method: 'none'};
    const choices = [];
    for (const img of surface.querySelectorAll('img')) {
      if (!b64ValidMedia(img) || ![img.currentSrc, bestSrcset(img.getAttribute('srcset')),
        img.getAttribute('src'), img.getAttribute('data-src')].some(thumbnailCandidate)) continue;
      const otherLink = img.closest(links);
      if (otherLink && otherLink !== anchor) continue;
      const score = b64IntersectionScore(box, img.getBoundingClientRect());
      if (score > 0) choices.push({img, score});
    }
    choices.sort((a,b) => b.score - a.score);
    if (choices.length) return {images: [choices[0].img], video: null, method: 'visual_overlap'};
    return {images: [], video: null, method: 'none'};
  }

  function extractThumbnail(anchor) {
    const match = b64AssociatedMedia(anchor);
    const images = match.images;
    const video = match.video;
    const ranked = [];
    for (const img of images) {
      const ready = img.complete && img.naturalWidth > 0;
      const sources = [
        img.currentSrc, bestSrcset(img.getAttribute('srcset')),
        bestSrcset(img.parentElement?.querySelector('source[srcset]')?.getAttribute('srcset')),
        img.getAttribute('src'), img.getAttribute('data-src')
      ];
      for (const [i, value] of sources.entries()) {
        const url = thumbnailCandidate(value);
        if (url) ranked.push({ url, score: (ready ? 100 : 0) + (url.startsWith('https://') ? 20 : 0)
          + (sources.length - i) + Math.min(img.naturalWidth || img.width || 0, 2000) / 1000,
          alt: img.alt || 'Instagram 게시물' });
      }
    }
    if (video) {
      const url = thumbnailCandidate(video.getAttribute('poster'));
      if (url) ranked.push({ url, score: 24, alt: 'Instagram 동영상' });
    }
    // B64: background URLs can also be on a sibling within a single-post tile.
    const backgroundNodes = new Set([anchor, ...anchor.querySelectorAll('[style*="background"]')]);
    if (match.method === 'single_tile') {
      let parent = anchor.parentElement;
      const links = 'a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]';
      for (let depth=0; parent && depth < 6 && parent !== document.body;
           depth++, parent=parent.parentElement) {
        if (parent.querySelectorAll(links).length !== 1) break;
        for (const el of [parent, ...parent.querySelectorAll('[style*="background"]')]) {
          if (backgroundNodes.size < 65) backgroundNodes.add(el);
        }
      }
    }
    for (const item of backgroundNodes) {
      if (!b64ValidMedia(item)) continue;
      const css = item.style?.backgroundImage || '';
      const value = css.match(/url\(["']?(https:\/\/[^"')]+)["']?\)/i);
      const url = value && thumbnailCandidate(value[1]);
      if (url) ranked.push({url, score: 5, alt: 'Instagram 게시물'});
    }
    ranked.sort((x,y) => y.score - x.score);
    const unique = [];
    const seen = new Set();
    for (const item of ranked) {
      if (!seen.has(item.url)) { seen.add(item.url); unique.push(item); }
    }
    return unique.length ? {
      src: unique[0].url, alt: unique[0].alt,
      sources: unique.slice(0, 6).map(item => item.url),
      method: match.method
    } : null;
  }


  // B67: additional strictly local, bounded association for CSS backgrounds,
  // picture sources and SVG <image> on the SAME single-post tile. No API/network
  // requests, post-detail loads, synthetic placeholder URLs or unrelated feed.
  // The extra report fields contain only counts, never URLs or account details.
  function b67ImageSource(node) {
    if (!b64ValidMedia(node)) return null;
    const name = (node.localName || '').toLowerCase();
    let url = null;
    if (name === 'source' && node.closest('picture')) {
      url = bestSrcset(node.getAttribute('srcset')) ||
        bestSrcset(node.getAttribute('data-srcset'));
      if (url) return {url, kind: 'picture'};
    } else if (name === 'image' && node.closest('svg')) {
      url = thumbnailCandidate(node.getAttribute('href') ||
        node.getAttribute('xlink:href'));
      if (url) return {url, kind: 'svg'};
    } else if (name === 'video') {
      url = thumbnailCandidate(node.getAttribute('poster'));
      if (url) return {url, kind: 'video'};
    }
    // Only background-image url(...), not gradients or decorative colors.
    // Computed styles matter when Instagram supplies images via CSS classes.
    let background = node.style?.backgroundImage || '';
    if (!/url\(/i.test(background)) {
      try { background = getComputedStyle(node).backgroundImage || ''; }
      catch (_error) { background = ''; }
    }
    const match = background.match(/url\(\s*["']?([^"')\s]+)["']?\s*\)/i);
    url = match ? thumbnailCandidate(match[1]) : null;
    return url ? {url, kind: 'css'} : null;
  }

  // B68: aggregate-only audit; never export DOM text, URLs or attributes.
  let b68Audit = {};
  // B70: native video previews avoid black GPU/cross-origin canvas captures.
  const b69WatchedVideos = new WeakSet();
  let b69VideoAudit = {};
  // B71: bound decoder allocation to six visible previews, release all others.
  let b71RefreshQueued = false;
  function b71Release(video) {
    if (video._b72Timeout) { clearTimeout(video._b72Timeout); video._b72Timeout = null; }
    if (!video.hasAttribute('src')) return;
    video.pause();
    video.removeAttribute('src');
    video.load();
    video.dataset.frameReady = 'deferred';
  }
  function b71RefreshVideos() {
    if (!grid) return;
    const bounds = grid.getBoundingClientRect();
    const videos = [...grid.querySelectorAll('video.b69-video-frame')];
    const visible = isOpen ? videos.filter(video => {
      if (video.dataset.snapshotReady === 'true' || video.dataset.frameReady === 'error') return false;
      const r = video.getBoundingClientRect();
      return r.bottom > bounds.top && r.top < bounds.bottom &&
        r.right > bounds.left && r.left < bounds.right;
    }).sort((a,b) => {
      const center = (bounds.top + bounds.bottom) / 2;
      return Math.abs(a.getBoundingClientRect().top-center) -
        Math.abs(b.getBoundingClientRect().top-center);
    }).slice(0,6) : [];
    const active = new Set(visible);
    // Release before allocating; the source URL is kept only in local memory.
    for (const video of videos) if (!active.has(video)) b71Release(video);
    for (const video of visible) {
      if (!video.hasAttribute('src') && video.dataset.previewSource) {
        video.dataset.frameReady = 'pending';
        video.src = video.dataset.previewSource;
        video.load();
        video._b72Timeout = setTimeout(() => {
          b71Release(video); video.dataset.frameReady = 'error'; b71ScheduleVideos();
        }, 12000);
      }
    }
  }
  function b71ScheduleVideos() {
    if (b71RefreshQueued) return;
    b71RefreshQueued = true;
    requestAnimationFrame(() => { b71RefreshQueued = false; b71RefreshVideos(); });
  }

  function b69VideoFrame(anchor) {
    const count = key => { b69VideoAudit[key] = (b69VideoAudit[key] || 0) + 1; };
    for (const original of anchor.querySelectorAll('video')) {
      if (!b69WatchedVideos.has(original)) {
        b69WatchedVideos.add(original);
        original.addEventListener('loadeddata', scheduleScan, {once:true});
      }
      const source = original.currentSrc || original.getAttribute('src');
      if (!source) { count('no_video_source'); continue; }
      let url;
      try { url = new URL(source, location.href); } catch (_) { continue; }
      if (!['https:', 'blob:'].includes(url.protocol)) continue;
      const preview = document.createElement('video');
      preview.className = 'thumb b69-video-frame';
      preview.muted = true;
      preview.defaultMuted = true;
      preview.playsInline = true;
      preview.preload = 'none';
      preview.setAttribute('aria-label', 'Instagram 영상 미리보기');
      if (original.videoWidth && original.videoHeight) preview.style.setProperty('--video-ratio', String(original.videoWidth / original.videoHeight));
      preview.dataset.frameReady = 'deferred';
      preview.addEventListener('loadedmetadata', () => {
        preview.style.setProperty('--video-ratio', String(preview.videoWidth / preview.videoHeight || 1));
        // Seek locally in the preview only. Do not play or seek the original.
        try {
          preview.currentTime = Number.isFinite(preview.duration) && preview.duration > 0 ?
            Math.min(0.5, preview.duration / 2) : 0.1;
        } catch (_) { preview.dataset.frameReady = 'pending'; }
      });
      preview.addEventListener('seeked', () => {
        if (preview.readyState < 2 || !preview.videoWidth || preview.dataset.snapshotReady === 'true') return;
        // B72: freeze the decoded preview, then hand the decoder to the next card.
        const canvas = document.createElement('canvas');
        const scale = Math.min(1, 480 / Math.max(preview.videoWidth, preview.videoHeight));
        canvas.width = Math.max(1, Math.round(preview.videoWidth * scale));
        canvas.height = Math.max(1, Math.round(preview.videoHeight * scale));
        try {
          const context = canvas.getContext('2d');
          if (!context) throw new Error('NO_CANVAS');
          context.drawImage(preview, 0, 0, canvas.width, canvas.height);
          canvas.className = 'thumb b72-snapshot';
          canvas.setAttribute('role', 'img');
          canvas.setAttribute('aria-label', 'Instagram 영상 정지 미리보기');
          // Keep canvas display local; do not export or read cross-origin pixels.
          if (!preview.parentElement) throw new Error('NOT_MOUNTED');
          preview.parentElement.appendChild(canvas);
          preview.dataset.snapshotReady = 'true';
          preview.hidden = true;
          b71Release(preview);
          preview.dataset.frameReady = 'true';
        } catch (_) {
          b71Release(preview); preview.dataset.frameReady = 'error';
        }
        b71ScheduleVideos();
      });
      preview.addEventListener('error', () => {
        // Ignore cancellation events caused by releasing a completed decoder.
        if (!preview.hasAttribute('src')) return;
        b71Release(preview); preview.dataset.frameReady = 'error'; b71ScheduleVideos();
      });
      preview.dataset.previewSource = url.href;
      count('native_preview_created');
      return preview;
    }
    return null;
  }

  function b68AuditMissing(anchor) {
    const count = key => { b68Audit[key] = (b68Audit[key] || 0) + 1; };
    count('missing');
    if (anchor.querySelector('img')) count('has_img');
    if (anchor.querySelector('picture,source')) count('has_picture_source');
    if (anchor.querySelector('video')) count('has_video');
    if (anchor.querySelector('svg,image')) count('has_svg');
    if (!anchor.children.length) count('empty_anchor');
    const parent = anchor.parentElement;
    if (parent) {
      if (parent.querySelector('img')) count('parent_has_img');
      const links = parent.querySelectorAll('a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]').length;
      count(links > 1 ? 'parent_multiple_post_links' : 'parent_single_post_link');
    }
  }

  function b67RecoverThumbnail(anchor) {
    b67FallbackAttempts += 1;
    const rect = anchor.getBoundingClientRect();
    const validAnchor = rect.width >= 12 && rect.height >= 12;
    const selectors = 'a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]';
    const own = b67ImageSource(anchor);
    if (own && rect.width >= 40 && rect.height >= 40) {
      b67FallbackRecovered += 1;
      return {src:own.url, alt:'Instagram 게시물', sources:[own.url],
        method:'b67_' + own.kind};
    }
    let node = anchor.parentElement;
    let best = null;
    for (let depth = 0; node && depth <= 5 && node !== document.body;
         depth++, node = node.parentElement) {
      if (node.matches('header,nav,footer,main,[role="main"]')) break;
      // Do not cross into a gallery containing multiple different posts.
      if (node.querySelectorAll(selectors).length !== 1) break;
      const descendants = node.querySelectorAll('*');
      if (descendants.length > 120) break;
      const candidates = [];
      for (const item of [node, ...descendants]) {
        if (!b64ValidMedia(item)) continue;
        const media = b67ImageSource(item);
        if (!media) continue;
        const geometryNode = (item.localName || '').toLowerCase() === 'source' ?
          (item.closest('picture')?.querySelector('img') || item.closest('picture') || item) : item;
        const imageRect = geometryNode.getBoundingClientRect();
        if (imageRect.width < 40 || imageRect.height < 40) continue;
        const overlap = validAnchor ? b64IntersectionScore(rect, imageRect) : 0;
        if (overlap > 0) candidates.push({...media, score: overlap + 1, depth});
        else if (depth <= 2 &&
                 imageRect.width <= 800 && imageRect.height <= 1000) {
          // May be a non-overlapping sibling, but only if it is the SOLE
          // candidate of a shallow, single-link tile (checked below).
          candidates.push({...media, score: 0, depth});
        }
      }
      const distinct = [...new Map(candidates.map(c => [c.url, c])).values()];
      const overlapping = distinct.filter(c => c.score > 0);
      const chosen = overlapping.sort((a,b) => b.score-a.score)[0] ||
        (distinct.length === 1 && distinct[0].depth <= 2 ? distinct[0] : null);
      if (chosen && (!best || chosen.score > best.score)) best = chosen;
      if (best && best.score > 0) break;
    }
    if (!best) {
      const offscreen = validAnchor && (rect.bottom < 0 || rect.top > window.innerHeight ||
        rect.right < 0 || rect.left > window.innerWidth);
      if (!validAnchor) b67MissingZeroRect += 1;
      else if (offscreen) b67MissingOffscreen += 1;
      else b67MissingVisible += 1;
      return null;
    }
    b67FallbackRecovered += 1;
    return {src:best.url, alt:'Instagram 게시물', sources:[best.url],
      method:'b67_' + best.kind};
  }

  function getThumbnailAlternatives(post, excluded = null) {
    // After the first four fail, permit newly observed lazy-load sources.
    const unique = [...new Set([post.src, ...(post.sources || [])]
      .filter(url => thumbnailCandidate(url)))];
    return unique.filter(url => !(excluded && excluded.has(url))).slice(0, 4);
  }

  function createEl(tag, cls, text) {
    const el = document.createElement(tag);
    if (cls) el.className = cls;
    if (text !== undefined) el.textContent = text;
    return el;
  }

  function formatEstimatedTime(ms) {
    if (ms === null) return '날짜 확인 불가';
    try {
      return new Intl.DateTimeFormat('ko-KR', {
        year: 'numeric', month: '2-digit', day: '2-digit',
        hour: '2-digit', minute: '2-digit', hour12: false
      }).format(new Date(ms));
    } catch (_error) {
      return '날짜 확인 불가';
    }
  }

  function buildUI() {
    if (host || !document.body) return;
    host = createEl('div');
    host.id = 'instagram-recent-local-root';
    host.dataset.igRecentBuild = BUILD;
    // B41: explicit popup-triggered, aggregate-only live diagnostic bridge.
    // No report is created or transmitted until the user clicks the popup.
    host.addEventListener('ig-recent-b41-proof-request', event => {
      if (!event.detail || !isSearchLocation()) return;
      if (!isOpen) togglePanel(true);
      verifyAndShow();
      event.detail.report = getSafeDiagnostics();
    });
    shadow = host.attachShadow({ mode: 'open' });
    const style = createEl('style');
    style.textContent = `
      /* B73: lift the Shadow DOM root above mobile Instagram's own
         stacking contexts; pass through touches except for our controls. */
      :host{all:initial!important;display:block!important;position:fixed!important;
        inset:0!important;z-index:2147483647!important;visibility:visible!important;
        pointer-events:none!important}
      *{box-sizing:border-box}
      .float{position:fixed;z-index:2147483644;right:18px;bottom:20px;border:0;
        pointer-events:auto;
        padding:12px 18px;border-radius:999px;background:#2c2c33;color:#fff;
        font:600 14px system-ui,sans-serif;box-shadow:0 5px 20px #0006;
        cursor:pointer}
      .float:hover{background:#454550}
      .backdrop{display:none;position:fixed;z-index:2147483645;inset:0;
        pointer-events:auto;background:#000a;align-items:center;justify-content:center;padding:16px;
        font:14px system-ui,sans-serif;color:#1c1c1c}
      .backdrop.open{display:flex}
      .panel{display:flex;flex-direction:column;width:min(1320px,98vw);height:min(94vh,1080px);
        background:white;border-radius:15px;overflow:hidden;box-shadow:0 10px 48px #0009}
      .header{display:flex;align-items:center;gap:12px;flex-wrap:wrap;padding:13px 16px;
        border-bottom:1px solid #ddd}
      .title{font-weight:750;font-size:17px;flex:1;min-width:140px}
      .btn{border:1px solid #ddd;border-radius:8px;background:#fff;color:#222;
        cursor:pointer;padding:7px 11px;font:600 12px system-ui,sans-serif}
      .btn:hover{background:#eee}.btn:focus-visible,.float:focus-visible{outline:3px solid #4c8af9}
      .hint{padding:10px 16px;background:#f7f7f7;color:#555;font:12px/1.5 system-ui,sans-serif}
      [hidden]{display:none!important}
      /* B20: one independently scrolling grid with natural-ratio media.
         Content rows never shrink as more posts are collected. */
      .grid{overflow-y:auto;overflow-x:hidden;min-height:0;flex:1 1 0;display:grid;
        grid-template-columns:repeat(var(--ig-recent-cols,3),minmax(0,1fr));
        grid-auto-rows:max-content;gap:12px;align-content:start;align-items:start;
        padding:12px;overscroll-behavior:contain;touch-action:pan-y;scrollbar-gutter:stable}
      .card{min-width:0;min-height:0;align-self:start;display:block;
        color:inherit;text-decoration:none;border:1px solid #444;
        border-radius:10px;overflow:hidden;background:#161616}
      .card:hover{outline:2px solid #888}
      .media{position:relative;display:block;width:100%;min-height:0;
        background:#141414;overflow:hidden}
      .thumb{display:block;width:100%;height:auto;max-height:75vh;max-width:100%;
        object-fit:contain;background:#141414}
      .media:has(video[data-frame-ready="deferred"])::before,.media:has(video[data-frame-ready="pending"])::before{content:"영상 미리보기 준비 중";position:absolute;inset:0;display:grid;place-items:center;color:#ddd;background:#343434;z-index:1;font:12px system-ui;text-align:center;padding:12px;pointer-events:none}
      .media:has(video[data-frame-ready="error"])::before{content:"영상 미리보기를 표시할 수 없습니다";position:absolute;inset:0;display:grid;place-items:center;color:#ddd;background:#343434;z-index:1;font:12px system-ui;pointer-events:none}
      video.b69-video-frame[hidden]{display:none}
      .b69-video-frame{aspect-ratio:var(--video-ratio, 1/1)}
      .empty-thumb{display:grid;width:100%;aspect-ratio:1/1;min-height:0;
        place-items:center;background:#343434;color:#fff}
      .date{position:absolute;left:8px;bottom:8px;z-index:2;
        padding:6px 9px;border-radius:6px;background:rgba(0,0,0,.76);
        color:#fff;font:600 13px/1.3 system-ui,sans-serif;
        text-shadow:0 1px 2px #000;max-width:calc(100% - 16px);
        overflow-wrap:anywhere;pointer-events:none}
      .size-picker{border:1px solid #ddd;border-radius:8px;background:#fff;
        color:#222;padding:7px;font:600 12px system-ui,sans-serif;max-width:130px}
      .empty{grid-column:1/-1;text-align:center;margin:30px 10px;color:#555;line-height:1.8}
      @media(max-width:600px){
        .backdrop{padding:0}
        .panel{width:100vw;height:100vh;height:100dvh;max-height:100dvh;border-radius:0}
        .header{gap:6px;max-height:34vh;overflow-y:auto;flex-shrink:0}
        .grid{gap:8px;padding:8px;grid-template-columns:repeat(var(--ig-recent-cols,3),minmax(0,1fr))}
        .date{font-size:12px;padding:5px 7px}
        .title{font-size:15px;min-width:100px}
        .btn{padding:9px 11px;min-height:38px}
        .float{right:10px;bottom:calc(48px + env(safe-area-inset-bottom, 0px));min-height:44px}
      }
    `;
    shadow.appendChild(style);
    floating = createEl('button', 'float', '최신순 보기');
    floating.type = 'button';
    floating.hidden = true;
    floating.addEventListener('click', () => togglePanel(true));
    shadow.appendChild(floating);

    const backdrop = createEl('div', 'backdrop');
    backdrop.setAttribute('role', 'dialog');
    backdrop.setAttribute('aria-label', 'Instagram 최신순 게시물');
    backdrop.setAttribute('aria-modal', 'true');
    const container = createEl('section', 'panel');
    const header = createEl('div', 'header');
    header.appendChild(createEl('span', 'title', '최신순 게시물'));
    const sizePicker = createEl('select', 'size-picker');
    sizePicker.setAttribute('aria-label', '썸네일 열 수');
    for (const [value, name] of [['1', '매우 크게 · 1열'], ['2', '크게 · 2열'], ['3', '보통 · 3열'], ['4', '작게 · 4열'], ['5', '더 작게 · 5열'], ['6', '촘촘하게 · 6열']]) {
      const option = createEl('option', '', name);
      option.value = value;
      sizePicker.appendChild(option);
    }
    sizePicker.value = String(thumbnailColumns);
    sizePicker.addEventListener('change', () => {
      thumbnailColumns = [1, 2, 3, 4, 5, 6].includes(Number(sizePicker.value)) ? Number(sizePicker.value) : 3;
      if (grid) grid.style.setProperty('--ig-recent-cols', String(thumbnailColumns));
      render(); b71ScheduleVideos();
    });
    const refresh = createEl('button', 'btn', '다시 스캔');
    refresh.type = 'button';
    refresh.addEventListener('click', () => { scan(true); render(); });
    const more = createEl('button', 'btn', '게시물 더 불러오기');
    more.type = 'button';
    more.addEventListener('click', () => {
      const moved = scrollOriginalOnce();
      if (summary) summary.textContent = moved ?
        '새 게시물을 확인하고 있습니다…' :
        '더 불러올 게시물이 없다면 Instagram 검색 화면을 직접 내려보세요.';
      if (moved) {
        manualOriginalScrollCount += 1;
        setTimeout(() => { scan(true); render(); }, 1300);
        setTimeout(() => { scan(true); render(); }, 3200);
      }
    });
    const clear = createEl('button', 'btn', '목록 초기화');
    clear.type = 'button';
    clear.addEventListener('click', () => {
      verificationRun = false;
      postMap.clear(); lastRenderCount = -1; scan(true); render();
    });
    const close = createEl('button', 'btn', '닫기 ✕');
    close.type = 'button';
    close.addEventListener('click', () => togglePanel(false));
    header.append(sizePicker, refresh, more, clear, close);
    summary = createEl('div', 'hint');
    const guidance = createEl('div', 'hint');
    guidance.id = 'ig-recent-guidance';
    guidance.hidden = true;
    guidance.appendChild(createEl('span', '', '게시물이 없다면 # 검색 오류일 수 있습니다. '));
    const retry = createEl('a', 'btn', '# 없이 키워드 검색 열기');
    retry.id = 'ig-recent-hash-alternative';
    retry.target = '_blank';
    retry.rel = 'noopener noreferrer';
    guidance.appendChild(retry);
    grid = createEl('div', 'grid');
    grid.addEventListener('scroll', b71ScheduleVideos, {passive:true});
    window.addEventListener('resize', b71ScheduleVideos, {passive:true});
    grid.style.setProperty('--ig-recent-cols', String(thumbnailColumns));
    container.append(header, summary, guidance, grid);
    backdrop.append(container);
    backdrop.addEventListener('click', event => {
      if (event.target === backdrop) togglePanel(false);
    });
    shadow.append(backdrop);
    panel = backdrop;
    document.body.appendChild(host);
    document.addEventListener('keydown', event => {
      if (event.key === 'Escape' && isOpen) togglePanel(false);
    });
  }

  function scrollOriginalOnce() {
    // Explicit user action only; never run continuously or load post URLs.
    const surface = getSearchSurface();
    const roots = surface ? [surface.root] :
      [...document.querySelectorAll('main, [role="main"]')];
    const firstPost = roots.flatMap(root =>
      [...root.querySelectorAll('a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]')])
      .find(a => !host?.contains(a));
    const findScrollable = node => {
      for (let el = node; el && el !== document.body; el = el.parentElement) {
        const style = getComputedStyle(el);
        if (el.scrollHeight > el.clientHeight + 80 &&
            /auto|scroll/.test(style.overflowY)) return el;
      }
      return null;
    };
    const container = firstPost ? findScrollable(firstPost) : null;
    const target = container || document.scrollingElement;
    if (!target || target.scrollHeight <= target.clientHeight + 60) return false;
    const before = target.scrollTop;
    const amount = Math.max(400, Math.floor(target.clientHeight * 0.85));
    target.scrollBy({ top: amount, behavior: 'auto' });
    return target.scrollTop > before;
  }

  function verifyAndShow() {
    if (!isOpen || !grid) return;
    scan(true);
    render();
    verificationRun = true;
  }

  // Verify only the cards rendered in our OWN shadow DOM. Neither media IDs
  // nor permalinks are included in exported diagnostics.
  function getSortVerification() {
    const cards = grid ? [...grid.querySelectorAll('a.card')] : [];
    const ids = cards.map(card => core.decodePostUrl(card.href)?.mediaId ?? null);
    const validIds = ids.every(id => typeof id === 'bigint');
    const orderDescending = validIds && ids.every((id, i) => i === 0 || ids[i - 1] >= id);
    const collectedMatches = cards.length === postMap.size &&
      cards.every(card => !!core.decodePostUrl(card.href));
    const renderedImages = cards.filter(card => !!card.querySelector('img.thumb,video.b69-video-frame')).length;
    const loadedImages = cards.filter(card => {
      const img = card.querySelector('img.thumb');
      return !!card.querySelector('canvas.b72-snapshot') || (img && img.complete && img.naturalWidth > 0);
    }).length;
    const pendingImages = cards.filter(card => {
      const img = card.querySelector('img.thumb');
      const preview = card.querySelector('video.b69-video-frame');
      return (img && !img.complete) || (preview && ['pending', 'deferred'].includes(preview.dataset.frameReady));
    }).length;
    let status;
    if (!isSearchLocation()) status = 'NOT_SEARCH_SCREEN';
    else if (!cards.length) status = 'NO_POSTS_FOUND';
    else if (!validIds || !orderDescending || !collectedMatches)
      status = 'FAIL';
    else if (renderedImages !== cards.length) status = 'PASS_WITH_PLACEHOLDERS';
    else if (cards.length === 1) status = 'ONE_POST_ONLY';
    else status = 'PASS';
    return {
      status,
      rendered_card_count: cards.length,
      rendered_image_count: renderedImages,
      loaded_image_count: loadedImages,
      pending_image_count: pendingImages,
      order_descending: !!orderDescending,
      collection_matches: !!collectedMatches
    };
  }

  // B39: image state is independent of the chronological sort verdict.
  // Do not count a pending lazy-loaded image as a successful thumbnail.
  function classifyImageHealth(total, loaded, pending, placeholders, broken) {
    if (!total) return 'NO_POSTS';
    if (broken) return 'BROKEN_IMAGES';
    if (placeholders) return 'PLACEHOLDERS';
    if (pending) return 'STILL_LOADING';
    return loaded === total ? 'ALL_LOADED' : 'NOT_LOADED';
  }

  function getImageHealth() {
    const cards = grid ? [...grid.querySelectorAll('a.card')] : [];
    let loaded = 0;
    let pending = 0;
    let broken = 0;
    let placeholders = 0;
    for (const card of cards) {
      const img = card.querySelector('img.thumb');
      if (!img) {
        const preview = card.querySelector('video.b69-video-frame');
        if (!preview) placeholders += 1;
        else if (preview.dataset.frameReady === 'error') broken += 1;
        else if (preview.dataset.snapshotReady === 'true' && card.querySelector('canvas.b72-snapshot')) loaded += 1;
        else pending += 1;
        continue;
      }
      if (!img.complete) pending += 1;
      else if (img.naturalWidth > 0) loaded += 1;
      else broken += 1;
    }
    return {
      thumbnail_images_loaded: loaded,
      thumbnail_images_pending: pending,
      thumbnail_images_broken: broken,
      thumbnail_placeholders: placeholders,
      thumbnail_health_status: classifyImageHealth(cards.length, loaded,
        pending, placeholders, broken),
      thumbnail_loaded_percent: cards.length ? Math.round(100 * loaded / cards.length) : 0
    };
  }

  // This report never contains post links, handles, captions, cookies, or access tokens.
  function getSafeDiagnostics() {
    const timestamps = Array.from(postMap.values())
      .map(p => p.estimatedMs).filter(v => Number.isFinite(v));
    const verification = getSortVerification();
    return {
      build: BUILD,
      verification_check_ran: verificationRun,
      verification_status: verificationRun ? verification.status : 'NOT_RUN',
      rendered_card_count: verification.rendered_card_count,
      rendered_image_count: verification.rendered_image_count,
      thumbnail_columns: thumbnailColumns,
      thumbnail_layout: 'natural_ratio_contain_overlay_scroll',
      b36_thumbnail_strategy: 'visible_dom_srcset_picture_poster_background',
      b36_thumbnail_source_updates: lastThumbnailSourceUpdates,
      b36_rendered_thumbnail_errors: renderedThumbnailFailures,
      b37_fallback_attempts: b37FallbackAttempts,
      b37_fallback_recovered: b37FallbackRecovered,
      b37_fallback_exhausted: b37FallbackExhausted,
      b50_media_attribute_updates: b50MediaAttributeUpdates,
      b64_association_direct: b64AssociationCounts.direct,
      b64_association_single_tile: b64AssociationCounts.single_tile,
      b64_association_visual_overlap: b64AssociationCounts.visual_overlap,
      b64_association_none: b64AssociationCounts.none,
      b67_fallback_attempts: b67FallbackAttempts,
      b67_fallback_recovered: b67FallbackRecovered,
      b67_missing_zero_rect: b67MissingZeroRect,
      b67_missing_offscreen: b67MissingOffscreen,
      b67_missing_visible: b67MissingVisible,
      b68_patch: 'eager_dom_audit',
      b69_patch: 'decoded_video_frame',
      b70_patch: 'native_video_preview_6_columns',
      b71_patch: 'bounded_visible_video_decoders',
      b72_patch: 'queued_static_video_previews',
      b74_patch: 'independent_early_launcher_website_injection_probe',
      b75_patch: 'single_visible_launcher_and_compact_date_overlay',
      b73_patch: 'android_dom_sandbox_launcher_recovery',
      b72_snapshot_count: grid ? grid.querySelectorAll('canvas.b72-snapshot').length : 0,
      b71_decoder_limit: 6,
      b71_active_video_sources: grid ? grid.querySelectorAll('video.b69-video-frame[src]').length : 0,
      b71_deferred_video_count: grid ? grid.querySelectorAll('video.b69-video-frame[data-frame-ready="deferred"]').length : 0,
      b70_visual_content_verified: false,
      b69_video_audit: {...b69VideoAudit},
      b69_rendered_video_frames: grid ? grid.querySelectorAll('video.b69-video-frame').length : 0,
      b68_missing_dom: {...b68Audit},
      b51_manual_retry_clicks: b51ManualRetryClicks,
      b51_manual_retried_cards: b51ManualRetriedCards,
      b38_reused_cards: b38ReusedCardCount,
      b38_thumbnail_refreshes: b38ThumbnailRefreshCount,
      ...getImageHealth(),
      diagnostic_captured_at_utc: new Date().toISOString(),
      diagnostic_origin: 'browser-user-click',
      diagnostic_scope: 'visible-page-DOM-only',
      thumbnail_date_overlay_count: grid ? grid.querySelectorAll('.media .date').length : 0,
      thumbnail_fit_contain_count: grid ? [...grid.querySelectorAll('img.thumb')]
        .filter(img => getComputedStyle(img).objectFit === 'contain').length : 0,
      grid_scrollable: !!grid && grid.scrollHeight > grid.clientHeight + 2,
      order_descending: verification.order_descending,
      collection_matches: verification.collection_matches,
      route_type: location.protocol === 'file:' ? 'offline-fixture' :
        getSearchDialog() ? 'overlay-search' :
        getSearchSurface()?.kind === 'search-sidebar' ? 'sidebar-search' :
        location.pathname.startsWith('/explore/tags/') ? 'hashtag' :
        location.pathname.startsWith('/explore/search') ? 'search' :
        location.pathname.startsWith('/popular/') ? 'popular' : 'other',
      active_search_screen: isSearchLocation(),
      search_dialog_detected: !!getSearchDialog(),
      search_sidebar_detected: getSearchSurface()?.kind === 'search-sidebar',
      hash_query_detected: !!core.hashtagWithoutHashUrl(location.href),
      collected_post_count: postMap.size,
      last_scan_anchor_count: lastMatchCount,
      last_total_candidate_count: lastTotalCandidateCount,
      last_document_image_count: lastDocImageCount,
      last_main_region_count: lastMainRegionCount,
      original_scroll_click_count: manualOriginalScrollCount,
      scan_count: scanCount,
      last_scan_source: lastScanSource,
      last_missing_thumbnail_count: lastMissingThumbnailCount,
      has_visible_result_images: lastMatchCount > 0,
      estimated_date_count: timestamps.length,
      estimated_newest_utc: timestamps.length ? new Date(Math.max(...timestamps)).toISOString() : null,
      estimated_oldest_utc: timestamps.length ? new Date(Math.min(...timestamps)).toISOString() : null,
      observed_last_scan_utc: lastScanAt
    };
  }

  function updateFloating() {
    if (!floating) return;
    const active = isSearchLocation() || postMap.size > 0;
    floating.hidden = !active;
    const label = `최신순 보기 (${postMap.size})`;
    if (floating.textContent !== label) floating.textContent = label;
    if (window.__IG_RECENT_B78_SYNC_LAUNCHER__) window.__IG_RECENT_B78_SYNC_LAUNCHER__();
  }

  function scan(force = false) {
    if (!document.body) return;
    const searchKey = getSearchKey();
    if (location.href !== lastUrl || searchKey !== lastSearchKey) {
      verificationRun = false;
      postMap.clear();
      // Cards are cleared by render(); do not destroy mounted images prematurely.
      lastRenderCount = -1;
      lastUrl = location.href;
      lastSearchKey = searchKey;
      if (isOpen) togglePanel(false);
    }
    // Keep the sorter anchored to search/tag pages. Do not rewrite home/profile feeds.
    if (!isSearchLocation()) {
      postMap.clear();
      // Cards are cleared by render(); do not destroy mounted images prematurely.
      lastRenderCount = -1;
      if (floating) floating.hidden = true;
      if (window.__IG_RECENT_B78_SYNC_LAUNCHER__) window.__IG_RECENT_B78_SYNC_LAUNCHER__();
      return;
    }
    buildUI();
    // B73: expose the launcher before thumbnail extraction or layout probing.
    // Android-specific DOM variations must not suppress the entry button.
    updateFloating();
    scanCount += 1;
    lastScanAt = new Date().toISOString();
    const anchors = collectVisibleLinks();
    lastMatchCount = anchors.size;
    lastMissingThumbnailCount = 0;
    b64AssociationCounts = {direct:0, single_tile:0, visual_overlap:0, none:0};
    b68Audit = {}; b69VideoAudit = {};
    b67FallbackAttempts = 0; b67FallbackRecovered = 0;
    b67MissingZeroRect = 0; b67MissingOffscreen = 0; b67MissingVisible = 0;
    let thumbnailsChanged = false;
    for (const anchor of anchors) {
      const data = core.decodePostUrl(anchor.href);
      if (!data) continue;
      const thumb = extractThumbnail(anchor) || b67RecoverThumbnail(anchor);
      const association = thumb?.method || 'none';
      b64AssociationCounts[association] = (b64AssociationCounts[association] || 0) + 1;
      if (!thumb) { lastMissingThumbnailCount += 1; b68AuditMissing(anchor); }
      const old = postMap.get(data.shortcode);
      const videoFrame = !thumb && !old?.videoFrame ? b69VideoFrame(anchor) : null;
      if (old && videoFrame) { old.videoFrame = videoFrame; thumbnailsChanged = true; }
      if (!old) postMap.set(data.shortcode, { ...data,
        src: thumb?.src || null, videoFrame, alt: thumb?.alt || 'Instagram 게시물',
        sources: thumb?.sources || [] });
      else if (thumb?.src) {
        // Retain a working fallback URL; add newly revealed lazy-load sources.
        const previous = Array.isArray(old.sources) ? old.sources : [old.src].filter(Boolean);
        const incoming = Array.isArray(thumb.sources) ? thumb.sources : [thumb.src];
        const discovered = incoming.filter(url => !previous.includes(url));
        if (discovered.length || !old.src) {
          old.sources = [...new Set([...previous, ...incoming])].slice(0, 8);
          if (!old.src) old.src = thumb.src;
          old.alt = thumb.alt || old.alt;
          lastThumbnailSourceUpdates += 1;
          thumbnailsChanged = true;
        }
      }
    }
    updateFloating();
    if (isOpen && (force || thumbnailsChanged || lastRenderCount !== postMap.size)) render();
  }

  function buildPostCard(post) {
    const card = createEl('a', 'card');
    card.href = post.url;
    card.target = '_blank';
    card.rel = 'noopener noreferrer';
    card.title = `${formatEstimatedTime(post.estimatedMs)} (추정 시각) · Instagram에서 열기`;
    const media = createEl('div', 'media');
    if (post.src) {
      const img = createEl('img', 'thumb');
      img.alt = post.alt;
      img.loading = 'eager';
      img.fetchPriority = 'high';
      img.referrerPolicy = 'no-referrer-when-downgrade';
      const alternatives = getThumbnailAlternatives(post, post.failedUrls);
      let active = 0;
      img.addEventListener('error', () => {
        renderedThumbnailFailures += 1;
        if (active + 1 < alternatives.length && img.isConnected) {
          active += 1;
          b37FallbackAttempts += 1;
          img.src = alternatives[active];
        } else if (img.parentNode) {
          b37FallbackExhausted += 1;
          // Only retry after the scan discovers NEW URLs, not every repaint.
          post.failedUrls = new Set([...(post.failedUrls || []), ...alternatives]);
          img.replaceWith(createEl('div', 'empty-thumb', '이미지 로드 실패 · 원본 열기'));
        }
      });
      img.addEventListener('load', () => {
        if (active > 0 && post.src !== alternatives[active]) {
          post.src = alternatives[active];
          b37FallbackRecovered += 1;
        }
      });
      media.appendChild(img);
      img.src = alternatives[0] || post.src;
    } else {
      if (post.videoFrame) media.appendChild(post.videoFrame);
      else media.appendChild(createEl('div', 'empty-thumb', '미리보기 없음'));
    }
    media.appendChild(createEl('span', 'date', formatEstimatedTime(post.estimatedMs)));
    card.appendChild(media);
    return card;
  }

  function render() {
    if (!grid || !summary) return;
    const sorted = Array.from(postMap.values()).sort(core.newerFirst);
    summary.textContent = `게시물 ${sorted.length}개 · 게시 시각은 추정값입니다.`;
    const guidance = shadow.querySelector('#ig-recent-guidance');
    const retry = shadow.querySelector('#ig-recent-hash-alternative');
    const alternateUrl = core.hashtagWithoutHashUrl(location.href);
    if (guidance && retry) {
      guidance.hidden = sorted.length !== 0 || !alternateUrl;
      if (alternateUrl) retry.href = alternateUrl;
      else retry.removeAttribute('href');
    }
    // Remove cards from the previous query; never keep stale posts.
    const active = new Set(sorted.map(p => p.shortcode));
    for (const [key, card] of cardMap) {
      if (!active.has(key)) { for (const video of card.querySelectorAll('video.b69-video-frame')) b71Release(video); card.remove(); cardMap.delete(key); }
    }
    if (!sorted.length) {
      grid.replaceChildren(createEl('p', 'empty',
        '게시물 썸네일을 찾지 못했습니다. 인스타그램 검색 결과를 연 뒤 다시 스캔하세요.'));
      lastRenderCount = 0;
      return;
    }
    const empty = grid.querySelector('.empty');
    if (empty) empty.remove();
    const previousScrollTop = grid.scrollTop;
    for (let index = 0; index < sorted.length; ++index) {
      const post = sorted[index];
      let card = cardMap.get(post.shortcode);
      if (card) {
        // Late lazy-load discoveries are allowed to use sources beyond the
        // first four after earlier candidates fail.
        const untriedSources = getThumbnailAlternatives(post, post.failedUrls);
        const needsNewSource = !!post.src && !card.querySelector('img.thumb') &&
          untriedSources.length > 0;
        if (needsNewSource || (post.videoFrame && !card.querySelector('video.b69-video-frame'))) {
          const replacement = buildPostCard(post);
          card.replaceWith(replacement);
          card = replacement;
          cardMap.set(post.shortcode, card);
          b38ThumbnailRefreshCount += 1;
        } else {
          b38ReusedCardCount += 1;
        }
      } else {
        card = buildPostCard(post);
        cardMap.set(post.shortcode, card);
      }
      // Move an existing card only if its sort position changed. Other nodes
      // retain their <img> elements and ongoing load/fallback state.
      if (grid.children[index] !== card) {
        grid.insertBefore(card, grid.children[index] || null);
      }
    }
    grid.scrollTop = previousScrollTop;
    lastRenderCount = sorted.length;
    b71ScheduleVideos();
  }

  function togglePanel(open) {
    if (!panel) return;
    isOpen = open;
    panel.classList.toggle('open', open);
    if (open) { verifyAndShow(); }
    b71ScheduleVideos();
  }

  function b73SafeScan(force = false) {
    try { scan(force); }
    catch (_error) {
      // Existing fallback button remains available after a failed scan.
      if (window.__IG_RECENT_B78_SYNC_LAUNCHER__)
        window.__IG_RECENT_B78_SYNC_LAUNCHER__();
    }
  }

  // B74: let the independent mobile launcher open the existing sorter.
  window.__IG_RECENT_B78_OPEN__=function(){
    try{
      scan(true);
      buildUI();
      if(!panel)return false;
      togglePanel(true);
      return true;
    }catch(_){return false;}
  };

  function scheduleScan() {
    if (scanTimer !== null) clearTimeout(scanTimer);
    scanTimer = setTimeout(() => {
      scanTimer = null;
      b73SafeScan();
    }, 350);
  }

  function start() {
    b73SafeScan();
    // B50: Instagram may replace lazy thumbnail src/srcset in-place without
    // inserting/removing nodes. Child-list-only watching missed those updates.
    // Ignore our own panel and unrelated CSS animations; no network/API use.
    const observer = new MutationObserver(records => {
      for (const change of records) {
        const target = change.target;
        if (!(target instanceof Element)) continue;
        if (host && (target === host || host.contains(target))) continue;
        if (change.type === 'attributes') {
          const media = target.matches('img, picture source, video');
          const background = change.attributeName === 'style' &&
            !!target.style?.backgroundImage && target.style.backgroundImage !== 'none';
          if (!media && !background) continue;
          b50MediaAttributeUpdates += 1;
        }
        scheduleScan();
        break;
      }
    });
    observer.observe(document.documentElement, {
      childList: true, subtree: true, attributes: true,
      attributeFilter: ['src', 'srcset', 'data-src', 'poster', 'style']
    });
    window.addEventListener('popstate', scheduleScan);
    window.addEventListener('hashchange', scheduleScan);
    document.addEventListener('input', event => {
      if (getSearchSurface()?.root.contains(event.target)) scheduleScan();
    }, true);
    // No persistent timers in the offline smoke fixture; production SPA URL check only.
    if (location.protocol !== 'file:') {
      setInterval(() => { if (location.href !== lastUrl) scheduleScan(); }, 1200);
    }
  }
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', start, { once: true });
  } else {
    start();
  }
})();
