# Patched Statement

This document describes the patches applied to the `v1.273.0-patched` branch of `MetaCubeX/metacubexd`.

---

## Patch 1: Load-Balance Active Node Tagging & Strategy Indicators

### 1. Issues & Motivation
In Mihomo proxy groups configured with `type: load-balance` (especially the `least-load` strategy), multiple candidate nodes are actively selected into a dynamic balancing pool.
* Previously, the dashboard only indicated the single last-dialed node (`now`), giving no visual indication of which other candidate nodes were currently qualified and actively handling balanced connections.
* The specific balancing strategy (e.g. `least-load`, `round-robin`, `consistent-hashing`, `sticky-sessions`) was not shown in group headers.

### 2. Implementation Details
* **`packages/ui/types/index.ts`**:
  * Extended the `Proxy` interface with optional `strategy?: string` and `active?: string[]` metadata fields received from the Mihomo `/proxies` REST API.
* **`packages/ui/components/ProxyNodeCard.vue`**:
  * Added `isActiveInLB?: boolean` and `lbStrategy?: string` props.
  * Added dynamic `proxy-card--active-lb` CSS styling with secondary glow accents (`border-secondary/70`, `shadow-[0_0_12px_rgba(...)]`).
  * Rendered a pulsing **`LB`** badge with a localized tooltip (*"Active in {strategy}"*).
  * Dynamically adjusted badge horizontal positioning when `isRecommended` is true to prevent visual badge collisions.
* **`packages/ui/components/ProxyNodeListItem.vue`**:
  * Added `isActiveInLB` and `lbStrategy` props.
  * Added `bg-secondary/10` and `shadow-[inset_2px_0_var(--color-secondary)]` styling with a pulsing **`LB`** badge.
* **`packages/ui/components/ProxyNodeTableRow.vue`**:
  * Added `isActiveInLB` and `lbStrategy` props.
  * Rendered the pulsing **`LB`** badge in place of the checkmark status column.
* **`packages/ui/components/ProxyMasterDetail.vue`**:
  * Added strategy pill badge in the right-side detail group header.
  * Forwarded `isActiveInLB` and `lbStrategy` to `ProxyNodeListItem`.
* **`packages/ui/components/ProxyPreviewDots.vue` & `packages/ui/components/ProxyNodePreview.vue`**:
  * Added `activeNodes?: string[]` prop passthrough.
  * Rendered active candidate donut rings around preview dots.
* **`packages/ui/pages/proxies.vue`**:
  * Added strategy badge pill (e.g. `LEAST-LOAD`) in `ProxyGroupTitle`.
  * Forwarded `activeNodes` to `ProxyNodePreview` and passed `isActiveInLB` & `lbStrategy` down to all node card/list/table components.

---

## Patch 2: Deterministic Natural Alphanumeric Proxy Node Ordering

### 1. Issues & Motivation
* When displaying proxy nodes in proxy groups, node ordering was frequently erratic or jumping.
* Standard alphabetical ASCII sorting sorted numbered nodes unnaturally (e.g. `w106-vless-reality` was sorted before `w2-vless-reality`).
* In `sortProxiesByOrderingType`, comparison of multiple disconnected nodes violated JavaScript's strict weak ordering requirement (`prevDis && nextDis => 1` returned asymmetry), which caused non-deterministic sort outcomes in browsers.

### 2. Implementation Details
* **`packages/ui/utils/index.ts`**:
  * Implemented natural alphanumeric comparison helper:
    `compareNatural = (a, b) => a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' })`
  * Applied `compareNatural` across `NAME_ASC`, `NAME_DESC`, and fallback tie-breakers.
  * Fixed strict weak ordering bug by ensuring `prevDis && nextDis` produces `diff = 0`, allowing disconnected nodes to sink to the bottom while sorting stably among themselves via natural name comparison.

---

## Patch 3: Relative Subdirectory BaseURL Deployment Fix

### 1. Issues & Motivation
* When dashboard static assets are hosted directly inside Mihomo on embedded routers (`/var/lib/mihomo/ui`), the web UI is accessed under the `/ui/` subpath (`http://<router-ip>:9093/ui/`).
* Nuxt's default configuration generated absolute paths (`/_nuxt/...`), resulting in 404 HTTP errors for scripts and stylesheets requested at the domain root.

### 2. Implementation Details
* **`packages/ui/nuxt.config.ts`**:
  * Configured `app.baseURL = process.env.NUXT_APP_BASE_URL || './'`.
  * Configured `app.buildAssetsDir = process.env.NUXT_APP_BUILD_ASSETS_DIR || '_nuxt/'`.
* **`package.json`**:
  * Updated `build:ui` script to explicitly set `NUXT_APP_BASE_URL=./`.

---

## Patch 4: Group RTT Test & Provider Proxy Latency History Synchronization Fix

### 1. Issues & Motivation
* When triggering an RTT test for an entire proxy group, the latest delay and average RTT numbers refreshed in the corner of node cards, but hovering over the cards revealed that some nodes (particularly nodes defined in both root config and proxy providers) had no historical test records in the tooltip.
* **Root Cause 1**: `recordLatencyTestResults()` (used during batch group delay tests) only invoked `replaceNodeLatency()`, omitting the required `appendLatencyHistoryEntry()` call which pushes the timestamped test record into the node's `latencyTestHistory` queue.
* **Root Cause 2**: In `fetchProxiesImpl()`, when proxy nodes were defined under top-level `proxies:` in `clash.yaml` (which have empty `history: []` in `GET /proxies`) and also loaded via `proxy-providers` (which have full 10-item `history` and `extra`), the provider's richer data was filtered out by `!(proxy.name in proxiesData)`.

### 2. Implementation Details
* **`packages/ui/stores/proxies.ts`**:
  * Unified latency history recording by directly invoking `appendLatencyHistoryEntry(resolvedNodeName, finalTestUrl, resultDelay)` inside `recordLatencyTestResult()`.
  * Merged provider `history`, `extra`, and `provider` metadata into root proxies when shared between `proxiesData` and `proxy-providers`.
  * Handled failed/timed-out nodes during group delay tests by recording timeout failure entries (`NOT_CONNECTED`) for unreturned members.
  * All nodes across groups now consistently preserve and display their full latency histories, statistics, and tooltips.

---

## Release History & Tags

* **`v1.273.0+patch1`** / **`v1.273.0-patch1`**: Initial load-balance active tagging, natural sorting, and relative baseURL fixes.
* **`v1.273.0+patch2`** / **`v1.273.0-patch2`**: Version bump, full dynamic API integration with Mihomo `v1.19.30+patch6`, and group RTT history recording fix.
