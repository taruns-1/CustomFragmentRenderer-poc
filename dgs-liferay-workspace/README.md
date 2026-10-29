# LPP-66052: Fragment Cache Workaround for Multiple Virtual Hosts

This Liferay Workspace reproduces [LPP-66052](https://liferay.atlassian.net/browse/LPP-66052) and contains an OSGi override that works around it on DXP 2026.Q1.

| | |
| --- | --- |
| Ticket | [LPP-66052](https://liferay.atlassian.net/browse/LPP-66052) |
| Follow-up feature | LPD-111705 (admin setting for which hosts may use fragment caching) |
| DXP version | `dxp-2026.q1.0-lts` |
| Overridden class | `com.liferay.fragment.internal.renderer.FragmentEntryFragmentRenderer` ([source, tag `2026.q1.0`](https://github.com/liferay/liferay-portal/blob/2026.q1.0/modules/apps/fragment/fragment-impl/src/main/java/com/liferay/fragment/internal/renderer/FragmentEntryFragmentRenderer.java)) |

## The Problem

When one site is reachable from more than one host, for example a public virtual host and the instance host using `/web/<site>`, fragments can render links that point to the wrong host.

`FragmentEntryFragmentRenderer` uses two caches, and neither includes the host in its key:

- **HTML cache** (`FragmentEntryLinkCache`): the full rendered output of cacheable fragments.
- **Configuration cache** (`_configurationPortalCache`): the `configuration` JSON passed to fragment JavaScript. It is used for all fragments, including non-cacheable ones.

The first host to render a fragment stores its URLs, and every other host receives them.

## The Workaround

The module turns off both caches for one host, `liferay.intranet` (the `_NON_CACHEABLE_SERVER_NAME` constant). The host is read from `ThemeDisplay`, and requests from that host:

- never read from the HTML cache or the configuration cache, so they always render with their own URLs;
- never write to either cache, so other hosts never receive their URLs.

All other hosts keep using the caches exactly as before.

## Repository Layout

| Path | Contents |
| --- | --- |
| `modules/lpp-66052-fragment-renderer` | The OSGi override module |
| `fragments/lpp-66052` | Test fragment (`cacheable: true`) that prints the rendering host and the navigation menu URLs |

## Changes

`CustomFragmentEntryFragmentRenderer` is a copy of the original class from tag `2026.q1.0`. Every change is wrapped in `// LPP-66052 Customization - START` and `// LPP-66052 Customization - END` comments.

### 1. Register the override

```java
@Component(
	property = "service.ranking:Integer=100", service = FragmentRenderer.class
)
public class CustomFragmentEntryFragmentRenderer implements FragmentRenderer {
```

`FragmentRendererRegistryImpl` picks the highest ranked `FragmentRenderer` for each renderer key. The original component has no ranking (0), so this one replaces it without changing any Liferay module. The original class is in an internal, unexported package, so it is copied rather than extended.

### 2. Add the host check (`_isCacheableHost`)

```java
private static final String _NON_CACHEABLE_SERVER_NAME =
	"liferay.intranet";

private boolean _isCacheableHost(
	HttpServletRequest httpServletRequest,
	String nonCacheableServerName) {

	ThemeDisplay themeDisplay =
		(ThemeDisplay)httpServletRequest.getAttribute(
			WebKeys.THEME_DISPLAY);

	if (themeDisplay == null) {
		return false;
	}

	return !StringUtil.equalsIgnoreCase(
		themeDisplay.getServerName(), nonCacheableServerName);
}
```

This new method decides whether the current request may use the caches. Both caches use it, so they always make the same decision. The excluded host is kept in the `_NON_CACHEABLE_SERVER_NAME` constant and passed in as an argument, so the method itself has no hardcoded host.

- `getServerName()` is the host the visitor used. Liferay uses the same value to build URLs. Unlike `getPortalDomain()`, it is not replaced by the `web.server.host` portal property.
- Host names are not case sensitive, so the comparison ignores case.
- If `ThemeDisplay` is missing, the host cannot be determined, so the caches are skipped.

### 3. Skip the HTML cache for the excluded host (`_renderFragmentEntryLink`)

```java
boolean cacheableHost = _isCacheableHost(
	httpServletRequest, _NON_CACHEABLE_SERVER_NAME);

boolean cacheable = false;

if (cacheableHost) {
	cacheable = _isCacheable(fragmentEntryLink, fragmentRendererContext);
}
```

`cacheable` controls both the read from and the write to `FragmentEntryLinkCache`. When it is `false`, the existing code renders the fragment fresh and returns it without storing it.

The result of the host check is also passed to `_toConfiguration`:

```java
content = _renderFragmentEntry(
	_toConfiguration(cacheableHost, fragmentEntryLink, fragmentRendererContext),
	css, fragmentRendererContext, html, httpServletRequest, nonce);
```

### 4. Skip the configuration cache for the excluded host (`_toConfiguration`)

```java
Map.Entry<Long, String> entry = null;

if (cacheableHost) {
	entry = _configurationPortalCache.get(key);
}

// ... unchanged: build the configuration JSON ...

if (cacheableHost) {
	_configurationPortalCache.put(key, entry);
}
```

Both the `get` and the `put` are guarded. Skipping only the `get` would let the excluded host write its URLs into the shared entry.

This change is needed even with change 3 in place:

- Non-cacheable fragments always reach `_toConfiguration`.
- Cacheable fragments reach it for the excluded host, which now skips the HTML cache.

## Build and Deploy

```bash
blade gw :modules:lpp-66052-fragment-renderer:deploy
```

Check that the bundle started:

```bash
grep "lpp66052.fragment.renderer" bundles/logs/liferay.$(date +%F).log | tail -1
# STARTED com.liferay.lpp66052.fragment.renderer_1.0.0
```

After deploying, go to **Control Panel → Server Administration** and run **Clear content cached across the cluster**, so no HTML cached before the deploy is served.

## Testing

1. Add both hosts to `/etc/hosts`:
   ```
   127.0.0.1  www.liferay.public
   127.0.0.1  liferay.intranet
   ```
2. Give the site the public virtual host `www.liferay.public`.
3. Add the test fragment from `fragments/lpp-66052` to a content page and publish it.
4. Clear the content cache, then open the page as a guest on both hosts:
   - `http://liferay.intranet:8080/web/<site>/<page>`
   - `http://www.liferay.public:8080/<page>`
5. Each host should only show its own URLs. The fragment's "Rendered by" line shows which host rendered it and when:
   - On the public host, the time stays the same across refreshes (served from the cache).
   - On the intranet host, the time changes on every refresh (always rendered fresh).

   Only check the URLs inside the fragment. The `<link data-senna-track="temporary">` canonical and alternate language tags in the page `<head>` always point to the site's virtual host (`www.liferay.public`), even on the intranet host. That is standard Liferay SEO output, not a cache leak.

## Limitations

- **Temporary workaround only.** The class is a copy of an internal Liferay class and must be compared with the original and updated on every DXP update.
- **One excluded host.** Only `liferay.intranet` skips the caches. If more hosts serve the same site, the hosts that still use the cache share entries with each other. A safer variant allows only one host to use the cache.
- **Hardcoded host.** The host name is the `_NON_CACHEABLE_SERVER_NAME` constant. Because `_isCacheableHost` already takes the host as an argument, reading it from a portal property or OSGi configuration instead would only change the call site, and would let each environment set its own without a rebuild.
- **Performance.** Requests from the excluded host render every fragment on every request, the same as marking all fragments non-cacheable for that host.
