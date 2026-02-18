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

### Step 0.1: Site Type, Source URL & Data Path

Use the `AskUserQuestion` tool to ask THREE questions:

**Question 1**: "Where is the target WordPress site running?"
- **Local (Local by Flywheel)** (Recommended for development) — WordPress is running locally via Local by Flywheel app
- **Remote server (SSH)** — WordPress is on a remote server accessible via SSH

**Question 2**: "What is the source (original) StoresOnline site URL?"
- User provides the full URL (e.g., `https://www.marysgardenpatch.com/`)
- This is used later for visual comparison and header menu detection

**Question 3**: "What is the path to the client data directory containing XML files?"
- For local sites, default suggestion: `wp-content/client-brief/{client-name}/data/`
- For remote sites: ask if files are already on server or need to be uploaded
  - If already on server: full remote path (e.g., `/sites/example.com/data`)
  - If need upload: local path to the data directory

Store as: `SOURCE_SITE_URL`, `DATA_DIR`

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

Set these variables:
- `SSH_CMD`: `ssh -p {PORT} -i {KEYFILE} -o ServerAliveInterval=60 -o ServerAliveCountMax=5 {USER}@{HOST}` (ALWAYS include keepalive)
- `WP_PATH`: Remote WordPress path
- `SITE_TYPE`: `remote`
- `NEEDS_UPLOAD`: `true` or `false`

### Step 0.3: Data Scan & Scope of Work

Now that the connection is verified and `DATA_DIR` is set (from Step 0.1 Question 3), scan the data directory to determine the scope of work. Run as a SINGLE command:

```bash
# For local:
for f in Customer.xml CustomerGroup.xml AccountGroup.xml Product.xml Category.xml ProductCategory.xml Price.xml RewriteURL.xml Page.xml PageElement.xml AccountOrder.xml Form.xml FormElement.xml; do
  if [ -f "{DATA_DIR}/$f" ]; then
    COUNT=$(grep -c '<row>' "{DATA_DIR}/$f" 2>/dev/null || echo "0")
    echo "FOUND: $f ($COUNT records)"
  else
    echo "MISSING: $f"
  fi
done

# For remote (wrap entire loop in single SSH call):
ssh {SSH_CMD} 'for f in Customer.xml CustomerGroup.xml AccountGroup.xml Product.xml Category.xml ProductCategory.xml Price.xml RewriteURL.xml Page.xml PageElement.xml AccountOrder.xml Form.xml FormElement.xml; do if [ -f "{DATA_DIR}/$f" ]; then COUNT=$(grep -c "<row>" "{DATA_DIR}/$f" 2>/dev/null || echo "0"); echo "FOUND: $f ($COUNT records)"; else echo "MISSING: $f"; fi; done'
```

Present results as a table and SOW summary:

| File | Status | Records |
|------|--------|---------|
| Customer.xml | Found | 11,148 |
| ... | ... | ... |

```
=== SCOPE OF WORK ===

Based on data files found, this migration includes:
- Customers: {count} records
- Products: {count} records (with {category_count} categories, {price_count} price entries)
- Pages: {count} records
- Orders: {count} records
- Forms: {Yes/No} ({form_count} form definitions)
- URL Rewrites: {Yes/No} ({rewrite_count} entries)
- Images: (will be scanned in Phase 1)
```

Flag any MISSING required files. The minimum required files are:
- **Customer import**: Customer.xml, CustomerGroup.xml, AccountGroup.xml
- **Product import**: Product.xml, Category.xml, ProductCategory.xml, Price.xml
- **Page import**: Page.xml, PageElement.xml
- **Page import (optional)**: Form.xml, FormElement.xml (for WPForms creation), ProductCategory.xml (for featured products)
- **Order import**: AccountOrder.xml

### Step 0.4: Check Required Plugins

Verify that both required custom plugins are installed on the target site. Run as a SINGLE command:

```bash
# Remote:
ssh {SSH_CMD} "echo '=== customer-xml-importer ===' && wp plugin list --name=customer-xml-importer --format=csv --path={WP_PATH} && echo '=== cxi-site-customizations ===' && wp plugin list --name=cxi-site-customizations --format=csv --path={WP_PATH}"

# Local:
echo '=== customer-xml-importer ===' && wp plugin list --name=customer-xml-importer --format=csv --path="{WP_PATH}" && echo '=== cxi-site-customizations ===' && wp plugin list --name=cxi-site-customizations --format=csv --path="{WP_PATH}"
```

If either plugin is missing, inform the user:

```
The following required plugins are not installed on the target site.
Please add them to: {WP_PATH}/wp-content/plugins/

Missing plugins:
  - customer-xml-importer — WP-CLI migration engine (handles customer, product, page, order imports)
  - cxi-site-customizations — Runtime companion plugin (cart widget, category URL rewrites, theme compatibility)
```

For remote sites, offer to SCP the plugins from the local machine if they exist:
```bash
scp -r -P {PORT} -i {KEYFILE} "{LOCAL_WP_PATH}/wp-content/plugins/customer-xml-importer" {USER}@{HOST}:{WP_PATH}/wp-content/plugins/
scp -r -P {PORT} -i {KEYFILE} "{LOCAL_WP_PATH}/wp-content/plugins/cxi-site-customizations" {USER}@{HOST}:{WP_PATH}/wp-content/plugins/
```

Wait for user confirmation that plugins are available before proceeding.

### Step 0.5: SOW-Based Configuration Questions

Based on the data scan results from Step 0.3, ask relevant questions using `AskUserQuestion`:

- **If Form.xml + FormElement.xml found**: "Should StoresOnline forms be migrated to WPForms? (Requires WPForms Lite plugin — free)"
- **If AccountOrder.xml has >5,000 records**: Note: "Large order volume detected ({count} orders). Will use `nohup` + `--fast` mode to prevent timeouts."
- **If any extra SOW items are identified** (e.g., custom post types, special integrations): Ask questions to clarify scope

Store decisions as: `MIGRATE_FORMS` (true/false)

---

## PHASE 1: Image & Order Analysis

### Step 1.1: Scan for Image Folders

Look for image folders and count images in a SINGLE command:
```bash
# Local:
echo "=== FOLDERS ===" && find "{CLIENT_DIR}" -maxdepth 1 -type d | sort && echo "=== IMAGE COUNT ===" && find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) | wc -l

# Remote (single SSH call):
ssh {SSH_CMD} 'echo "=== FOLDERS ===" && find "{CLIENT_DIR}" -maxdepth 1 -type d | sort && echo "=== IMAGE COUNT ===" && find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) | wc -l'
```

Report the number of image folders and total image count.

### Step 1.2: Analyze Orders

If `AccountOrder.xml` exists, run BOTH commands in a single SSH call:
```bash
# Remote:
ssh {SSH_CMD} "wp cxi list-order-form-names '{DATA_DIR}/AccountOrder.xml' --path={WP_PATH} && wp cxi validate-orders '{DATA_DIR}/AccountOrder.xml' --path={WP_PATH}"

# Local:
wp cxi list-order-form-names "{DATA_DIR}/AccountOrder.xml" --path="{WP_PATH}" && wp cxi validate-orders "{DATA_DIR}/AccountOrder.xml" --path="{WP_PATH}"
```

Report form types with counts and validation results.

### Step 1.3: Present Summary

Display a comprehensive summary table combining the data scan from Phase 0 Step 0.3 with image counts from Step 1.1 and order analysis from Step 1.2.

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
- **Other free theme (WordPress.org)** — Specify theme slug (e.g., `flavor`, `flavor`)
- **Other premium theme (upload)** — User provides theme zip path on server

Store as: `THEME_SLUG` (e.g., `blocksy`, `astra`, `flavor`) and `THEME_TYPE` (`free-builtin`, `free-wporg`, `premium-upload`)

### Step 2.1.5: Theme Details (if "Other" selected)

**If "Other free theme" selected:**
1. Ask for the theme slug
2. Verify it exists on WordPress.org:
   ```bash
   # Remote:
   ssh {SSH_CMD} "wp theme search {THEME_SLUG} --fields=name,slug --format=table --path={WP_PATH}"
   ```
3. If not found, ask the user to re-enter or choose a different theme

**If "Other premium theme" selected:**
1. Ask the user to upload the theme zip file to the server
2. Ask for the full path to the zip file on the server (e.g., `/tmp/mytheme.zip`)
3. Store as: `THEME_ZIP_PATH`

**For non-Blocksy/Astra themes:**
- `CXI_Theme_Compat::get_theme_config()` returns generic defaults (no sidebar position control, basic footer menu location)
- Note in config summary: "Theme '{THEME_SLUG}' uses generic configuration. Manual Customizer setup may be needed for sidebar position and header/footer layout after migration."

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

Display all collected configuration:

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
Migrate Forms: {MIGRATE_FORMS}
Order Status Mapping: Default / Custom ({STATUS_MAP})

Data Scope (from Phase 0):
  Customers: {count} | Products: {count} | Pages: {count}
  Orders: {count} | Forms: {count} | Images: {count}

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

Use `{THEME_SLUG}` and `{THEME_TYPE}` from Phase 2 configuration (default: `blocksy`).

**Theme install command depends on theme type:**
- `free-builtin` (Blocksy/Astra): `wp theme install {THEME_SLUG} --activate`
- `free-wporg` (Other free): `wp theme install {THEME_SLUG} --activate`
- `premium-upload`: `wp theme install {THEME_ZIP_PATH} --activate`

Run as a single chained command:
```bash
# Remote (free themes):
ssh {SSH_CMD} "wp plugin install woocommerce --activate --path={WP_PATH} && wp theme install {THEME_SLUG} --activate --path={WP_PATH} && wp plugin deactivate customer-xml-importer --path={WP_PATH} && wp plugin activate customer-xml-importer --path={WP_PATH} && echo '=== VERIFY ===' && wp plugin list --status=active --format=csv --path={WP_PATH}"

# Remote (premium theme from zip):
ssh {SSH_CMD} "wp plugin install woocommerce --activate --path={WP_PATH} && wp theme install {THEME_ZIP_PATH} --activate --path={WP_PATH} && wp plugin deactivate customer-xml-importer --path={WP_PATH} && wp plugin activate customer-xml-importer --path={WP_PATH} && echo '=== VERIFY ===' && wp plugin list --status=active --format=csv --path={WP_PATH}"

# Local:
wp plugin install woocommerce --activate --path="{WP_PATH}" && wp theme install {THEME_SLUG_OR_ZIP} --activate --path="{WP_PATH}" && wp plugin deactivate customer-xml-importer --path="{WP_PATH}" && wp plugin activate customer-xml-importer --path="{WP_PATH}"
```

If addon mode is enabled:
```bash
wp plugin install "{ADDON_PLUGIN_PATH}" --activate --path="{WP_PATH}"
```

### Step 3.1.5: Install WPForms & Site Customizations Plugin

Install WPForms Lite (for form migration) and the cxi-site-customizations plugin (for cart widget, URL rewrites, theme compatibility):

```bash
# Remote:
ssh {SSH_CMD} "wp plugin install wpforms-lite --activate --path={WP_PATH} && wp plugin activate cxi-site-customizations --path={WP_PATH}"

# Local:
wp plugin install wpforms-lite --activate --path="{WP_PATH}" && wp plugin activate cxi-site-customizations --path="{WP_PATH}"
```

**NOTE**: The `cxi-site-customizations` plugin must be uploaded to the remote server first if not already present. Use SCP:
```bash
scp -r "{LOCAL_WP_PATH}/wp-content/plugins/cxi-site-customizations" {SSH_USER}@{SSH_HOST}:{WP_PATH}/wp-content/plugins/
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

### Step 3.5.5: Mark Featured Products

**NOTE**: The `import-pages` command (Step 3.6) now auto-marks featured products when it encounters `showfeatured=true` elements in PageElement.xml, using the `--productcategory-file` flag. This step is only needed if you want to pre-mark featured products before page import, or if running product import without page import.

If you need to manually mark featured products for specific categories:

```bash
# Remote (for each category):
ssh {SSH_CMD} "wp cxi mark-featured '{DATA_DIR}/ProductCategory.xml' --category={CATEGORY_ID} --limit={PRODUCT_COUNT} --path={WP_PATH}"

# Local:
wp cxi mark-featured "{DATA_DIR}/ProductCategory.xml" --category={CATEGORY_ID} --limit={PRODUCT_COUNT} --path="{WP_PATH}"
```

This uses ProductCategory.xml POSITION ordering to determine which products are featured. In most migrations, this step can be skipped since import-pages handles it automatically.

### Step 3.6: Page Import

```bash
# Remote:
ssh {SSH_CMD} "wp cxi import-pages '{DATA_DIR}/Page.xml' '{DATA_DIR}/PageElement.xml' --category-file='{DATA_DIR}/Category.xml' --product-file='{DATA_DIR}/Product.xml' --rewriteurl-file='{DATA_DIR}/RewriteURL.xml' --form-file='{DATA_DIR}/Form.xml' --formelement-file='{DATA_DIR}/FormElement.xml' --productcategory-file='{DATA_DIR}/ProductCategory.xml' --path={WP_PATH}"

# Local:
wp cxi import-pages "{DATA_DIR}/Page.xml" "{DATA_DIR}/PageElement.xml" --category-file="{DATA_DIR}/Category.xml" --product-file="{DATA_DIR}/Product.xml" --rewriteurl-file="{DATA_DIR}/RewriteURL.xml" --form-file="{DATA_DIR}/Form.xml" --formelement-file="{DATA_DIR}/FormElement.xml" --productcategory-file="{DATA_DIR}/ProductCategory.xml" --path="{WP_PATH}"
```

**FLAGS**: `import-pages` supports `--batch-size`, `--category-file`, `--product-file`, `--rewriteurl-file`, `--form-file`, `--formelement-file`, `--productcategory-file`, `--notification-email`, `--include-hidden`, `--skip-sidebar`.

**NEW FLAGS (v1.2)**:
- `--form-file` — Path to Form.xml. Enables WPForms creation from StoresOnline form definitions.
- `--formelement-file` — Path to FormElement.xml. Required with `--form-file` for form field mapping.
- `--productcategory-file` — Path to ProductCategory.xml. Enables featured product marking from `showfeatured=true` elements.

**AUTO-GENERATED**: The importer automatically detects the background template (shared layout) and generates:
- **Sidebar widgets** in the theme's primary sidebar (auto-detected via `CXI_Theme_Compat`) from LOCATION=1 elements (nav menus, testimonials, forms, category lists, cart, search)
- **Left sidebar widgets** from LOCATION=2 elements (newsletter forms, text blocks)
- **Footer menu** from LOCATION=4 elements (assigned to the correct theme menu location via `CXI_Theme_Compat`)
- **Site Navigation menu** reconstructed from Page.xml page hierarchy (parent/child relationships)
- **Homepage title hiding** via theme-agnostic `CXI_Theme_Compat::hide_page_title()` (works with Astra, Blocksy, and generic themes)
- **WPForms forms** created from Form.xml/FormElement.xml (if `--form-file` provided)
- **Featured products** marked from `showfeatured=true` elements (if `--productcategory-file` provided)
- **Category page detection** — pure category pages are NOT created as WP pages; instead, their descriptions are set on WC category terms and old URLs are mapped for rewriting
- **Border/background styling** preserved from inline HTML and XML `<border>` / `<seperate>` params

Use `--skip-sidebar` flag on re-runs to skip sidebar/footer regeneration.

Report results: pages created, category pages detected, product blocks embedded, featured images set, sidebar widgets created, footer menu items, WPForms created, featured products marked, URL rewrites stored.

### Step 3.6.5: Flush Rewrite Rules

After page import (which stores `cxi_category_url_map` for old category URLs), flush rewrite rules so the `cxi-site-customizations` plugin's URL rewrites take effect:

```bash
# Remote:
ssh {SSH_CMD} "wp rewrite flush --path={WP_PATH}"

# Local:
wp rewrite flush --path="{WP_PATH}"
```

This makes old StoresOnline category URLs (e.g., `/tulips`, `/daffodils`) resolve to WooCommerce category archives.

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

## PHASE 3.9: Site Configuration

**This phase handles configuration that import-pages does NOT do automatically.** The `import-pages` command (Step 3.6) already auto-generates sidebar widgets, footer menu, site navigation menu, WPForms, featured products, and category URL rewrites. This phase covers:

1. **Source site crawl** — Detect header navigation links and copyright text from the source site
2. **WooCommerce pages** — Create Cart, Checkout, My Account, Shop pages
3. **Homepage setting** — Set the static homepage
4. **Theme visual config** — Sidebar position, header/footer placements (Blocksy/Astra only)
5. **Header menu** — Build top navigation bar from source site links
6. **Menu location assignment** — Assign header and footer menus to theme locations
7. **Cache flush** — Clear caches and flush rewrite rules

**Theme-Agnostic Architecture**: The `cxi-site-customizations` plugin provides `CXI_Theme_Compat::get_theme_config()` which returns theme-specific configuration (sidebar IDs, menu locations, title hiding meta keys) for the active theme.

Use the **PHP Script Upload Pattern** (write PHP to `/tmp/`, scp, wp eval-file, rm) for all complex operations in this phase. This avoids SSH quoting issues with complex PHP arrays and strings.

### Step 3.9.0: Source Site Crawl (Header Nav + Copyright)

Use the Playwright MCP browser tools to crawl the source StoresOnline site (`SOURCE_SITE_URL`) homepage. This is a focused, lightweight crawl to detect only what import-pages doesn't handle:

1. Navigate to `SOURCE_SITE_URL` at 1400x900 viewport
2. Take an accessibility snapshot of the page
3. **Detect header navigation links**: Look for the top navigation bar (typically `<div class="topLinks">` or similar). Extract link text and URL paths. Store as `SOURCE_HEADER_LINKS` array of `{title, slug, type, url}` objects.
4. **Detect copyright text**: Look for copyright notice in the footer area (typically contains year and company name). Store as `SOURCE_COPYRIGHT_TEXT`.
5. **Detect any extra pages** not in the XML data (e.g., "Site Map", "Gift Card Balance") that should be created as empty WordPress pages. Store as `PAGES_TO_CREATE`.

If the source site is unreachable, ask the user to provide header menu links and copyright text manually.

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

**Skip this step entirely for non-Blocksy/non-Astra themes.** For other themes, note: "Theme '{THEME_SLUG}' requires manual Customizer setup for sidebar position and header/footer layout."

**For Blocksy theme:**
```php
<?php
// Sidebar position: default to left sidebar for StoresOnline migrations
// CRITICAL: Blocksy uses 'type-2' for left, 'type-1' for right, 'type-4' for none
// WARNING: Do NOT use 'left-sidebar' — it does NOT work with Blocksy!
$sidebar_map = array('left' => 'type-2', 'right' => 'type-1', 'none' => 'type-4');
// Default to left sidebar — StoresOnline sites typically use left sidebar layout
$structure = $sidebar_map['left'];
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
// Default to left sidebar — StoresOnline sites typically use left sidebar layout
$astra_settings['site-sidebar-layout'] = $sidebar_map['left'];
update_option('astra-settings', $astra_settings);
```

### Step 3.9.4: Build Header Menu (from SOURCE_HEADER_LINKS)

Using the `SOURCE_HEADER_LINKS` detected in Step 3.9.0 (source site crawl), write a PHP script that:
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

// SOURCE_HEADER_LINKS populated from Step 3.9.0 (source site crawl)
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

### Step 3.9.5: Assign Menu Locations

The `import-pages` command (Step 3.6) auto-creates the **Footer Menu** and **Site Navigation** menu from Page.xml data. This step assigns all menus to the correct theme locations:

```php
<?php
$locations = get_theme_mod('nav_menu_locations', array());

// 1. Assign Primary Menu (built in Step 3.9.4) to 'primary' location
$primary_menu = wp_get_nav_menu_object('Primary Menu');
if ($primary_menu) {
    $locations['primary'] = $primary_menu->term_id;
}

// 2. Assign Footer Menu (auto-created by import-pages) to footer location
$footer_menu = wp_get_nav_menu_object('Footer Menu');
if ($footer_menu) {
    // Astra uses 'footer_menu', Blocksy uses 'footer'
    $footer_location_key = (wp_get_theme()->get_template() === 'astra') ? 'footer_menu' : 'footer';
    $locations[$footer_location_key] = $footer_menu->term_id;
}

// 3. Verify Site Navigation menu exists (auto-created by import-pages for sidebar)
$sidebar_nav = wp_get_nav_menu_object('Site Navigation');
if ($sidebar_nav) {
    echo "Site Navigation menu found with " . count(wp_get_nav_menu_items($sidebar_nav->term_id)) . " items\n";
} else {
    echo "WARNING: Site Navigation menu not found — sidebar nav may need manual setup\n";
}

set_theme_mod('nav_menu_locations', $locations);
echo "Menu locations assigned successfully\n";
```

Upload and execute via `wp eval-file`.

### Step 3.9.6: Final Cache Flush

```bash
# Remote:
ssh {SSH_CMD} "wp cache flush --path={WP_PATH} && wp rewrite flush --path={WP_PATH}"

# Local:
wp cache flush --path="{WP_PATH}" && wp rewrite flush --path="{WP_PATH}"
```

### Step 3.9.7: Report Site Configuration Results

```
=== SITE CONFIGURATION COMPLETE ===

Homepage: {page_title} (ID: {page_id})
Theme: {THEME_SLUG} with left sidebar

Header Menu ({count} items):
  {item1} | {item2} | {item3} | ...

Sidebar (auto-generated by import-pages):
  - Site Navigation menu: {count} items
  - Widgets: {count} in sidebar-1

Footer Menu (auto-generated by import-pages):
  {count} items assigned to footer location
Copyright: "{SOURCE_COPYRIGHT_TEXT}"

WooCommerce Pages Created: Cart, Checkout, My Account, Shop
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
- Navigate to homepage — verify the "Home" title is NOT displayed (hidden via `CXI_Theme_Compat::hide_page_title()`)
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

#### Check 9: WooCommerce Cart/Shop & AJAX
- Navigate to /shop/ page — verify it loads
- Check if mini-cart or cart widget appears in sidebar with correct structure (heading, "Items: 0", subtotal, View Cart/Checkout links)
- Add a product to cart and verify the cart widget updates WITHOUT page reload (AJAX via `wc-cart-fragments`)
- If AJAX updates don't work, check browser console for `wc-cart-fragments` script loading

#### Check 10: Category URL Rewrites
- Test 2-3 old StoresOnline category URLs (e.g., `/tulips`, `/daffodils`) — should resolve to WC category archives (200, not 404)
- Verify WC categories have descriptions: `wp term list product_cat --fields=name,description --format=table --path={WP_PATH}`
- Verify `/product-category/{slug}/` also works (WC default URL)

#### Check 11: WPForms & Forms
- Check if WPForms were created: `wp post list --post_type=wpforms --format=table --path={WP_PATH}`
- Navigate to a page with a form (e.g., Newsletter Signup, Contact Us) and verify form renders
- Check form has correct fields (name, email, etc.)

#### Check 12: Featured Products
- Navigate to homepage and verify featured product sections display correct products
- Verify via WP-CLI: `wp wc product list --featured=true --user=1 --format=table --path={WP_PATH}`

#### Check 13: Border/Background Styling
- Navigate to pages with text blocks that had `<border>true</border>` in XML
- Verify border and background color styling is present (green borders, cornsilk backgrounds, etc.)
- Check that inline CSS from source HTML is preserved

#### Check 14: Left Sidebar Navigation
- Check if "Site Navigation" menu was created: `wp menu list --path={WP_PATH}`
- Verify it has correct hierarchy (parent/child categories from Page.xml)

#### Check 15: Plugin Status
- Verify `cxi-site-customizations` plugin is active: `wp plugin list --name=cxi-site-customizations --format=csv --path={WP_PATH}`
- Verify `wpforms-lite` plugin is active (if forms were imported)

#### Check 16: Compile Diff Report
- For each check above, record PASS/FAIL with details
- If any FAIL, attempt auto-fix using PHP eval-file scripts uploaded to the server:
  - Missing sidebar widgets → re-run `wp cxi import-pages` with sidebar generation
  - Wrong menu location → fix via `set_theme_mod('nav_menu_locations', ...)`
  - Page title showing on homepage → `CXI_Theme_Compat::hide_page_title($homepage_id)` via wp eval
  - Category URLs 404 → `wp rewrite flush`
  - `cxi-site-customizations` not active → `wp plugin activate cxi-site-customizations`
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
| 9  | Cart widget AJAX         | Updates on add       | {result}             | PASS/FAIL |
| 10 | Category URL rewrites    | Old URLs work        | {result}             | PASS/FAIL |
| 11 | WPForms forms            | Forms render         | {result}             | PASS/FAIL |
| 12 | Featured products        | Correct products     | {result}             | PASS/FAIL |
| 13 | Border/bg styling        | Styles preserved     | {result}             | PASS/FAIL |
| 14 | Site navigation menu     | Menu with hierarchy  | {result}             | PASS/FAIL |
| 15 | Plugin status            | Both active          | {result}             | PASS/FAIL |
| 16 | Auto-fixes applied       | N/A                  | {fixes_count} fixes  | INFO |
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
| `--form-file` | NO | NO | YES | NO |
| `--formelement-file` | NO | NO | YES | NO |
| `--productcategory-file` | NO | NO | YES | NO |
| `--notification-email` | NO | NO | YES | YES |
| `--path` | YES (always!) | YES (always!) | YES (always!) | YES (always!) |

### Additional Commands

**`mark-featured`** — Mark top N products in a category as WooCommerce featured:
```bash
wp cxi mark-featured ProductCategory.xml --category=<CATEGORY_ID> --limit=<N> --path={WP_PATH}
```
Uses ProductCategory.xml POSITION ordering. Requires WooCommerce to be active.

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
