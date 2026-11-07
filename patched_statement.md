# Codebase Change Statement: MetaCubeXD

**Current Codebase:** `zhfreal/metacubexd` (`main-patched`, Release `v1.274.4-patched`)  
**Original Upstream Baseline:** `MetaCubeX/metacubexd` (Release `v1.274.4`)  
**Repository Location:** `./metacubexd-mine`  
**Status:** Production-Ready, Tested across Desktop, Web, and Embedded Router Deployments

---

## 1. Executive Summary

This document specifies the complete architectural, functional, and user-experience delta between the current codebase (`zhfreal/metacubexd @ main-patched`) and the original upstream baseline (`MetaCubeX/metacubexd v1.274.4`).

The modifications against original upstream are organized into two principal categories:
1. **Type 1: New Features & UI/UX Extensions**: Load-balance active candidate node tagging with real-time strategy indicators (`least-load`, `round-robin`, `consistent-hashing`, `sticky-sessions`), pulsing visual badges, preview dot donut rings, and deterministic natural alphanumeric proxy node ordering.
2. **Type 2: Bug Fixes, Concurrency, Latency Synchronization & Deployment Remediations**: Complete synchronization of group RTT test history and provider metadata, second-resolution epoch timestamp deduplication eliminating ghost records, strict weak ordering compliance for disconnected node sorting stability, and relative subdirectory baseURL deployment for embedded router hosting (`/ui/`).

All changes are maintained without patch history narratives or version logs, presenting a clean and direct technical specification of how this codebase differs from the original upstream repository.

---

## 2. Type 1: New Features & UI/UX Extensions

### 2.1 Load-Balance Active Candidate Node Tagging & Strategy Indicators (`packages/ui/`)
* **REST API Schema Extension (`packages/ui/types/index.ts`)**:
  - Extended the `Proxy` TypeScript interface with optional `strategy?: string` and `active?: string[]` metadata fields received from the Mihomo `/proxies` REST API.
  - Exposes the dynamic balancing pool members computed by Mihomo's load-balancer strategies (specifically `least-load`, `round-robin`, `consistent-hashing`, and `sticky-sessions`) directly to UI renderers.
* **Active Load-Balance Node Visual Accents (`packages/ui/components/ProxyNodeCard.vue`)**:
  - Added `isActiveInLB?: boolean` and `lbStrategy?: string` component props.
  - Implemented dynamic `proxy-card--active-lb` CSS styling with secondary glow accents (`border-secondary/70`, `shadow-[0_0_12px_rgba(var(--color-secondary),0.35)]`).
  - Rendered a pulsing **`LB`** badge with a localized tooltip (_"Active in {strategy}"_).
  - Dynamically adjusted badge horizontal positioning when `isRecommended` is true to prevent visual badge collisions.
  - Made the `LB` badge consistently visible across all candidate pool nodes (including the node currently selected as `now`).
* **List Item & Table Row Visual Indicators (`ProxyNodeListItem.vue`, `ProxyNodeTableRow.vue`)**:
  - Added `isActiveInLB` and `lbStrategy` prop passthrough.
  - In `ProxyNodeListItem.vue`: Added `bg-secondary/10` background tint and `shadow-[inset_2px_0_var(--color-secondary)]` border styling alongside the pulsing **`LB`** badge.
  - In `ProxyNodeTableRow.vue`: Rendered the pulsing **`LB`** badge in place of the checkmark status column for non-`now` active candidate nodes.
* **Master-Detail View Strategy Badge (`ProxyMasterDetail.vue`)**:
  - Added dynamic strategy pill badge (e.g. `LEAST-LOAD`, `CONSISTENT-HASHING`) in the right-side detail group header.
  - Forwarded `isActiveInLB` and `lbStrategy` props down to all child `ProxyNodeListItem` instances.
* **Preview Dots & Group Previews (`ProxyPreviewDots.vue`, `ProxyNodePreview.vue`)**:
  - Added `activeNodes?: string[]` prop passthrough.
  - Rendered active candidate donut rings around preview dots, visually highlighting which nodes are actively serving load-balanced traffic directly from compact group summaries.
* **Group Title Strategy Pill (`packages/ui/pages/proxies.vue`)**:
  - Rendered strategy badge pill in `ProxyGroupTitle`.
  - Forwarded `activeNodes` to `ProxyNodePreview` and bound `isActiveInLB` & `lbStrategy` down to all node card, list, and table components.

### 2.2 Deterministic Natural Alphanumeric Proxy Node Ordering (`packages/ui/utils/index.ts`)
* **Natural Alphanumeric Sorting Engine**:
  - Implemented `compareNatural(a: string, b: string): number`:
    ```typescript
    export const compareNatural = (a: string, b: string): number =>
      a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' })
    ```
  - Replaced naive ASCII string comparisons across `NAME_ASC` and `NAME_DESC` ordering types. Multi-digit numbered nodes (e.g. `node-2`, `node-10`, `w2-vless`, `w106-vless`) now sort in natural human numerical order rather than standard ASCII character order (`w106` before `w2`).
  - Applied `compareNatural` as the universal tie-breaker across latency, upload, and download sort modes, guaranteeing identical, stable order across multiple UI re-renders and browser engines.

---

## 3. Type 2: Bug Fixes, Concurrency, Latency Synchronization & Deployment Remediations

### 3.1 Group RTT Test & Provider Proxy Latency History Synchronization Fix (`packages/ui/stores/proxies.ts`)
* **Root Cause in Upstream Baseline**:
  - In upstream, triggering an RTT test for an entire proxy group updated the latest delay and average RTT numbers in the corner of node cards, but hovering over the cards revealed that some nodes (particularly nodes defined in both root config and proxy providers) had no historical test records in the tooltip.
  - *Defect 1*: `recordLatencyTestResults()` (used during batch group delay tests) only called `replaceNodeLatency()`, omitting the required `appendLatencyHistoryEntry()` call which pushes the timestamped test record into the node's `latencyTestHistory` queue.
  - *Defect 2*: In `fetchProxiesImpl()`, when proxy nodes were defined under top-level `proxies:` in `clash.yaml` (which have empty `history: []` in `GET /proxies`) and also loaded via `proxy-providers` (which have full 10-item `history` and `extra`), the provider's richer data was filtered out by `!(proxy.name in proxiesData)`.
* **Resolution**:
  - Unified latency history recording by directly invoking `appendLatencyHistoryEntry(resolvedNodeName, finalTestUrl, resultDelay)` inside `recordLatencyTestResult()`.
  - Merged provider `history`, `extra`, and `provider` metadata into root proxies when shared between `proxiesData` and `proxy-providers`.
  - Handled failed/timed-out nodes during group delay tests by recording timeout failure entries (`NOT_CONNECTED`) for unreturned members.
  - Chronologically derived `alive` state from the latest merged history entry (`latestEntry.delay > 0`), ensuring fresh test results take precedence over stale state.
  - All nodes across groups now consistently preserve and display their full latency histories, statistics, and tooltips.

### 3.2 Group RTT Test History Deduplication & Second-Resolution Epoch Normalization (`packages/ui/stores/proxies.ts`)
* **Root Cause in Upstream Baseline**:
  - When performing a group RTT test on a proxy group (e.g. `PROXY`), the frontend UI showed an incremented history record count (e.g. up to 8 records), but upon hard-refreshing the page (`SHIFT + F5`), the count dropped back down (to 7 records), giving the impression that the latest manual test record went missing.
  - *Defect 1*: `proxyGroupLatencyTest()` called `await fetchProxies()` (which loads the actual updated records from kernel memory) and immediately followed with `recordLatencyTestResults()`. `recordLatencyTestResults()` invoked `appendLatencyHistoryEntry()`, creating a synthetic client-side timestamp record `{ time: new Date().toISOString(), delay }` on top of the already fetched records.
  - *Defect 2*: `mergeDelayHistories()` previously deduplicated entries using exact raw string matching on `item.time`. Because the Go backend produces timestamps with timezone offset strings (e.g. `2026-08-20T20:12:51.93046656+08:00`) while JavaScript generates UTC ISO strings (`2026-08-20T12:12:52.123Z`), the string keys did not match and created a ghost duplicate entry in client memory.
* **Resolution**:
  - Removed redundant post-fetch `recordLatencyTestResults()` from `proxyGroupLatencyTest()`, relying on `fetchProxies()` to accurately synchronize the authentic kernel state.
  - Removed duplicate `appendLatencyHistoryEntry()` call in `proxyProviderLatencyTest()`.
  - Updated `mergeDelayHistories()` to index and deduplicate entries by epoch second resolution (`Math.round(new Date(time).getTime() / 1000)`), making history deduplication robust against timezone formatting differences and sub-second variations.
  - Added unit test assertions ensuring latency history length is strictly preserved and deduplicated during group RTT testing and across multiple providers.

### 3.3 Strict Weak Ordering Disconnected Node Stability Fix (`packages/ui/utils/index.ts`)
* **Root Cause in Upstream Baseline**:
  - In `sortProxiesByOrderingType`, the comparison logic for disconnected nodes violated JavaScript's strict weak ordering contract:
    ```typescript
    // Upstream defective logic:
    if (prevDis && nextDis) return 1 // Asymmetric! compare(a, b) === 1 AND compare(b, a) === 1
    ```
  - Because `compare(a, b)` and `compare(b, a)` both returned `1`, JavaScript engine sorting algorithms (such as V8's Timsort) produced non-deterministic sort outcomes, causing proxy cards to jump positions erratically on re-renders.
* **Resolution**:
  - Fixed strict weak ordering: when both nodes are disconnected (`prevDis && nextDis`), `diff = 0`.
  - Disconnected nodes sink to the bottom of the list while sorting stably among themselves via natural alphanumeric name comparison (`compareNatural`).

### 3.4 Relative Subdirectory BaseURL Deployment for Embedded Router Environments (`packages/ui/nuxt.config.ts`, `package.json`)
* **Root Cause in Upstream Baseline**:
  - When dashboard static assets are hosted directly inside Mihomo on embedded routers (`/var/lib/mihomo/ui`), the web UI is accessed under the `/ui/` subpath (`http://<router-ip>:9093/ui/`).
  - Nuxt's default configuration generated absolute paths (`/_nuxt/...`), resulting in 404 HTTP errors for scripts and stylesheets requested at the domain root.
* **Resolution**:
  - In `packages/ui/nuxt.config.ts`:
    - Configured `app.baseURL = process.env.NUXT_APP_BASE_URL || './'`.
    - Configured `app.buildAssetsDir = process.env.NUXT_APP_BUILD_ASSETS_DIR || '_nuxt/'`.
  - In `package.json`:
    - Updated `build:ui` script to explicitly set `NUXT_APP_BASE_URL=./`.
  - Static distribution builds now deploy seamlessly to root paths, arbitrary reverse-proxy subpaths, and embedded Mihomo/OpenWrt `/ui/` endpoints without asset 404s.

---

## 4. Comprehensive File Modification & Addition Matrix

| File Path | Status | Change Type | Description of Delta Relative to Upstream Baseline (`v1.274.4`) |
| :--- | :--- | :--- | :--- |
| `packages/ui/types/index.ts` | **Modified** | **Type 1** (Feature) | Extended `Proxy` interface with `strategy?: string` and `active?: string[]` metadata fields. |
| `packages/ui/components/ProxyNodeCard.vue` | **Modified** | **Type 1** (Feature) | Added `isActiveInLB` and `lbStrategy` props, `proxy-card--active-lb` CSS glow styling, pulsing `LB` badge with tooltip, and collision prevention with `isRecommended`. |
| `packages/ui/components/ProxyNodeListItem.vue` | **Modified** | **Type 1** (Feature) | Added `isActiveInLB` and `lbStrategy` props, `bg-secondary/10` background tint, inset shadow styling, and pulsing `LB` badge. |
| `packages/ui/components/ProxyNodeTableRow.vue` | **Modified** | **Type 1** (Feature) | Added `isActiveInLB` and `lbStrategy` props, rendering pulsing `LB` badge in status column. |
| `packages/ui/components/ProxyMasterDetail.vue` | **Modified** | **Type 1** (Feature) | Added strategy pill in detail group header and forwarded active candidate props to list items. |
| `packages/ui/components/ProxyPreviewDots.vue` | **Modified** | **Type 1** (Feature) | Added `activeNodes?: string[]` prop passthrough and active candidate donut rings around preview dots. |
| `packages/ui/components/ProxyNodePreview.vue` | **Modified** | **Type 1** (Feature) | Forwarded `activeNodes` to `ProxyPreviewDots`. |
| `packages/ui/pages/proxies.vue` | **Modified** | **Type 1** (Feature) | Added strategy badge pill in `ProxyGroupTitle`, forwarded active candidates to preview and all node cards/lists/tables. |
| `packages/ui/utils/index.ts` | **Modified** | **Type 1** & **Type 2** | Implemented `compareNatural` natural alphanumeric string comparison helper; fixed strict weak ordering bug for disconnected nodes (`prevDis && nextDis => diff = 0`). |
| `packages/ui/stores/proxies.ts` | **Modified** | **Type 2** (Bugfix) | Unified latency history recording with `appendLatencyHistoryEntry`, merged provider metadata into root proxies, removed redundant duplicate recording in `proxyGroupLatencyTest`, and implemented second-resolution epoch deduplication in `mergeDelayHistories`. |
| `packages/ui/nuxt.config.ts` | **Modified** | **Type 2** (Deployment) | Configured `app.baseURL = './'` and `app.buildAssetsDir = '_nuxt/'` for relative subpath hosting. |
| `package.json` | **Modified** | **Type 2** (Deployment) | Updated `build:ui` script to set `NUXT_APP_BASE_URL=./`. |
| `packages/ui/stores/proxies.test.ts` | **Modified** | **Testing** | Added unit tests verifying latency history preservation, second-resolution deduplication, and provider metadata merging. |
