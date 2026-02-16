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

## PHASE 0: Environment Detection

### Step 0.1: Site Type

Use the `AskUserQuestion` tool to ask:

**Question**: "Where is the target WordPress site running?"
- **Local (Local by Flywheel)** (Recommended for development) — WordPress is running locally via Local by Flywheel app
- **Remote server (SSH)** — WordPress is on a remote server accessible via SSH

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
- `WP_CMD`: `wp` (direct, no prefix)
- `WP_PATH`: The WordPress installation root path
- `SITE_TYPE`: `local`

### Step 0.2b: If Remote Server

1. Use `AskUserQuestion` to gather SSH connection details one by one:
   - **SSH Username** (e.g., `root`, `deploy`, `www-data`)
   - **SSH Host** (e.g., `example.com` or `192.168.1.100`)
   - **SSH Port** (default: `22`)
   - **Authentication method**: "Password" or "SSH key file"
     - If key file: ask for path (e.g., `~/.ssh/id_rsa`)
     - If password: ask for the password

2. Construct and test the SSH connection:
   ```bash
   # With key file:
   ssh -p {PORT} -i {KEYFILE} -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new {USER}@{HOST} echo "SSH connection successful"

   # With password (use sshpass if available, otherwise inform user):
   sshpass -p '{PASSWORD}' ssh -p {PORT} -o ConnectTimeout=10 {USER}@{HOST} echo "SSH connection successful"
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
- `WP_CMD`: `ssh {SSH_CMD} "wp --path={WP_REMOTE_PATH}"` (all wp commands via SSH)
- `SSH_CMD`: The full SSH connection string
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

Check for each required XML file and report found/missing:

```bash
# For each file, check existence and count rows
for f in Customer.xml CustomerGroup.xml AccountGroup.xml Product.xml Category.xml ProductCategory.xml Price.xml RewriteURL.xml Page.xml PageElement.xml AccountOrder.xml; do
  if [ -f "{DATA_DIR}/$f" ]; then
    COUNT=$(grep -c '<row>' "{DATA_DIR}/$f" 2>/dev/null || echo "0")
    echo "FOUND: $f ($COUNT records)"
  else
    echo "MISSING: $f"
  fi
done
```

For remote sites, prefix with `ssh {SSH_CMD}`.

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

Look for image folders in the client data parent directory:
```bash
find "{CLIENT_DIR}" -maxdepth 1 -type d | sort
```
Count image files across all folders:
```bash
find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) | wc -l
```

Report the number of image folders and total image count.

### Step 1.4: Analyze Orders

If `AccountOrder.xml` exists, run:
```bash
{WP_CMD} cxi list-order-form-names "{DATA_DIR}/AccountOrder.xml"
```

This shows all order form types and counts (e.g., "Secure Checkout": 13,334, "Contact Form": 2,000).

Then run validation:
```bash
{WP_CMD} cxi validate-orders "{DATA_DIR}/AccountOrder.xml"
```

Report validation results including customer ID matches, product matches, and any warnings.

### Step 1.5: Present Summary

Display a comprehensive summary table:

```
=== MIGRATION DATA ANALYSIS ===

| Data Type        | Source File        | Records | Status  |
|------------------|--------------------|---------|---------|
| Customers        | Customer.xml       | 11,148  | Ready   |
| Customer Groups  | AccountGroup.xml   | 8       | Ready   |
| Products         | Product.xml        | 770     | Ready   |
| Categories       | Category.xml       | 913     | Ready   |
| Prices           | Price.xml          | 1,540   | Ready   |
| Pages            | Page.xml           | 180     | Ready   |
| Page Elements    | PageElement.xml    | 2,345   | Ready   |
| Orders           | AccountOrder.xml   | 19,726  | Ready   |
| Images           | 40 folders         | 1,023   | Ready   |

Order Form Types:
  - Secure Checkout: 13,334 (purchase orders)
  - Contact Form: 2,000 (contact submissions)
  - Newsletter Registration: 1,500

Validation Issues:
  - None (or list issues found)
```

---

## PHASE 2: Configuration Prompts

Gather ALL configuration decisions BEFORE starting migration.

### Step 2.1: Product Variation Strategy

Use `AskUserQuestion`:

**Question**: "How should products with variations (size, color, etc.) be imported?"
- **Standard variable products (Recommended)** — Uses built-in WooCommerce variable products. Free, no extra plugins needed. Creates parent product with child variations.
- **WooCommerce Product Add-Ons** — Creates simple products with add-on options. Requires the paid "WooCommerce Product Add-Ons" plugin.

If user selects Add-Ons:
1. Check if the plugin is installed: `{WP_CMD} plugin list --name=woocommerce-product-addons --format=csv`
2. If not installed, ask user to provide the plugin zip file and specify its location
3. Ask addon type with `AskUserQuestion`:
   - **Select dropdown (Recommended)**
   - **Radio buttons**
   - **Checkboxes**
4. Ask if add-ons should be required: Yes/No

Store configuration:
- `ADDON_MODE`: `true` or `false`
- `ADDON_TYPE`: `select`, `radiobutton`, or `checkbox`
- `ADDON_REQUIRED`: `true` or `false`

### Step 2.2: Order Form Names

Based on the form names discovered in Phase 1, use `AskUserQuestion`:

**Question**: "Which order form types should be imported as WooCommerce orders?"
- Show each form name found with its count
- Default recommendation: "Secure Checkout" (actual purchase orders)
- Allow multiple selection if applicable

Store as: `ORDER_FORM_NAMES` (comma-separated string for `--form-names` flag)

### Step 2.3: Default Country

Use `AskUserQuestion`:

**Question**: "What is the default country code for customers with missing country data?"
- **US - United States (Recommended)**
- **CA - Canada**
- **GB - United Kingdom**
- **Other** (ask for ISO 3166-1 alpha-2 code)

Store as: `DEFAULT_COUNTRY`

### Step 2.4: Batch Sizes

Use `AskUserQuestion`:

**Question**: "Use default batch sizes for imports?"
- **Yes, use defaults (Recommended)** — Customers: 100, Products: 50, Orders: 50
- **Custom** — Specify custom batch sizes

Store as: `BATCH_CUSTOMER`, `BATCH_PRODUCT`, `BATCH_ORDER`

### Step 2.5: Skip Images

Use `AskUserQuestion`:

**Question**: "Should image migration be performed?"
- **Yes, copy images (Recommended for first run)** — Copies all product images to WordPress uploads directory
- **No, skip images** — Use this for re-runs where images are already in place

Store as: `SKIP_IMAGES`: `true` or `false`

### Step 2.6: Configuration Summary

Display all collected configuration:

```
=== MIGRATION CONFIGURATION ===

Site: {siteurl} ({SITE_TYPE})
Product Variations: Standard variable products / Add-Ons ({ADDON_TYPE})
Order Forms: {ORDER_FORM_NAMES}
Default Country: {DEFAULT_COUNTRY}
Batch Sizes: Customers={BATCH_CUSTOMER}, Products={BATCH_PRODUCT}, Orders={BATCH_ORDER}
Skip Images: {SKIP_IMAGES}

Proceed with migration? (Continue to Phase 3)
```

---

## PHASE 3: Prerequisites Check & Migration Execution

### Step 3.0: Pre-flight Checks

Run these checks and report status:

1. **WP-CLI connection**:
   ```bash
   {WP_CMD} option get siteurl
   ```

2. **WordPress memory limits** — Check wp-config.php for:
   - `WP_MEMORY_LIMIT` should be `2048M`
   - `WP_MAX_MEMORY_LIMIT` should be `2048M`
   If missing, warn the user and suggest adding them.

3. **Plugin status**:
   ```bash
   {WP_CMD} plugin list --name=customer-xml-importer --format=csv
   ```
   Must be active. If not active, activate it:
   ```bash
   {WP_CMD} plugin activate customer-xml-importer
   ```

4. **Uploads directory**:
   ```bash
   mkdir -p "{WP_PATH}/wp-content/uploads/upload-images"
   ```
   For remote: `ssh {SSH_CMD} "mkdir -p {WP_PATH}/wp-content/uploads/upload-images"`

5. **Debug logging** — Check wp-config.php has:
   - `WP_DEBUG` = `true`
   - `WP_DEBUG_LOG` = `true`
   - `WP_DEBUG_DISPLAY` = `false`

Report pre-flight results as a checklist:
```
Pre-flight Check:
  [x] WP-CLI connected: {siteurl}
  [x] Memory: 2048M
  [x] Plugin: customer-xml-importer v1.1.0 (active)
  [x] Upload directory: created
  [x] Debug logging: enabled
```

If any critical check fails, stop and help the user fix it before continuing.

### Step 3.1: Install WooCommerce & Theme

```bash
{WP_CMD} plugin install woocommerce --activate
{WP_CMD} theme install astra --activate
```

If addon mode is enabled:
```bash
# Install the addon plugin from provided path
{WP_CMD} plugin install "{ADDON_PLUGIN_PATH}" --activate
```

Verify WooCommerce is active:
```bash
{WP_CMD} plugin list --name=woocommerce --status=active --format=csv
```

### Step 3.2: Database Preparation

```bash
{WP_CMD} cxi db-migrate
```

This creates performance indexes on wp_postmeta. Report which indexes were created or skipped.

### Step 3.3: Customer Import

```bash
{WP_CMD} cxi import "{DATA_DIR}/Customer.xml" "{DATA_DIR}/CustomerGroup.xml" "{DATA_DIR}/AccountGroup.xml" --batch-size={BATCH_CUSTOMER} --default-country={DEFAULT_COUNTRY} --import-fields
```

**IMPORTANT**: Use a timeout of 600000ms (10 minutes) for this command. For very large datasets (>20,000 customers), consider running in background.

Report results: customers created, updated, errors.

### Step 3.4: Image Migration (BEFORE products)

**CRITICAL**: This step MUST happen BEFORE product import. The product importer looks for images in `wp-content/uploads/upload-images/` during import.

If `SKIP_IMAGES` is false:

For local sites:
```bash
# Find all image files in client data folders and copy to upload-images
find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) -exec cp {} "{WP_PATH}/wp-content/uploads/upload-images/" \;
```

For remote sites (files already on server):
```bash
ssh {SSH_CMD} 'find "{CLIENT_DIR}" -maxdepth 2 -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.gif" -o -name "*.webp" -o -name "*.bmp" \) -exec cp {} "{WP_PATH}/wp-content/uploads/upload-images/" \;'
```

For remote sites (files need upload):
```bash
# First copy all images to a local temp directory, then rsync/scp
rsync -avz --include="*/" --include="*.jpg" --include="*.jpeg" --include="*.png" --include="*.gif" --include="*.webp" --include="*.bmp" --exclude="*" "{CLIENT_DIR}/" "{SSH_USER}@{SSH_HOST}:{WP_PATH}/wp-content/uploads/upload-images/"
```

Count and report images copied:
```bash
ls "{WP_PATH}/wp-content/uploads/upload-images/" | wc -l
```

### Step 3.5: Product Import (AFTER images)

Build the command based on configuration:

```bash
# Base command (4 positional args required)
{WP_CMD} cxi import-products "{DATA_DIR}/Product.xml" "{DATA_DIR}/Category.xml" "{DATA_DIR}/ProductCategory.xml" "{DATA_DIR}/Price.xml" --batch-size={BATCH_PRODUCT} --import-fields
```

If addon mode is enabled, append: `--addon --addon-type={ADDON_TYPE} --addon-required` (if required was chosen)

**IMPORTANT**: Use a timeout of 600000ms (10 minutes).

Report results: products created, variations created, categories, errors.

### Step 3.6: Page Import

```bash
{WP_CMD} cxi import-pages "{DATA_DIR}/Page.xml" "{DATA_DIR}/PageElement.xml" --category-file="{DATA_DIR}/Category.xml" --product-file="{DATA_DIR}/Product.xml" --rewriteurl-file="{DATA_DIR}/RewriteURL.xml"
```

Report results: pages created, product blocks embedded, featured images set.

### Step 3.7: Order Import

```bash
{WP_CMD} cxi import-orders "{DATA_DIR}/AccountOrder.xml" --fast --form-names="{ORDER_FORM_NAMES}" --batch-size={BATCH_ORDER} --import-fields
```

**IMPORTANT**: Use a timeout of 600000ms (10 minutes). This is typically the longest-running import.

The `--fast` flag uses direct database insertion bypassing WooCommerce hooks for much faster import.

Report results: orders imported, skipped, errors.

### Step 3.8: Post-Migration Cleanup

```bash
{WP_CMD} cache flush
{WP_CMD} rewrite flush
{WP_CMD} wc tool run recount_terms --user=1
```

---

## PHASE 4: Verification

Count all imported data and compare against source XML counts from Phase 1:

```bash
# Customers
{WP_CMD} user list --role=customer --format=count

# Products
{WP_CMD} post list --post_type=product --post_status=any --format=count

# Product variations
{WP_CMD} post list --post_type=product_variation --post_status=any --format=count

# Product categories
{WP_CMD} term list product_cat --format=count

# Pages
{WP_CMD} post list --post_type=page --post_status=any --format=count

# Orders (HPOS)
{WP_CMD} wc shop_order list --format=count --user=1 2>/dev/null || {WP_CMD} post list --post_type=shop_order --post_status=any --format=count

# Products with images
{WP_CMD} eval 'global $wpdb; echo $wpdb->get_var("SELECT COUNT(DISTINCT post_id) FROM {$wpdb->postmeta} WHERE meta_key = \"_thumbnail_id\" AND post_id IN (SELECT ID FROM {$wpdb->posts} WHERE post_type = \"product\")");'

# Media attachments
{WP_CMD} post list --post_type=attachment --format=count
```

Present verification as comparison table:

```
=== VERIFICATION ===

| Data Type          | Source XML | Imported | Match |
|--------------------|------------|----------|-------|
| Customers          | 11,148     | 11,092   | ~99%  |
| Products           | 770        | 770      | 100%  |
| Product Variations | -          | 4,602    | -     |
| Categories         | 913        | 912      | ~100% |
| Pages              | 180        | 178      | ~99%  |
| Orders             | 13,334     | 13,352   | ~100% |
| Images             | 1,023      | 692/770  | 90%   |
```

Flag any significant mismatches (>5% difference) for investigation.

---

## PHASE 5: Report Generation

Generate three documentation files in the client-brief directory:

### 5.1: MIGRATION_REPORT.md

Create a comprehensive report with:
- Migration summary table (all phases with status)
- Data counts table (source vs imported)
- Issues encountered and how they were resolved
- Plugin modifications made (if any)
- Environment details (WordPress version, WooCommerce version, PHP version)
- Performance metrics (duration of each import phase)
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
Duration: ~{total_time}

Data Imported:
  Customers: {count}
  Products: {count} ({variations} variations)
  Categories: {count}
  Pages: {count}
  Orders: {count}
  Images: {count}

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
```

---

## IMPORTANT NOTES

### WP-CLI Argument Syntax
The `customer-xml-importer` plugin uses **positional arguments**, NOT named flags for file paths:
```bash
# CORRECT - positional args:
wp cxi import Customer.xml CustomerGroup.xml AccountGroup.xml

# WRONG - named flags:
wp cxi import --file=Customer.xml --groups=CustomerGroup.xml
```

### The `--import-fields` Flag
ALWAYS include `--import-fields` on `import`, `import-products`, and `import-orders` commands. Without it, WP-CLI will prompt interactively for field mapping which will hang in non-interactive mode.

### Timeouts
Customer, product, and order imports can take 10-15 minutes for large datasets. Always use `timeout: 600000` (10 minutes) when running these commands with the Bash tool.

### Image Timing
Images MUST be copied to `wp-content/uploads/upload-images/` BEFORE running product import. The product importer checks for local image files during import and assigns them to products.

### Error Recovery
- The plugin supports resume capability. If an import crashes, re-running the same command will skip already-imported records (identified by `_xml_product_id`, `_xml_customer_id`, or `_xml_order_id` meta).
- Invalid billing emails during order import have been fixed with `sanitize_email()` and try-catch in `class-cxi-order-importer.php`.
