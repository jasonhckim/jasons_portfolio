# Fashion Product Description Generator

Automates the generation of SEO-optimized product titles and descriptions for wholesale fashion catalogs, replacing manual copywriting with AI-powered vision analysis and structured output.

## Problem

Wholesale fashion manufacturers produce hundreds of new styles per season, each requiring:
- SEO-optimized product titles (32-45 characters)
- Detailed product descriptions (300-350 characters)
- Structured attributes (fabric, silhouette, neckline, sleeve, length)
- Marketplace-specific formatting (Faire, FashionGo, Shopify)

Manual copywriting for each product takes 5-10 minutes. For a 50-product catalog, this represents 4-8 hours of repetitive work. The system reduces this to minutes while maintaining consistency and quality standards.

## Constraints

**Technical:**
- Must process PDF catalogs with mixed text/image layouts
- Images are primary evidence; PDF text is secondary and often inconsistent
- Output must meet strict character limits (250-350 chars) for marketplace APIs
- Must integrate with Google Sheets (collaboration) and Shopify (e-commerce)

**Business:**
- Descriptions must be factual, not marketing fluff (B2B audience)
- Must handle style number variations (DZ/HF prefixes, SET suffixes)
- Must preserve human review workflow (editable Google Sheets)
- Cost-sensitive: OpenAI API calls must be efficient

**Operational:**
- Runs on Google Cloud Run (serverless, pay-per-use)
- Must handle authentication via service accounts
- Must gracefully degrade when AI fails
- No manual intervention during batch processing

## System Architecture

```
PDF Catalog (Google Drive)
    ↓
[PDF Processor Service]
    ├─ Extract text + images per page
    ├─ Parse style numbers (regex: DZ/HF patterns)
    └─ Skip pages without images
    ↓
[AI Description Generator]
    ├─ Vision analysis (GPT-4o) of product images
    ├─ Cross-reference with extracted PDF text
    ├─ Generate title + description + attributes
    └─ Retry logic (3 attempts) with JSON fallback
    ↓
[Data Validation & Formatting]
    ├─ Enforce character limits (250-350)
    ├─ Fill missing attributes with "N/A"
    ├─ Map attributes to marketplace schema
    └─ Create structured DataFrames
    ↓
[Google Sheets Writer]
    ├─ Create sheet from template (or blank)
    ├─ Write main tab (all attributes)
    ├─ Write Designer tab (title/description only)
    ├─ Create Marketplace sheet (Faire attributes)
    └─ Transfer ownership + add editors
    ↓
[Shopify Sync Service] (optional, separate trigger)
    ├─ Read all sheets from folder
    ├─ Find products by SKU (style number)
    └─ Update title + description via API
```

**Deployment:**
- PDF Processor: Cloud Run service (Flask wrapper)
- Shopify Sync: Separate Cloud Run service
- Trigger: Manual (`trigger-cloud-run.sh`) or Cloud Scheduler
- Storage: Google Drive (PDFs, Sheets)
- Authentication: Service account with Drive/Sheets/Apps Script scopes

## AI vs Deterministic Logic

**AI (GPT-4o Vision):**
- **Product title generation**: Analyzes images to identify garment type, construction details, neckline, sleeve style
- **Product description generation**: Creates 300-350 character descriptions focusing on construction, fit, and styling
- **Attribute extraction**: Infers fabric, silhouette, length, neckline, sleeve from visual analysis
- **Evidence prioritization**: Images override PDF text when inconsistent (e.g., PDF says "high neck" but image shows crew neck)

**Why AI here:**
- Visual analysis requires understanding garment construction, fabric texture, and design details
- Natural language generation must match brand voice and B2B tone
- Attribute inference from images is non-trivial (e.g., distinguishing "A-line" vs "fit-and-flare")

**Deterministic Logic:**
- **PDF extraction**: PyMuPDF for text/image extraction (no AI needed)
- **Style number parsing**: Regex patterns (`DZ\d{2}[A-Z]\d{3,5}(-SET|-D)?`)
- **Character count validation**: Truncate to 349 chars, pad to 250 chars minimum
- **Data structure mapping**: Fixed schema for Google Sheets columns
- **Marketplace attribute mapping**: Direct mapping from AI attributes to Faire schema (e.g., `sleeve` → `TOP: Sleeve Length (1)`)
- **Sheet creation/formatting**: Template copying, column ordering, tab creation

**Why deterministic here:**
- PDF parsing is well-solved (PyMuPDF)
- Style numbers follow fixed patterns
- Data validation requires exact rules (character limits, required fields)
- Sheet formatting is structural, not semantic

**Hybrid Approach:**
- AI generates content; deterministic logic validates and formats it
- AI provides attributes; deterministic logic maps to marketplace schemas
- AI may return "N/A" for unclear features; deterministic logic ensures all required fields exist

## Failure Modes & Safeguards

**Invalid AI Output:**
- **JSON parsing failures**: Regex extraction from raw text if function calling fails; 3 retry attempts with exponential backoff
- **Missing required fields**: Default to "N/A" for all attributes; ensure all 13 expected columns exist
- **Character count violations**: Truncate to 349 chars (add "..."), pad to 250 chars minimum
- **"N/A" product titles**: Skip entry entirely (logged, doesn't break batch)

**Edge Cases:**
- **Pages without images**: Skip page, log warning, continue processing
- **PDF download failures**: Skip PDF, log error, continue to next PDF
- **Style number not found**: Default to "Unknown", still process (may generate invalid output)
- **Template sheet copy fails**: Fallback to blank sheet creation
- **Apps Script attachment fails**: Log warning, continue (manual attachment possible)

**Data Quality Issues:**
- **Inconsistent PDF text**: AI prioritizes images over text (explicit prompt instruction)
- **Unclear visual features**: AI omits rather than assumes (prompt: "Never assume")
- **Missing keywords file**: Empty keyword list, AI still generates descriptions
- **Shopify product not found**: Log error, continue to next product (doesn't fail batch)

**Operational Safeguards:**
- **API rate limits**: Retry logic with 2-second delays between attempts
- **Service account permissions**: Multiple credential fallback paths (env var → file → ADC)
- **Cloud Run timeouts**: Process one PDF per invocation (stateless design)
- **Error logging**: Structured logging to Google Cloud Logging with error types and style numbers
- **Partial failures**: Continue processing remaining products if one fails

**Human Review Layer:**
- All output written to editable Google Sheets (not directly to Shopify)
- "Edit Product Title" and "Edit Product Description" columns for manual overrides
- Designer tab provides simplified view for copywriters
- Marketplace sheet separated for different workflow

## Results & Impact

**Quantitative:**
- **Time savings**: 4-8 hours → 5-10 minutes per 50-product catalog (95%+ reduction)
- **Consistency**: 100% adherence to character limits and required fields
- **Throughput**: Process 50 products in ~10-15 minutes (vs. 4-8 hours manual)
- **Cost**: ~$0.02-0.05 per catalog (OpenAI API + Cloud Run compute)

**Qualitative:**
- **Quality**: AI-generated descriptions match or exceed manual quality for factual content
- **Scalability**: Handles seasonal catalog volumes (200+ products) without additional staffing
- **Reliability**: Graceful degradation ensures partial success even with API failures
- **Workflow integration**: Fits existing Google Sheets + Shopify workflow without disruption

**Business Impact:**
- Enables faster time-to-market for new product launches
- Frees copywriters to focus on brand voice and marketing copy (not technical descriptions)
- Reduces human error in character counting and attribute mapping
- Standardizes output format across multiple marketplaces

## What I'd Improve Next

**AI Improvements:**
- Fine-tune GPT-4o on brand-specific examples to improve tone consistency
- Add confidence scores for AI-generated attributes (flag low-confidence for human review)
- Implement A/B testing framework for prompt variations
- Cache similar product descriptions to reduce API costs

**System Reliability:**
- Add structured validation layer before Google Sheets write (catch errors earlier)
- Implement dead-letter queue for failed products (retry with exponential backoff)
- Add monitoring dashboard (success rate, API latency, cost per product)
- Create automated tests for edge cases (missing images, malformed PDFs)

**Feature Enhancements:**
- Support batch processing of multiple PDFs in single invocation
- Add webhook notifications when sheets are created (Slack/email)
- Implement version control for descriptions (track edits, rollback capability)
- Add marketplace-specific formatting rules (beyond Faire)

**Operational:**
- Migrate to Cloud Functions Gen 2 (better cold start performance)
- Implement cost alerts (budget notifications for OpenAI API)
- Add health check endpoints for monitoring
- Create runbook documentation for common failure scenarios
