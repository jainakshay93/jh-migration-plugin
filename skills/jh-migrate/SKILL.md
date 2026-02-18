---
name: jh-migrate
description: StoresOnline to WordPress/WooCommerce migration orchestrator. Automates the full migration process using the customer-xml-importer plugin.
user_invocable: true
arguments: "[client-data-path]"
---

# StoresOnline to WordPress/WooCommerce Migration

You are a migration orchestrator for StoresOnline → WordPress/WooCommerce data migrations using the `customer-xml-importer` plugin (v1.1.0+).

Execute the following phases IN ORDER. Do NOT skip phases. Ask all configuration questions BEFORE starting any migration work.

$ARGUMENTS

---

## AUTONOMOUS EXECUTION RULES

Once Phase 2 configuration is confirmed and Phase 3 begins, **execute ALL remaining phases autonomously without asking for user input or confirmation**. The user has already approved the migration plan. Do NOT:
- Ask "should I continue?" between steps
- Ask for permission to run Bash commands — chain them and run them
- Pause for confirmation between import phases
- Ask the user to verify intermediate results (just report them)

If a step fails, retry once with adjusted parameters. If it still fails, report the error and continue to the next step. Only stop and ask the user if a critical prerequisite is missing (e.g., WP-CLI not available, SSH connection completely broken).

### Minimizing Permission Prompts

To avoid repeated Bash permission prompts:
1. **Chain sequential commands** with `&&` into a single Bash call wherever possible
2. **Use `run_in_background: true`** for imports expected to take >2 minutes
3. **Combine verification queries** into a single SSH + shell command
4. For remote sites, batch all checks into one SSH call with semicolons or `&&`

### Command Construction for Remote Sites

For remote sites, ALL wp-cli commands must include `--path={WP_PATH}`:
```bash
ssh -p {PORT} -i {KEYFILE} -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new {USER}@{HOST} "wp <command> --path={WP_PATH}"
```

Store the full SSH prefix as `SSH_CMD` and reuse it for all remote commands. The `ServerAliveInterval` and `ServerAliveCountMax` prevent SSH broken pipe disconnects during long-running imports.

### PHP Script Upload Pattern for Remote Sites

For complex WordPress operations (menu creation, widget setup, theme config), write a PHP script to `/tmp/`, upload via `scp`, execute via `wp eval-file`, then cleanup:
```bash
# 1. Write PHP to local /tmp/
# 2. Upload:
scp -P {PORT} -i {KEYFILE} /tmp/script.php {USER}@{HOST}:/tmp/script.php
# 3. Execute:
ssh {SSH_CMD} "wp eval-file /tmp/script.php --path={WP_PATH}"
# 4. Cleanup:
ssh {SSH_CMD} "rm /tmp/script.php"
```

This avoids SSH quoting issues with complex PHP code containing quotes, arrays, and special characters.

---

## PHASE 0: Environment Detection

### Step 0.1: Site Type & Source Site URL

Use the `AskUserQuestion` tool to ask TWO questions:

**Question 1**: "Where is the target WordPress site running?"
- **Local (Local by Flywheel)** (Recommended for development) — WordPress is running locally via Local by Flywheel app
- **Remote server (SSH)** — WordPress is on a remote server accessible via SSH

**Question 2**: "What is the source (original) StoresOnline site URL?"
- User provides the full URL (e.g., `https://www.marysgardenpatch.com/`)
- This is used to auto-detect the site's layout, menus, sidebar, footer, and content

Store as: `SOURCE_SITE_URL`

### Step 0.2a: If Local by Flywheel

1. List available Local by Flywheel MySQL sockets:
   ```bash
   ls ~/Library/Application\ Support/Local/run/*/mysql/mysqld.sock 2>/dev/null
   ```
2. Use `AskUserQuestion` to ask the user to pick the correct socket ID from the list found. Explain: "Each Local by Flywheel site has a unique socket ID. You can find yours in the Local app under Site > Database tab."
3. Construct the full socket path: `localhost:$HOME/Library/Application Support/Local/run/{SOCKET_ID}/mysql/mysqld.sock`
4. Test the connection by running:
   ```bash
   wp option get siteurl --path="{WP_PATH}"
   ```
   where `WP_PATH` is the WordPress root (typically the `public` directory of the Local site).
5. Show the detected site URL to the user and ask for confirmation: "Is this the correct site: {siteurl}?"
6. If verification fails, ask the user to provide the correct socket ID and retry.

Set these variables for all subsequent phases:
- `WP_CMD`: `wp --path="{WP_PATH}"` (always include --path)
- `WP_PATH`: The WordPress installation root path
- `SITE_TYPE`: `local`

### Step 0.2b: If Remote Server

1. Use `AskUserQuestion` to gather ALL SSH connection details in a SINGLE question using multiple fields:
   - **SSH Username** (e.g., `root`, `deploy`, `www-data`)
   - **SSH Host** (e.g., `example.com` or `192.168.1.100`)
   - **SSH Port** (default: `22`)
   - **Authentication method**: "Password" or "SSH key file"
     - If key file: ask for path (e.g., `~/.ssh/id_rsa`)
     - If password: ask for the password

2. Construct and test the SSH connection:
   ```bash
   # ALWAYS include keepalive flags to prevent broken pipe on long imports:
   ssh -p {PORT} -i {KEYFILE} -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new {USER}@{HOST} echo "SSH connection successful"
   ```
3. If connection fails, show the error and ask the user to re-enter the credentials. Keep retrying until success or the user chooses to abort.

4. Test WP-CLI availability:
   ```bash
   ssh {SSH_CMD} "wp --version"
   ```
   If WP-CLI is not found, display install instructions and abort:
   ```
   WP-CLI is not installed on the remote server. Install it with:
   curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
   chmod +x wp-cli.phar
   sudo mv wp-cli.phar /usr/local/bin/wp
   ```

5. Ask for **remote WordPress root path** (e.g., `/var/www/html`, `/home/user/public_html`).

6. Verify the path:
   ```bash
   ssh {SSH_CMD} "wp option get siteurl --path={WP_REMOTE_PATH}"
   ```
   Show the detected site URL and ask for confirmation. If wrong, ask to re-enter the path.

7. Ask: "Are the XML data files and images already on the remote server, or do they need to be uploaded?"
   - If already on server: ask for the remote data directory path
   - If need upload: note that `scp`/`rsync` will be used in Phase 3

Set these variables:
- `SSH_CMD`: `ssh -p {PORT} -i {KEYFILE} -o ServerAliveInterval=60 -o ServerAliveCountMax=5 {USER}@{HOST}` (ALWAYS include keepalive)
- `WP_PATH`: Remote WordPress path
- `DATA_DIR`: Remote or local data directory
- `SITE_TYPE`: `remote`
- `NEEDS_UPLOAD`: `true` or `false`

---

## PHASE 1: Data Discovery & Analysis

### Step 1.1: Locate Data Files

Ask the user for the path to the client data directory containing the XML files.
- For local sites, default suggestion: `wp-content/client-brief/{client-name}/data/`
- For remote sites with files on server: ask for full remote path

### Step 1.2: Scan for Required XML Files

Check for each required XML file and report found/missing. Run as a SINGLE command:

```bash
# For local:
for f in Customer.xml CustomerGroup.xml AccountGroup.xml Product.xml Category.xml ProductCategory.xml Price.xml RewriteURL.xml Page.xml PageElement.xml AccountOrder.xml; do
  if [ -f "{DATA_DIR}/$f" ]; then
    COUNT=$(grep -c '<row>' "{DATA_DIR}/$f" 2>/dev/null || echo "0")
    echo "FOUND: $f ($COUNT records)"
  else
    echo "MISSING: $f"
  fi
done

# For remote (wrap entire loop in single SSH call):
ssh {SSH_CMD} 'for f in Customer.xml CustomerGroup.xml AccountGroup.xml Product.xml Category.xml ProductCategory.xml Price.xml RewriteURL.xml Page.xml PageElement.xml AccountOrder.xml; do if [ -f "{DATA_DIR}/$f" ]; then COUNT=$(grep -c "<row>" "{DATA_DIR}/$f" 2>/dev/null || echo "0"); echo "FOUND: $f ($COUNT records)"; else echo "MISSING: $f"; fi; done'
```

Present results as a table:

| File | Status | Records |
|------|--------|---------|
| Customer.xml | Found | 11,148 |
| ... | ... | ... |

Flag any MISSING required files. The minimum required files are:
- **Customer import**: Customer.xml, CustomerGroup.xml, AccountGroup.xml
- **Product import**: Product.xml, Category.xml, ProductCategory.xml, Price.xml
- **Page import**: Page.xml, PageElement.xml
- **Order import**: AccountOrder.xml

### Step 1.3: Scan for Image Folders

Look for image folders and count images in a SINGLE command:
```bash
# Local:
echo "=== FOLDERS ===" && find "{CLIENT_DIR}" -maxdepth 1 -type d | sort && echo "=== IMAGE COUNT ===" && find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) | wc -l

# Remote (single SSH call):
ssh {SSH_CMD} 'echo "=== FOLDERS ===" && find "{CLIENT_DIR}" -maxdepth 1 -type d | sort && echo "=== IMAGE COUNT ===" && find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) | wc -l'
```

Report the number of image folders and total image count.

### Step 1.4: Analyze Orders

If `AccountOrder.xml` exists, run BOTH commands in a single SSH call:
```bash
# Remote:
ssh {SSH_CMD} "wp cxi list-order-form-names '{DATA_DIR}/AccountOrder.xml' --path={WP_PATH} && wp cxi validate-orders '{DATA_DIR}/AccountOrder.xml' --path={WP_PATH}"

# Local:
wp cxi list-order-form-names "{DATA_DIR}/AccountOrder.xml" --path="{WP_PATH}" && wp cxi validate-orders "{DATA_DIR}/AccountOrder.xml" --path="{WP_PATH}"
```

Report form types with counts and validation results.

### Step 1.5: Present Summary

Display a comprehensive summary table with all data from Steps 1.2-1.4.

---

## PHASE 1.5: Source Site Analysis

**CRITICAL**: This phase auto-detects the source site's layout, menus, sidebar, footer, testimonials, and forms. The detected structure is used in Phase 3.9 to configure the migrated site automatically.

Use the Playwright MCP browser tools (`mcp__plugin_wisdmlabs-engineering_playwright__*`) to crawl the source site. If Playwright is not available, use `WebFetch` as a fallback.

### Step 1.5.1: Crawl Source Site Homepage

1. Navigate to `{SOURCE_SITE_URL}` using Playwright browser
2. Take an accessibility snapshot (`browser_snapshot`) of the full page
3. Also take a full-page screenshot for visual reference

### Step 1.5.2: Extract Header Navigation

From the snapshot, identify the **header/top navigation** links:
- Extract each link's text and URL path
- Note the order of items

Store as: `SOURCE_HEADER_LINKS` — array of `{title, url_path}` objects
Example: `[{title: "Home", path: "/"}, {title: "Site Map", path: "/sitemap/"}, ...]`

### Step 1.5.3: Extract Sidebar Structure

From the snapshot, identify the **left/right sidebar** content:

1. **Sidebar position**: Determine if sidebar is on the left or right (or none)
   Store as: `SOURCE_SIDEBAR_POSITION` (`left`, `right`, or `none`)

2. **Navigation links**: Extract all sidebar nav items with hierarchy:
   - Top-level items (e.g., "Home", "SPRING PLANTED BULBS", "Gardening FAQ")
   - Child items under each parent (e.g., subcategories under Spring/Fall Bulbs)
   Store as: `SOURCE_SIDEBAR_NAV` — array of `{title, url_path, children: [{title, url_path}]}`

3. **Widgets below navigation**: Identify any content below the nav links:
   - Testimonial blocks (look for quoted text, customer names)
   - Newsletter/email signup forms (look for form fields: name, email, submit)
   - Any other widget-like content
   Store as: `SOURCE_SIDEBAR_WIDGETS` — array of `{type, content}` where type is `testimonial`, `newsletter_form`, `custom_html`, etc.

For testimonials, extract:
- `quote_text`: The testimonial quote
- `attribution`: The author/location (e.g., "Jamie, Central Texas")

For newsletter forms, extract:
- `form_title`: Heading text (e.g., "Email Newsletter Signup!")
- `form_description`: Description/intro text
- `form_fields`: Array of field names (e.g., ["First Name", "Last Name", "Email Address"])
- `form_action`: Form submit URL path
- `submit_label`: Submit button text

### Step 1.5.4: Extract Footer Structure

From the snapshot, identify the **footer** content:

1. **Footer navigation links**: Extract link text and URLs
   Store as: `SOURCE_FOOTER_LINKS` — array of `{title, url_path}`

2. **Copyright text**: Extract the copyright/contact line
   Store as: `SOURCE_COPYRIGHT_TEXT` — the full text string

3. **Footer layout**: Note if footer has widget areas (e.g., Spring/Fall category columns) or just a simple menu + copyright
   Store as: `SOURCE_FOOTER_LAYOUT` (`menu-only`, `menu-with-widgets`)

### Step 1.5.5: Identify Additional Pages Needed

Compare detected menu/nav links against imported page slugs to find pages that need to be created (not in XML data):
- Common missing pages: Site Map, Gift Card Balance, Wish List, Privacy Policy, Cart, Checkout, My Account, Shop

Store as: `PAGES_TO_CREATE` — array of `{title, slug, content}` for pages not found in XML

### Step 1.5.6: Present Detected Structure

Display the complete detected structure for user confirmation:

```
=== SOURCE SITE STRUCTURE (Auto-Detected) ===

Header Navigation:
  Home | Site Map | Checkout | Gift Card Balance | USDA Hardiness Zones | Wish List | Login

Left Sidebar:
  Navigation:
    1. Home
    2. SPRING PLANTED BULBS (40 subcategories)
    3. FALL PLANTED BULBS (27 subcategories)
    4. Gardening FAQ
    5. Order & Shipping FAQ
    6. Contact Us
    7. Newsletter Signup & Discount
    8. Newsletter Archive
    9. Gift Card Balance
    10. Search
    11. Gift Certificates
    12. USDA Hardiness Zones
  Widgets:
    - Testimonial: "I am so IMPRESSED with your customer service!!" — Jamie, Central Texas
    - Newsletter Form: First Name, Last Name, Email Address → /newsletter-signup-discount/

Footer:
  Links: Home · Search · Order & Shipping FAQ · Gardening FAQ · Privacy Policy · Contact Us · Newsletter Signup & Discount · Newsletter Archive · Gift Card Balance · USDA Hardiness Zones
  Copyright: "Copyright © 2009 - Hummingbear Marketing, Inc. All rights reserved. | Phone: 1-888-321-0821 | Email: CustomerService@MarysGardenPatch.com"

Pages to Create (not in XML):
  - Site Map, Gift Card Balance, Cart, Checkout, My Account, Shop
```

This auto-detected structure will be replicated on the migrated site in Phase 3.9. The user does NOT need to manually specify any of these — they are all auto-detected.

---

## PHASE 2: Configuration Prompts

Gather ALL configuration decisions BEFORE starting migration. Use a SINGLE `AskUserQuestion` call with multiple questions where possible.

### Step 2.1: Combined Configuration Questions

Use `AskUserQuestion` with up to 4 questions at once:

**Question 1 - Product Variation Strategy:**
"How should products with variations (size, color, etc.) be imported?"
- **Standard variable products (Recommended)** — Uses built-in WooCommerce variable products. Free, no extra plugins needed.
- **WooCommerce Product Add-Ons** — Requires the paid "WooCommerce Product Add-Ons" plugin.

**Question 2 - Order Forms:**
Based on form names discovered in Phase 1:
"Which order form types should be imported as WooCommerce orders?"
- Show each form name with count, recommend "Secure Checkout"

**Question 3 - Default Country:**
"What is the default country code for customers with missing country data?"
- **US - United States (Recommended)**
- **CA - Canada**
- **GB - United Kingdom**

**Question 4 - Theme:**
"Which WordPress theme should be installed?"
- **Blocksy (Recommended)** — Best sidebar/header/footer control via theme mods, free, fast
- **Astra** — Lightweight, popular, but less sidebar control
- **Other** — Specify theme slug

Store as: `THEME_SLUG` (e.g., `blocksy`, `astra`)

### Step 2.2: Additional Questions

Use `AskUserQuestion` with up to 4 questions:

**Question 1 - Skip Images:**
"Should image migration be performed?"
- **Yes, copy images (Recommended for first run)**
- **No, skip images** — for re-runs where images already exist

**Question 2 - Homepage:**
"Which imported page should be the homepage?"
- **Auto-detect (Recommended)** — Looks for page with slug 'home' or 'index'
- **Specify slug** — Enter a custom page slug

Store as: `HOMEPAGE_SLUG` (default: `home`)

### Step 2.3: Addon Details (only if Add-Ons selected)

If user selected WooCommerce Product Add-Ons, ask:
- Addon type: select, radiobutton, or checkbox
- Required: yes/no
- Plugin zip file location

### Step 2.4: Order Status Mapping

Use `AskUserQuestion`:

**Question**: "Do you need to customize the order status mapping?"
- **No, use defaults (Recommended)** — Uses the built-in mapping (archived→completed, new→completed, backorder→on-hold, etc.)
- **Yes, customize mapping** — Specify WooCommerce status for each StoresOnline status

If user selects "Yes", present the default mapping and ask for each StoresOnline status what WooCommerce status it should map to. Use `AskUserQuestion` with up to 4 questions at a time, cycling through all StoresOnline statuses:

Default mapping:
| StoresOnline Status | Default WC Status | Valid WC Options |
|--------------------|--------------------|------------------|
| archived | completed | pending, processing, on-hold, completed, cancelled, refunded, failed |
| backorder | on-hold | pending, processing, on-hold, completed, cancelled, refunded, failed |
| complete | completed | pending, processing, on-hold, completed, cancelled, refunded, failed |
| completed | completed | pending, processing, on-hold, completed, cancelled, refunded, failed |
| new | completed | pending, processing, on-hold, completed, cancelled, refunded, failed |
| pending | pending | pending, processing, on-hold, completed, cancelled, refunded, failed |
| processing | processing | pending, processing, on-hold, completed, cancelled, refunded, failed |
| cancelled | cancelled | pending, processing, on-hold, completed, cancelled, refunded, failed |
| refunded | refunded | pending, processing, on-hold, completed, cancelled, refunded, failed |
| failed | failed | pending, processing, on-hold, completed, cancelled, refunded, failed |

For each status where the user wants a different mapping, ask which WooCommerce status to use. Only ask about statuses the user wants to change (not all 10).

Build the `--status-map` flag value as comma-separated pairs: `"archived:processing,new:on-hold,backorder:processing"`

Store as: `STATUS_MAP` (the full `--status-map=...` flag string, or empty if using defaults)

### Step 2.5: Configuration Summary

Display all collected configuration including auto-detected source site structure:

```
=== MIGRATION CONFIGURATION ===

Source Site: {SOURCE_SITE_URL}
Target Site: {siteurl} ({SITE_TYPE})
Theme: {THEME_SLUG}
Homepage: {HOMEPAGE_SLUG}
Product Variations: Standard variable products / Add-Ons ({ADDON_TYPE})
Order Forms: {ORDER_FORM_NAMES}
Default Country: {DEFAULT_COUNTRY}
Batch Sizes: Customers=100, Orders=50 (Products: no batch-size support)
Skip Images: {SKIP_IMAGES}
Order Status Mapping: Default / Custom ({STATUS_MAP})

Auto-Detected Site Structure:
  Header: {count} links
  Sidebar: {SOURCE_SIDEBAR_POSITION} with {count} nav items + {count} widgets
  Footer: {count} links + copyright
  Pages to create: {count}

Proceed with migration? (Continue to Phase 3)
```

Default batch sizes: `BATCH_CUSTOMER=100`, `BATCH_ORDER=50`. Product import does NOT support batch sizing.

---

## PHASE 3: Prerequisites Check & Migration Execution

**FROM THIS POINT FORWARD: Execute everything autonomously. No user prompts. No permission checks. Just run and report.**

### Step 3.0: Pre-flight Checks

Run ALL checks in a SINGLE command:

```bash
# Remote (one SSH call):
ssh {SSH_CMD} "echo '=== SITEURL ===' && wp option get siteurl --path={WP_PATH} && echo '=== PHP VERSION ===' && php -v | head -1 && echo '=== WP VERSION ===' && wp core version --path={WP_PATH} && echo '=== MEMORY ===' && wp eval 'echo WP_MEMORY_LIMIT;' --path={WP_PATH} && echo '=== CXI PLUGIN ===' && wp plugin list --name=customer-xml-importer --format=csv --path={WP_PATH} && echo '=== UPLOAD DIR ===' && mkdir -p {WP_PATH}/wp-content/uploads/upload-images && echo 'created' && echo '=== DEBUG ===' && grep -c 'WP_DEBUG' {WP_PATH}/wp-config.php"

# Local (one call):
echo '=== SITEURL ===' && wp option get siteurl --path="{WP_PATH}" && echo '=== PHP ===' && php -v | head -1 && echo '=== WP ===' && wp core version --path="{WP_PATH}" && echo '=== CXI ===' && wp plugin list --name=customer-xml-importer --format=csv --path="{WP_PATH}" && mkdir -p "{WP_PATH}/wp-content/uploads/upload-images"
```

If memory limit is not 2048M, add it to wp-config.php:
```bash
# Remote:
ssh {SSH_CMD} "sed -i '/<?php/a define(\"WP_MEMORY_LIMIT\", \"2048M\");\ndefine(\"WP_MAX_MEMORY_LIMIT\", \"2048M\");' {WP_PATH}/wp-config.php"
```

Report pre-flight results as a checklist. If any critical check fails, stop and help fix it.

### Step 3.1: Install WooCommerce, Theme & Reactivate CXI

**CRITICAL ORDER OF OPERATIONS**: WooCommerce MUST be installed and active BEFORE the CXI plugin is reactivated. The CXI plugin loads product/order importer classes only when `class_exists('WooCommerce')` is true. If CXI was activated before WooCommerce existed, its WC-dependent commands (`import-products`, `import-orders`) will NOT be registered.

Use `{THEME_SLUG}` from Phase 2 configuration (default: `blocksy`).

Run as a single chained command:
```bash
# Remote:
ssh {SSH_CMD} "wp plugin install woocommerce --activate --path={WP_PATH} && wp theme install {THEME_SLUG} --activate --path={WP_PATH} && wp plugin deactivate customer-xml-importer --path={WP_PATH} && wp plugin activate customer-xml-importer --path={WP_PATH} && echo '=== VERIFY ===' && wp plugin list --status=active --format=csv --path={WP_PATH}"

# Local:
wp plugin install woocommerce --activate --path="{WP_PATH}" && wp theme install {THEME_SLUG} --activate --path="{WP_PATH}" && wp plugin deactivate customer-xml-importer --path="{WP_PATH}" && wp plugin activate customer-xml-importer --path="{WP_PATH}"
```

If addon mode is enabled:
```bash
wp plugin install "{ADDON_PLUGIN_PATH}" --activate --path="{WP_PATH}"
```

### Step 3.2: Database Preparation

```bash
# Remote:
ssh {SSH_CMD} "wp cxi db-migrate --path={WP_PATH}"

# Local:
wp cxi db-migrate --path="{WP_PATH}"
```

This creates performance indexes on wp_postmeta. Report which indexes were created or skipped.

### Step 3.3: Customer Import

```bash
# Remote:
ssh {SSH_CMD} "wp cxi import '{DATA_DIR}/Customer.xml' '{DATA_DIR}/CustomerGroup.xml' '{DATA_DIR}/AccountGroup.xml' --batch-size={BATCH_CUSTOMER} --default-country={DEFAULT_COUNTRY} --skip-fields=none --path={WP_PATH}"

# Local:
wp cxi import "{DATA_DIR}/Customer.xml" "{DATA_DIR}/CustomerGroup.xml" "{DATA_DIR}/AccountGroup.xml" --batch-size={BATCH_CUSTOMER} --default-country={DEFAULT_COUNTRY} --skip-fields=none --path="{WP_PATH}"
```

**FLAGS**:
- `--skip-fields=none` → Non-interactive mode. Imports ALL fields without prompting.
- Do NOT use `--import-fields` (it triggers interactive readline prompts that hang).
- `--batch-size=100` → Process 100 customers per batch.

**IMPORTANT**: Use `timeout: 600000` (10 minutes). For very large datasets (>20,000), use `run_in_background: true`.

Report results: customers created, updated, errors.

### Step 3.4: Image Migration (BEFORE products)

**CRITICAL**: This step MUST happen BEFORE product import. The product importer looks for images in `wp-content/uploads/upload-images/` during import.

If `SKIP_IMAGES` is false:

For local sites:
```bash
find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) -exec cp {} "{WP_PATH}/wp-content/uploads/upload-images/" \; && echo "=== COUNT ===" && ls "{WP_PATH}/wp-content/uploads/upload-images/" | wc -l
```

For remote sites (files already on server):
```bash
ssh {SSH_CMD} 'find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) -exec cp {} "{WP_PATH}/wp-content/uploads/upload-images/" \; && echo "=== COUNT ===" && ls "{WP_PATH}/wp-content/uploads/upload-images/" | wc -l'
```

For remote sites (files need upload):
```bash
rsync -avz --include="*/" --include="*.jpg" --include="*.jpeg" --include="*.png" --include="*.gif" --include="*.webp" --include="*.bmp" --exclude="*" "{CLIENT_DIR}/" "{SSH_USER}@{SSH_HOST}:{WP_PATH}/wp-content/uploads/upload-images/"
```

Report number of images copied.

### Step 3.5: Product Import (AFTER images)

```bash
# Remote:
ssh {SSH_CMD} "wp cxi import-products '{DATA_DIR}/Product.xml' '{DATA_DIR}/Category.xml' '{DATA_DIR}/ProductCategory.xml' '{DATA_DIR}/Price.xml' --path={WP_PATH}"

# Local:
wp cxi import-products "{DATA_DIR}/Product.xml" "{DATA_DIR}/Category.xml" "{DATA_DIR}/ProductCategory.xml" "{DATA_DIR}/Price.xml" --path="{WP_PATH}"
```

**FLAGS**:
- `import-products` takes exactly 4 positional arguments: Product.xml, Category.xml, ProductCategory.xml, Price.xml
- It does NOT support `--batch-size`, `--import-fields`, or `--skip-fields`
- If addon mode: append `--addon --addon-type={ADDON_TYPE}` and optionally `--addon-required`

**IMPORTANT**: Use `timeout: 600000` (10 minutes). Use `run_in_background: true` for large datasets.

Report results: products created, variations created, categories, errors.

### Step 3.6: Page Import

```bash
# Remote:
ssh {SSH_CMD} "wp cxi import-pages '{DATA_DIR}/Page.xml' '{DATA_DIR}/PageElement.xml' --category-file='{DATA_DIR}/Category.xml' --product-file='{DATA_DIR}/Product.xml' --rewriteurl-file='{DATA_DIR}/RewriteURL.xml' --path={WP_PATH}"

# Local:
wp cxi import-pages "{DATA_DIR}/Page.xml" "{DATA_DIR}/PageElement.xml" --category-file="{DATA_DIR}/Category.xml" --product-file="{DATA_DIR}/Product.xml" --rewriteurl-file="{DATA_DIR}/RewriteURL.xml" --path="{WP_PATH}"
```

**FLAGS**: `import-pages` supports `--batch-size`, `--category-file`, `--product-file`, `--rewriteurl-file`, `--notification-email`, `--include-hidden`, `--skip-sidebar`.

**AUTO-GENERATED**: The importer automatically detects the background template (shared layout) and generates:
- **Sidebar widgets** in `sidebar-1` from LOCATION=1 elements (nav menus, testimonials, forms, category lists, cart, search)
- **Footer menu** from LOCATION=4 elements (assigned to the correct theme menu location)
- **Homepage title hiding** via Astra post meta (`site-post-title` = `disabled`)

Use `--skip-sidebar` flag on re-runs to skip sidebar/footer regeneration.

Report results: pages created, product blocks embedded, featured images set, sidebar widgets created, footer menu items.

### Step 3.7: Order Import

For remote sites, use `nohup` to prevent SSH disconnects from killing the import:

```bash
# Remote — use nohup to survive SSH drops:
ssh {SSH_CMD} "nohup wp cxi import-orders '{DATA_DIR}/AccountOrder.xml' --fast --form-names='{ORDER_FORM_NAMES}' --batch-size={BATCH_ORDER} {STATUS_MAP} --path={WP_PATH} > /tmp/order-import.log 2>&1 &" && echo "Order import started in background"

# Then monitor progress by tailing the log:
ssh {SSH_CMD} "tail -5 /tmp/order-import.log"

# Local:
wp cxi import-orders "{DATA_DIR}/AccountOrder.xml" --fast --form-names="{ORDER_FORM_NAMES}" --batch-size={BATCH_ORDER} {STATUS_MAP} --path="{WP_PATH}"
```

Where `{STATUS_MAP}` is either empty (for defaults) or `--status-map="archived:processing,new:on-hold"` (for custom mapping).

**FLAGS**:
- `import-orders` supports: `--batch-size`, `--form-names`, `--fast`, `--resume`, `--quiet`, `--import-id`, `--status-map`
- It does NOT support `--import-fields` or `--skip-fields`
- `--fast` → Direct DB inserts, disables WC hooks, suspends cache (much faster)
- `--resume` → Resume an interrupted import (only works if import is in-progress, not completed)

**IMPORTANT**: This is typically the longest-running import (can take hours for >10K orders). ALWAYS use `run_in_background: true` for the Bash tool. For remote sites, prefer the `nohup` approach above and poll the log file periodically.

**Monitoring remote nohup imports**: After starting the background import, poll every 60 seconds:
```bash
ssh {SSH_CMD} "tail -3 /tmp/order-import.log"
```
Look for "Import complete" or error messages. The import is done when the log shows final counts.

**If SSH drops during a non-nohup import**: The import likely died on the server. Check the order count:
```bash
ssh {SSH_CMD} "wp wc shop_order list --format=count --user=1 --path={WP_PATH}"
```
If the count matches the expected total, the import completed before the drop. If not, re-run the same command — the plugin skips already-imported orders (identified by `_xml_order_id` meta).

Report results: orders imported, skipped, errors.

### Step 3.8: Post-Import Cleanup

Run ALL cleanup in a SINGLE command:
```bash
# Remote:
ssh {SSH_CMD} "wp cache flush --path={WP_PATH} && wp rewrite flush --path={WP_PATH} && wp wc tool run recount_terms --user=1 --path={WP_PATH}"

# Local:
wp cache flush --path="{WP_PATH}" && wp rewrite flush --path="{WP_PATH}" && wp wc tool run recount_terms --user=1 --path="{WP_PATH}"
```

---

## PHASE 3.9: Site Configuration (Auto-Detected from Source)

**This entire phase runs autonomously using the structure detected in Phase 1.5.** All menus, sidebar, footer, widgets, and theme settings are configured to match the source site.

Use the **PHP Script Upload Pattern** (write PHP to `/tmp/`, scp, wp eval-file, rm) for all complex operations in this phase. This avoids SSH quoting issues with complex PHP arrays and strings.

### Step 3.9.1: Create WooCommerce Pages

Write a PHP script that creates standard WooCommerce pages if they don't exist:

```php
<?php
// Cart
$cart_id = wc_get_page_id('cart');
if ($cart_id <= 0 || !get_post($cart_id)) {
    $cart_id = wp_insert_post(array(
        'post_title' => 'Cart', 'post_name' => 'cart',
        'post_content' => '<!-- wp:woocommerce/cart --><!-- /wp:woocommerce/cart -->',
        'post_status' => 'publish', 'post_type' => 'page',
    ));
    update_option('woocommerce_cart_page_id', $cart_id);
}
// Repeat for: Checkout, My Account, Shop
// Also create any pages from PAGES_TO_CREATE (Site Map, Gift Card Balance, etc.)
```

Upload and execute via `wp eval-file`.

### Step 3.9.2: Set Homepage

```php
<?php
$home_page = get_page_by_path('{HOMEPAGE_SLUG}');
if (!$home_page) {
    $home_page = get_page_by_path('index');
}
if ($home_page) {
    update_option('show_on_front', 'page');
    update_option('page_on_front', $home_page->ID);
    echo "Homepage set to: {$home_page->post_title} (ID: {$home_page->ID})\n";
}
```

### Step 3.9.3: Configure Theme Settings

**For Blocksy theme:**
```php
<?php
// Sidebar position (auto-detected from source)
// CRITICAL: Blocksy uses 'type-2' for left, 'type-1' for right, 'type-4' for none
// WARNING: Do NOT use 'left-sidebar' — it does NOT work with Blocksy!
$sidebar_map = array('left' => 'type-2', 'right' => 'type-1', 'none' => 'type-4');
$structure = $sidebar_map['{SOURCE_SIDEBAR_POSITION}'];
set_theme_mod('single_page_structure', $structure);

// Disable page clutter
set_theme_mod('single_page_has_featured_image', 'no');
set_theme_mod('single_page_has_comments', 'no');
set_theme_mod('single_page_has_author_box', 'no');

// Header placements: logo+search top row, menu+cart bottom row
$header_placements = array(
    'current_section' => 'type-1',
    'sections' => array(array(
        'id' => 'type-1', 'mode' => 'placements',
        'items' => array(
            array('id' => 'logo', 'values' => array('logo_max_height' => 80)),
            array('id' => 'menu', 'values' => array()),
            array('id' => 'search', 'values' => array()),
            array('id' => 'cart', 'values' => array()),
            array('id' => 'trigger', 'values' => array()),
            array('id' => 'mobile-menu', 'values' => array()),
        ),
        'settings' => array(),
        'desktop' => array(
            array('id' => 'top-row', 'placements' => array(
                array('id' => 'start', 'items' => array()),
                array('id' => 'middle', 'items' => array()),
                array('id' => 'end', 'items' => array()),
                array('id' => 'start-middle', 'items' => array()),
                array('id' => 'end-middle', 'items' => array()),
            )),
            array('id' => 'middle-row', 'placements' => array(
                array('id' => 'start', 'items' => array('logo')),
                array('id' => 'middle', 'items' => array()),
                array('id' => 'end', 'items' => array('search')),
                array('id' => 'start-middle', 'items' => array()),
                array('id' => 'end-middle', 'items' => array()),
            )),
            array('id' => 'bottom-row', 'placements' => array(
                array('id' => 'start', 'items' => array('menu')),
                array('id' => 'middle', 'items' => array()),
                array('id' => 'end', 'items' => array('cart')),
                array('id' => 'start-middle', 'items' => array()),
                array('id' => 'end-middle', 'items' => array()),
            )),
            array('id' => 'offcanvas', 'placements' => array(
                array('id' => 'start', 'items' => array()),
            )),
        ),
        'mobile' => array(
            array('id' => 'top-row', 'placements' => array(
                array('id' => 'start', 'items' => array()),
                array('id' => 'middle', 'items' => array()),
                array('id' => 'end', 'items' => array()),
                array('id' => 'start-middle', 'items' => array()),
                array('id' => 'end-middle', 'items' => array()),
            )),
            array('id' => 'middle-row', 'placements' => array(
                array('id' => 'start', 'items' => array('logo')),
                array('id' => 'middle', 'items' => array()),
                array('id' => 'end', 'items' => array('trigger')),
                array('id' => 'start-middle', 'items' => array()),
                array('id' => 'end-middle', 'items' => array()),
            )),
            array('id' => 'bottom-row', 'placements' => array(
                array('id' => 'start', 'items' => array()),
                array('id' => 'middle', 'items' => array()),
                array('id' => 'end', 'items' => array()),
                array('id' => 'start-middle', 'items' => array()),
                array('id' => 'end-middle', 'items' => array()),
            )),
            array('id' => 'offcanvas', 'placements' => array(
                array('id' => 'start', 'items' => array('mobile-menu')),
            )),
        ),
    ))
);
set_theme_mod('header_placements', $header_placements);

// Footer placements: menu full-width + copyright
$footer_placements = array(
    'current_section' => 'type-1',
    'sections' => array(array(
        'id' => 'type-1', 'mode' => 'columns',
        'rows' => array(
            array('id' => 'top-row', 'columns' => array(array('menu'))),
            array('id' => 'middle-row', 'columns' => array(array('copyright'))),
            array('id' => 'bottom-row', 'columns' => array(array())),
        ),
        'items' => array(
            array('id' => 'copyright', 'values' => array(
                'copyright_text' => '{SOURCE_COPYRIGHT_TEXT}',
            )),
            array('id' => 'menu', 'values' => array()),
        ),
        'settings' => array(),
    ))
);
set_theme_mod('footer_placements', $footer_placements);
```

**For Astra theme:**
```php
<?php
$sidebar_map = array('left' => 'left-sidebar', 'right' => 'right-sidebar', 'none' => 'no-sidebar');
$astra_settings = get_option('astra-settings', array());
$astra_settings['site-sidebar-layout'] = $sidebar_map['{SOURCE_SIDEBAR_POSITION}'];
update_option('astra-settings', $astra_settings);
```

### Step 3.9.4: Build Header Menu (from SOURCE_HEADER_LINKS)

Write a PHP script that:
1. Gets or creates 'Primary Menu'
2. Deletes existing items
3. For each link in `SOURCE_HEADER_LINKS`:
   - If URL path matches an existing page slug → add as `post_type` menu item
   - If URL path matches a WooCommerce page (cart, checkout, my-account) → add as `post_type`
   - Otherwise → add as `custom` link with `home_url($path)`
4. Assigns menu to `primary` location via `set_theme_mod('nav_menu_locations', ...)`

```php
<?php
$primary_menu_name = 'Primary Menu';
$primary_menu = wp_get_nav_menu_object($primary_menu_name);
if (!$primary_menu) {
    $primary_menu_id = wp_create_nav_menu($primary_menu_name);
} else {
    $primary_menu_id = $primary_menu->term_id;
    $existing = wp_get_nav_menu_items($primary_menu_id);
    if ($existing) { foreach ($existing as $item) { wp_delete_post($item->ID, true); } }
}

// SOURCE_HEADER_LINKS populated from Phase 1.5.2
$header_items = array(
    // Each entry: array('title' => '...', 'slug' => '...', 'type' => 'page|custom', 'url' => '...')
    // Generated dynamically from SOURCE_HEADER_LINKS
);

$pos = 1;
foreach ($header_items as $item) {
    if ($item['type'] === 'custom') {
        wp_update_nav_menu_item($primary_menu_id, 0, array(
            'menu-item-title' => $item['title'],
            'menu-item-url' => $item['url'],
            'menu-item-type' => 'custom',
            'menu-item-status' => 'publish',
            'menu-item-position' => $pos++,
        ));
    } else {
        $page = get_page_by_path($item['slug']);
        if ($page) {
            wp_update_nav_menu_item($primary_menu_id, 0, array(
                'menu-item-title' => $item['title'],
                'menu-item-object' => 'page',
                'menu-item-object-id' => $page->ID,
                'menu-item-type' => 'post_type',
                'menu-item-status' => 'publish',
                'menu-item-position' => $pos++,
            ));
        }
    }
}

$locations = get_theme_mod('nav_menu_locations', array());
$locations['primary'] = $primary_menu_id;
set_theme_mod('nav_menu_locations', $locations);
```

### Step 3.9.5: Verify Sidebar Widgets (auto-generated by import-pages)

**NOTE**: The `import-pages` command now auto-generates sidebar widgets from the background template's LOCATION=1 elements. This step verifies they were created correctly and falls back to manual creation only if needed.

Verify sidebar widgets exist:
```bash
# Remote:
ssh {SSH_CMD} "wp widget list sidebar-1 --path={WP_PATH} 2>/dev/null || echo 'No widgets in sidebar-1'"
```

If sidebar-1 is empty (e.g., `--skip-sidebar` was used), fall back to manual creation.

**IMPORTANT**: For product category parents (Spring/Fall Planted Bulbs), the script should:
- Look up the category by name using `get_term_by('name', $name, 'product_cat')`
- Use uppercase title (e.g., "SPRING PLANTED BULBS") if the source site uses uppercase
- Fetch ALL child terms using `get_terms(array('taxonomy' => 'product_cat', 'parent' => $parent_id, 'hide_empty' => false))`
- Add each child with `'menu-item-parent-id' => $parent_item_id`

```php
<?php
$sidebar_menu_name = 'Sidebar Navigation';
$sidebar_menu = wp_get_nav_menu_object($sidebar_menu_name);
if (!$sidebar_menu) {
    $sidebar_menu_id = wp_create_nav_menu($sidebar_menu_name);
} else {
    $sidebar_menu_id = $sidebar_menu->term_id;
    $existing = wp_get_nav_menu_items($sidebar_menu_id);
    if ($existing) { foreach ($existing as $item) { wp_delete_post($item->ID, true); } }
}

$pos = 1;
// For each item in SOURCE_SIDEBAR_NAV:
// ... (dynamically generated from Phase 1.5 data)

// Create nav_menu widget
$nav_menu_widgets = get_option('widget_nav_menu', array());
$nav_menu_widgets[20] = array('title' => '', 'nav_menu' => $sidebar_menu_id);
$nav_menu_widgets['_multiwidget'] = 1;
update_option('widget_nav_menu', $nav_menu_widgets);

// Assign to sidebar-1 (will be extended in Step 3.9.6)
$sidebars = get_option('sidebars_widgets', array());
$sidebar_widgets = array('nav_menu-20');
// Widget IDs for testimonial/newsletter will be appended in Step 3.9.6
```

### Step 3.9.6: Verify/Add Sidebar Widgets (from SOURCE_SIDEBAR_WIDGETS)

**NOTE**: The `import-pages` command now auto-generates sidebar widgets (testimonials, forms, text, category lists, cart, search) from the background template. This step verifies correctness and adds any missing widgets.

For each widget detected in Phase 1.5.3, verify it exists in sidebar-1. If missing, create the appropriate widget:

**Testimonial widget:**
```php
$custom_html_widgets = get_option('widget_custom_html', array());
$custom_html_widgets[10] = array(
    'title' => 'Testimonial',
    'content' => '<div style="font-style: italic; padding: 15px; background: #f9f9f0; border-left: 3px solid #8bc34a; margin-bottom: 20px;">
<p style="font-size: 14px; line-height: 1.5; margin: 0 0 10px 0;">&ldquo;{TESTIMONIAL_QUOTE}&rdquo;</p>
<p style="font-size: 13px; color: #666; margin: 0; text-align: right;">&mdash; {TESTIMONIAL_ATTRIBUTION}</p>
</div>',
);
$custom_html_widgets['_multiwidget'] = 1;
update_option('widget_custom_html', $custom_html_widgets);
$sidebar_widgets[] = 'custom_html-10';
```

**Newsletter signup form widget:**
```php
$custom_html_widgets[11] = array(
    'title' => '{FORM_TITLE}',
    'content' => '<div style="padding: 15px; background: #f9f9f0; border: 1px solid #e0e0d0;">
<p style="font-size: 13px; margin: 0 0 10px 0;">{FORM_DESCRIPTION}</p>
<form method="post" action="{FORM_ACTION}" style="margin: 0;">
<!-- Generate fields from SOURCE_SIDEBAR_WIDGETS[].form_fields -->
<p style="margin: 0 0 10px 0;">
<label style="font-weight: bold; font-size: 13px; color: #c00;">First Name</label><br>
<input type="text" name="first_name" style="width: 100%; padding: 5px; border: 1px solid #ccc;">
</p>
<!-- ... repeat for each field ... -->
<p style="margin: 0; text-align: right;">
<button type="submit" style="background: #4CAF50; color: white; border: none; padding: 8px 20px; cursor: pointer; font-size: 13px;">{SUBMIT_LABEL}</button>
</p>
</form>
</div>',
);
update_option('widget_custom_html', $custom_html_widgets);
$sidebar_widgets[] = 'custom_html-11';
```

**Finalize sidebar widget assignment:**
```php
$sidebars['sidebar-1'] = $sidebar_widgets; // e.g., ['nav_menu-20', 'custom_html-10', 'custom_html-11']
// Clear footer widget areas (no Spring/Fall category lists in footer)
$sidebars['ct-footer-sidebar-1'] = array();
$sidebars['ct-footer-sidebar-2'] = array();
update_option('sidebars_widgets', $sidebars);
```

### Step 3.9.7: Build Footer Menu (from SOURCE_FOOTER_LINKS)

Same pattern as header menu but for footer:
1. Creates or clears 'Footer Menu'
2. Adds items from `SOURCE_FOOTER_LINKS`
3. Assigns to `footer` menu location

```php
<?php
$footer_menu_name = 'Footer Menu';
$footer_menu = wp_get_nav_menu_object($footer_menu_name);
if (!$footer_menu) {
    $footer_menu_id = wp_create_nav_menu($footer_menu_name);
} else {
    $footer_menu_id = $footer_menu->term_id;
    $existing = wp_get_nav_menu_items($footer_menu_id);
    if ($existing) { foreach ($existing as $item) { wp_delete_post($item->ID, true); } }
}

// Add items from SOURCE_FOOTER_LINKS
// ... (same page-lookup pattern as header menu)

$locations = get_theme_mod('nav_menu_locations', array());
// Astra uses 'footer_menu', Blocksy uses 'footer'
$footer_location_key = (wp_get_theme()->get_template() === 'astra') ? 'footer_menu' : 'footer';
$locations[$footer_location_key] = $footer_menu_id;
set_theme_mod('nav_menu_locations', $locations);
```

### Step 3.9.8: Final Cache Flush

```bash
# Remote:
ssh {SSH_CMD} "wp cache flush --path={WP_PATH} && wp rewrite flush --path={WP_PATH}"

# Local:
wp cache flush --path="{WP_PATH}" && wp rewrite flush --path="{WP_PATH}"
```

### Step 3.9.9: Report Site Configuration Results

```
=== SITE CONFIGURATION COMPLETE ===

Homepage: {page_title} (ID: {page_id})
Theme: {THEME_SLUG} with {SOURCE_SIDEBAR_POSITION} sidebar

Header Menu ({count} items):
  {item1} | {item2} | {item3} | ...

Sidebar Navigation ({top_level} top-level, {children} children, {total} total):
  Home, SPRING PLANTED BULBS ({N} sub), FALL PLANTED BULBS ({N} sub), ...
Sidebar Widgets:
  - Testimonial: "{quote_preview}..."
  - Newsletter Form: {field_count} fields

Footer Menu ({count} items):
  {item1} · {item2} · {item3} · ...
Copyright: "{copyright_preview}..."

Pages Created: {count} (Cart, Checkout, My Account, Shop, ...)
```

---

## PHASE 4: Verification

### Step 4.1: Data Count Verification

Run ALL counts in a SINGLE command:

```bash
# Remote:
ssh {SSH_CMD} "echo '=== CUSTOMERS ===' && wp user list --role=customer --format=count --path={WP_PATH} && echo '=== PRODUCTS ===' && wp post list --post_type=product --post_status=any --format=count --path={WP_PATH} && echo '=== VARIATIONS ===' && wp post list --post_type=product_variation --post_status=any --format=count --path={WP_PATH} && echo '=== CATEGORIES ===' && wp term list product_cat --format=count --path={WP_PATH} && echo '=== PAGES ===' && wp post list --post_type=page --post_status=any --format=count --path={WP_PATH} && echo '=== ORDERS ===' && wp wc shop_order list --format=count --user=1 --path={WP_PATH} && echo '=== MEDIA ===' && wp post list --post_type=attachment --format=count --path={WP_PATH} && echo '=== PRODUCTS WITH IMAGES ===' && wp eval 'global \$wpdb; echo \$wpdb->get_var(\"SELECT COUNT(DISTINCT post_id) FROM {\$wpdb->postmeta} WHERE meta_key = \\\"_thumbnail_id\\\" AND post_id IN (SELECT ID FROM {\$wpdb->posts} WHERE post_type = \\\"product\\\")\");' --path={WP_PATH}"

# Local (same commands without ssh prefix, with quoted --path):
echo '=== CUSTOMERS ===' && wp user list --role=customer --format=count --path="{WP_PATH}" && echo '=== PRODUCTS ===' && wp post list --post_type=product --post_status=any --format=count --path="{WP_PATH}" && echo '=== VARIATIONS ===' && wp post list --post_type=product_variation --post_status=any --format=count --path="{WP_PATH}" && echo '=== CATEGORIES ===' && wp term list product_cat --format=count --path="{WP_PATH}" && echo '=== PAGES ===' && wp post list --post_type=page --post_status=any --format=count --path="{WP_PATH}" && echo '=== ORDERS ===' && wp wc shop_order list --format=count --user=1 --path="{WP_PATH}" && echo '=== MEDIA ===' && wp post list --post_type=attachment --format=count --path="{WP_PATH}" && echo '=== PRODUCTS WITH IMAGES ===' && wp eval 'global $wpdb; echo $wpdb->get_var("SELECT COUNT(DISTINCT post_id) FROM {$wpdb->postmeta} WHERE meta_key = \"_thumbnail_id\" AND post_id IN (SELECT ID FROM {$wpdb->posts} WHERE post_type = \"product\")");' --path="{WP_PATH}"
```

Also get environment details in a SINGLE command:
```bash
# Remote:
ssh {SSH_CMD} "echo '=== WP VERSION ===' && wp core version --path={WP_PATH} && echo '=== WC VERSION ===' && wp plugin get woocommerce --field=version --path={WP_PATH} && echo '=== PHP VERSION ===' && php -v | head -1 && echo '=== THEME ===' && wp theme list --status=active --format=csv --path={WP_PATH} && echo '=== CXI VERSION ===' && wp plugin get customer-xml-importer --field=version --path={WP_PATH}"
```

Present verification as comparison table:

```
=== DATA VERIFICATION ===

| Data Type          | Source XML | Imported | Match |
|--------------------|------------|----------|-------|
| Customers          | {source}   | {count}  | {%}   |
| Products           | {source}   | {count}  | {%}   |
| Product Variations | -          | {count}  | -     |
| Categories         | {source}   | {count}  | {%}   |
| Pages              | {source}   | {count}  | {%}   |
| Orders             | {source}   | {count}  | {%}   |
| Products w/ Images | -          | {x}/{y}  | {%}   |
```

Flag any significant mismatches (>5% difference) for investigation.

### Step 4.2: Deep Visual Verification

Use the Playwright MCP browser tools to perform a comprehensive 10-step visual verification of the migrated site. For each step, navigate to the page, take a screenshot (saved to `/tmp/verify-*.png`), and check the accessibility snapshot.

**If `SOURCE_SITE_URL` is available**, also take matching screenshots of the source site for side-by-side comparison.

#### Check 1: Homepage Comparison
- Navigate to migrated homepage at 1400x900 viewport
- Screenshot: `/tmp/verify-homepage-migrated.png`
- If source URL available, screenshot source too: `/tmp/verify-homepage-source.png`
- Verify: page title is NOT showing (should be hidden), main content renders, product blocks display

#### Check 2: Sidebar Widget Check
- On the homepage, check the accessibility snapshot for sidebar presence
- Verify sidebar contains: navigation menu, testimonial widget, newsletter/form widget
- Count total sidebar widgets and compare to source site sidebar element count

#### Check 3: Footer Check
- Scroll to footer section on homepage
- Verify footer menu links are present and match source site links
- Check copyright text is present

#### Check 4: Random Product Pages (3 pages)
- Query 3 random product category pages: `wp post list --post_type=page --post_status=publish --format=csv --fields=ID,post_title,post_name --path={WP_PATH}`
- Pick 3 pages with product content (look for pages that have `[products` shortcode)
- Navigate to each page and verify: product grid renders, correct columns, images present
- Compare product count on page with XML's `numtodisplay` value for that category

#### Check 5: Page Title Visibility
- Navigate to homepage — verify the "Home" title is NOT displayed (Astra `site-post-title` meta)
- Navigate to 2 other pages — verify their titles ARE displayed

#### Check 6: Header Navigation
- Take accessibility snapshot of header area
- Click 3 menu links and verify they navigate to correct pages
- After each click, verify the destination page loads and contains expected content

#### Check 7: Product Category Pages
- Navigate to 2 WooCommerce product category archive pages
- Verify product thumbnails are loading (not broken images)
- Verify product count is reasonable (matches import data)

#### Check 8: Contact Info
- Find a page with a contact info block (search in accessibility snapshot for "Phone" or "Email")
- Verify phone number and email address are correct (not hardcoded placeholder values)

#### Check 9: WooCommerce Cart/Shop
- Navigate to /shop/ page — verify it loads
- Check if mini-cart or cart widget appears in sidebar

#### Check 10: Compile Diff Report
- For each check above, record PASS/FAIL with details
- If any FAIL, attempt auto-fix using PHP eval-file scripts uploaded to the server:
  - Missing sidebar widgets → re-run `wp cxi import-pages` with sidebar generation
  - Wrong menu location → fix via `set_theme_mod('nav_menu_locations', ...)`
  - Page title showing on homepage → `update_post_meta($homepage_id, 'site-post-title', 'disabled')`
- Re-verify after any auto-fix

Report results as a structured table:

```
=== DEEP VISUAL VERIFICATION ===

| #  | Check                    | Expected              | Actual                | Status |
|----|--------------------------|----------------------|----------------------|--------|
| 1  | Homepage renders         | Content visible       | {result}             | PASS/FAIL |
| 2  | Sidebar widgets          | {N} widgets          | {N} found            | PASS/FAIL |
| 3  | Footer menu              | {N} links + copyright | {result}             | PASS/FAIL |
| 4  | Product page: {name1}    | {N} products, grid   | {result}             | PASS/FAIL |
| 4  | Product page: {name2}    | {N} products, grid   | {result}             | PASS/FAIL |
| 4  | Product page: {name3}    | {N} products, grid   | {result}             | PASS/FAIL |
| 5  | Homepage title hidden    | Hidden               | {result}             | PASS/FAIL |
| 6  | Header nav works         | 3 links work         | {result}             | PASS/FAIL |
| 7  | Category archives        | Products load        | {result}             | PASS/FAIL |
| 8  | Contact info             | Correct phone/email  | {result}             | PASS/FAIL |
| 9  | WooCommerce shop         | Shop page loads      | {result}             | PASS/FAIL |
| 10 | Auto-fixes applied       | N/A                  | {fixes_count} fixes  | INFO |
```

**IMPORTANT**: Do NOT stop the migration for FAIL results. Report all issues, apply auto-fixes where possible, and continue to Phase 5. Document unresolved issues in the final report.

---

## PHASE 5: Report Generation

Generate three documentation files in the client-brief directory:

### 5.1: MIGRATION_REPORT.md

Create a comprehensive report with:
- Migration summary table (all phases with status)
- Data counts table (source vs imported)
- Site configuration summary (menus, sidebar, footer, theme)
- Issues encountered and how they were resolved
- Plugin modifications made (if any)
- Environment details (WordPress version, WooCommerce version, PHP version)
- Configuration used
- Remaining tasks checklist

### 5.2: MIGRATION_DATA_REQUIREMENTS.md

Document all required XML files with:
- File name, purpose, key fields
- Expected folder structure for images
- Supported image formats
- XML format description (`<table><row>` structure with CDATA parameters)

### 5.3: MIGRATION_PERMISSIONS.md

Document all permissions required:
- Server/environment requirements (PHP, MySQL, WordPress versions)
- Required WordPress capabilities
- File system access requirements
- WP-CLI database connection details
- Order form name types reference

### 5.4: Final Summary

Present the migration completion summary to the user:

```
=== MIGRATION COMPLETE ===

Site: {siteurl}
Source: {SOURCE_SITE_URL}
Theme: {THEME_SLUG}

Data Imported:
  Customers: {count}
  Products: {count} ({variations} variations)
  Categories: {count}
  Pages: {count}
  Orders: {count}
  Images: {count}

Site Configuration:
  Homepage: {page_title}
  Header Menu: {count} items
  Sidebar: {position} with {nav_count} nav items + {widget_count} widgets
  Footer: {count} links + copyright

Reports Generated:
  - MIGRATION_REPORT.md
  - MIGRATION_DATA_REQUIREMENTS.md
  - MIGRATION_PERMISSIONS.md

Remaining Tasks:
  - [ ] Verify sample products on frontend
  - [ ] Test checkout flow
  - [ ] Configure payment gateway
  - [ ] Set up email notifications
  - [ ] Configure shipping methods
  - [ ] Review products missing images
  - [ ] Test URL redirects from old site
  - [ ] Upload site logo
  - [ ] Configure color scheme to match source site
```

---

## IMPORTANT NOTES — COMMAND FLAG REFERENCE

### WP-CLI Argument Syntax
The `customer-xml-importer` plugin uses **positional arguments**, NOT named flags for file paths:
```bash
# CORRECT:
wp cxi import Customer.xml CustomerGroup.xml AccountGroup.xml
# WRONG:
wp cxi import --file=Customer.xml --groups=CustomerGroup.xml
```

### Flag Support Per Subcommand

Each `wp cxi` subcommand supports DIFFERENT flags. Do NOT assume a flag works across all subcommands.

| Flag | `import` (customers) | `import-products` | `import-pages` | `import-orders` |
|------|---------------------|-------------------|----------------|-----------------|
| `--batch-size` | YES | NO | YES | YES |
| `--skip-fields=none` | YES (use this!) | NO | NO | NO |
| `--import-fields` | BROKEN (hangs) | NO | NO | NO |
| `--default-country` | YES | NO | NO | NO |
| `--fast` | NO | NO | NO | YES |
| `--form-names` | NO | NO | NO | YES |
| `--resume` | NO | NO | NO | YES |
| `--quiet` | NO | NO | NO | YES |
| `--status-map` | NO | NO | NO | YES |
| `--addon` | NO | YES | NO | NO |
| `--addon-type` | NO | YES | NO | NO |
| `--addon-required` | NO | YES | NO | NO |
| `--category-file` | NO | NO | YES | NO |
| `--product-file` | NO | NO | YES | NO |
| `--rewriteurl-file` | NO | NO | YES | NO |
| `--notification-email` | NO | NO | YES | YES |
| `--path` | YES (always!) | YES (always!) | YES (always!) | YES (always!) |

### Non-Interactive Mode
- **Customer import**: Use `--skip-fields=none` to import all fields without interactive prompts.
- **Product import**: No field flags needed — always imports all fields automatically.
- **Order import**: No field flags needed — always imports all fields automatically.
- **NEVER use `--import-fields` without a value** — it triggers interactive readline() prompts that hang Claude.

### WooCommerce Plugin Activation Order
WooCommerce MUST be active BEFORE the CXI plugin is loaded. After installing WooCommerce, always deactivate then reactivate CXI:
```bash
wp plugin install woocommerce --activate && wp plugin deactivate customer-xml-importer && wp plugin activate customer-xml-importer
```

### SSH Keepalive for Remote Servers
ALWAYS use these SSH flags to prevent broken pipe disconnects during long imports:
```
-o ServerAliveInterval=60 -o ServerAliveCountMax=5
```
This sends a keepalive packet every 60 seconds and tolerates up to 5 missed responses.

### Long-Running Imports on Remote Servers
For imports expected to take >30 minutes (typically order imports with >5,000 records), use `nohup` to ensure the import survives SSH disconnects:
```bash
ssh {SSH_CMD} "nohup wp cxi import-orders ... > /tmp/order-import.log 2>&1 &"
```
Then monitor by polling the log file.

### Timeouts
Customer, product, and order imports can take 10+ minutes for large datasets. Always use `timeout: 600000` (10 minutes) when running these commands with the Bash tool. Use `run_in_background: true` for imports expected to take longer.

### Image Timing
Images MUST be copied to `wp-content/uploads/upload-images/` BEFORE running product import. The product importer checks for local image files during import and assigns them to products.

### Error Recovery
- The plugin supports resume capability. If an import crashes, re-running the same command will skip already-imported records (identified by `_xml_product_id`, `_xml_customer_id`, or `_xml_order_id` meta).
- The `--resume` flag on `import-orders` only works for imports that are in-progress (not completed). If an import completed but SSH dropped, `--resume` will say "already completed" — verify the count instead.
- Invalid billing emails during order import have been fixed with `sanitize_email()` and try-catch in `class-cxi-order-importer.php`.

### WooCommerce CLI Commands
Some WooCommerce CLI commands require `--user=1` to authenticate:
- `wp wc shop_order list --user=1`
- `wp wc tool run recount_terms --user=1`
Always include `--user=1` on `wp wc` commands.

---

## THEME CONFIGURATION REFERENCE

### Blocksy Theme (Recommended)

**Sidebar settings:**
| Desired Position | Theme Mod Value | WARNING |
|-----------------|-----------------|---------|
| Left sidebar | `type-2` | Do NOT use `left-sidebar` — it will NOT work! |
| Right sidebar | `type-1` | |
| No sidebar | `type-4` | This is the default |
| Narrow (no sidebar) | `type-3` | |

```php
set_theme_mod('single_page_structure', 'type-2'); // LEFT sidebar
```

**How Blocksy determines sidebar position** (from `inc/sidebar.php`):
1. Checks per-post meta `page_structure_type` first
2. Falls back to `blocksy_get_theme_mod($prefix . '_structure', $default_structure)`
3. Maps: `type-2` → `left`, `type-1` → `right`, anything else → `none`
4. For `single_page` prefix, default is `type-4` (no sidebar)

**Header placements** (`set_theme_mod('header_placements', ...)`):
- Structure: `sections[0].desktop` = array of rows (top-row, middle-row, bottom-row, offcanvas)
- Each row has `placements` array with `{id: 'start'|'middle'|'end', items: [...]}`
- Available items: `logo`, `menu`, `search`, `cart`, `trigger`, `mobile-menu`

**Footer placements** (`set_theme_mod('footer_placements', ...)`):
- Structure: `sections[0].rows` = array of rows, each with `columns` array
- `columns` = array of arrays, each inner array has item ID strings
- Available items: `menu`, `copyright`, `widget-area-1`, `widget-area-2`

**Widget areas:**
- `sidebar-1` — Page sidebar (left or right)
- `ct-footer-sidebar-1` — Footer widget area 1
- `ct-footer-sidebar-2` — Footer widget area 2

**Page feature toggles:**
```php
set_theme_mod('single_page_has_featured_image', 'no');
set_theme_mod('single_page_has_comments', 'no');
set_theme_mod('single_page_has_author_box', 'no');
```

**Database class caching** (`inc/classes/database.php`):
- Blocksy caches theme mods in `$this->mods` property
- On admin/ajax requests, it re-reads from `get_theme_mods()` every call
- On frontend, it caches after first read — WP-CLI `wp eval` may see stale values
- Fresh `wp eval-file` invocations always get fresh values (new PHP process)

### Astra Theme

**Sidebar settings:**
```php
$astra_settings = get_option('astra-settings', array());
$astra_settings['site-sidebar-layout'] = 'left-sidebar'; // or 'right-sidebar', 'no-sidebar'
update_option('astra-settings', $astra_settings);
```

**Header/Footer:** Configured via customizer settings in `astra-settings` option array.

### WordPress Nav Menu Widget Pattern

To assign a navigation menu to a sidebar widget area:
```php
// 1. Create nav_menu widget instance
$nav_menu_widgets = get_option('widget_nav_menu', array());
$nav_menu_widgets[20] = array('title' => '', 'nav_menu' => $menu_id);
$nav_menu_widgets['_multiwidget'] = 1;
update_option('widget_nav_menu', $nav_menu_widgets);

// 2. Create custom_html widget instances for testimonials/forms
$custom_html_widgets = get_option('widget_custom_html', array());
$custom_html_widgets[10] = array('title' => 'Testimonial', 'content' => '...');
$custom_html_widgets[11] = array('title' => 'Newsletter', 'content' => '...');
$custom_html_widgets['_multiwidget'] = 1;
update_option('widget_custom_html', $custom_html_widgets);

// 3. Assign all widgets to sidebar in order
$sidebars = get_option('sidebars_widgets', array());
$sidebars['sidebar-1'] = array('nav_menu-20', 'custom_html-10', 'custom_html-11');
update_option('sidebars_widgets', $sidebars);
```

**IMPORTANT**: `wp_update_nav_menu_item()` with only `menu-item-position` resets all other fields (title, parent, type). When reordering menu items, you must delete all items and recreate from scratch rather than updating positions individually.
