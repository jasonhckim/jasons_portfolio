# Post-Purchase Recommendation Email System

Automates personalized line sheet delivery to wholesale customers immediately after order placement, replacing manual follow-up emails with AI-generated content and data-driven product recommendations.

## Problem

Wholesale fashion customers place orders through Shopify, but the sales team lacks capacity to manually follow up with personalized product recommendations before shipping. This creates missed opportunities for order add-ons and reduces customer engagement. The system must trigger automatically on order events, generate personalized recommendations from thousands of SKUs, and deliver professional line sheets within minutes—all without human intervention.

## Constraints

- **Real-time processing**: Must respond to Shopify webhooks within Cloud Run timeout limits (typically 60-300 seconds)
- **Data freshness**: Product inventory and customer purchase history must be current, requiring daily BigQuery syncs from Shopify
- **Email deliverability**: Must use Gmail API with proper authentication and avoid spam triggers
- **Cost control**: AI API calls (OpenAI GPT-4) must be limited to essential content generation, not used for every recommendation
- **Duplicate prevention**: Customers must not receive multiple emails per day, even if multiple orders are placed
- **PDF generation**: Line sheets must render correctly across platforms without external dependencies (fallback mechanisms required)

## System Architecture

```
Shopify Order Webhook
    ↓
Cloud Run Service (order_email_webhook/main.py)
    ↓
┌─────────────────────────────────────┐
│ 1. Extract customer email & order  │
│ 2. Check duplicate email log       │
│ 3. Fetch customer purchase history │
│    from BigQuery (last 365 days)   │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Recommendation Engine               │
│ - Extract tags from purchased SKUs │
│ - Score unpurchased products by    │
│   tag similarity (deterministic)   │
│ - Add 5 push styles (new/overstock)│
│ - Return top 25 recommendations    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Match to Active Shopify Products    │
│ - Filter to recommended SKUs        │
│ - Group by style & color variants  │
│ - Validate inventory availability  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Generate PDF Line Sheet             │
│ - Fetch product images (with        │
│   fallback to placeholder)          │
│ - Render via Jinja2 template       │
│ - Convert to PDF (pdfkit → weasyprint)│
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Generate Email Content (AI)         │
│ - GPT-4 generates personalized body │
│ - Includes order context (total,    │
│   date) from webhook payload        │
│ - Fallback template if AI fails    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ Send via Gmail API                  │
│ - Attach PDF line sheet             │
│ - Log email ID & timestamp          │
│ - Prevent future sends today        │
└─────────────────────────────────────┘
```

**Supporting Services** (run separately):
- `shopify-products-sync`: Daily sync of product catalog to BigQuery
- `shopify-orders-sync`: Daily sync of order history to BigQuery
- `ai_tagger/tag_products.py`: Batch AI tagging of products for recommendation engine

## AI vs Deterministic Logic

**AI Components:**

1. **Email Content Generation** (`utils/email_writer.py`): GPT-4 generates personalized email body based on customer name, order total, and purchase date. AI is used here because tone, warmth, and professional phrasing are difficult to template deterministically while maintaining personalization.

2. **Product Tagging** (`ai_tagger/tag_products.py`): GPT-4 extracts fashion-related tags, style types, and vibes from product titles and descriptions. Used for products missing tags in Shopify, enabling the recommendation engine to work on untagged inventory.

**Deterministic Components:**

1. **Product Recommendations** (`recommend/recommend_products.py`): Tag-based similarity scoring using keyword matching. Deterministic because:
   - Consistency: Same customer + same purchase history = same recommendations
   - Performance: Processes thousands of products in seconds without API costs
   - Explainability: Recommendations can be traced to specific tag matches

2. **PDF Generation** (`linesheet/generate_linesheet.py`): Template-based rendering with deterministic image embedding and layout. No AI needed for structured document generation.

3. **Duplicate Prevention** (`email_log.py`): CSV-based logging with date matching. Simple, reliable, and doesn't require database infrastructure.

**Decision Rationale:** AI is reserved for content generation where variability and nuance add value. Product matching and filtering use deterministic logic for speed, cost, and reliability. This hybrid approach balances personalization with operational efficiency.

## Failure Modes & Safeguards

**Invalid AI Output:**
- Email generation includes try/except with fallback HTML template if GPT-4 fails or returns malformed content
- Product tagging skips products that fail AI processing rather than blocking the pipeline

**Edge Cases:**
- **Missing customer email**: Webhook returns 400 error, no email sent
- **No purchase history**: Recommendation engine returns empty result, email skipped with logged reason
- **No matching products**: Filters out SKUs not found in active Shopify catalog, skips email if no valid matches
- **Image fetch failures**: PDF generator uses transparent placeholder if product images are unavailable
- **PDF generation failure**: Falls back from pdfkit to weasyprint if wkhtmltopdf is unavailable

**Data Quality Issues:**
- **Missing product tags**: Untagged products are excluded from recommendations (handled by tag-based scoring)
- **Stale inventory data**: Daily BigQuery syncs ensure product availability is current
- **Duplicate webhook events**: Email log prevents sending multiple emails per customer per day

**Operational Safeguards:**
- `DRY_RUN` and `TEST_MODE` environment variables route all emails to test address
- Comprehensive error logging with stack traces for debugging
- Graceful degradation: System skips problematic customers rather than failing entire batch

## Results & Impact

- **Automation**: Eliminates manual follow-up emails, freeing sales team time for high-value activities
- **Speed**: Customers receive personalized line sheets within minutes of order placement (vs. hours/days manually)
- **Scale**: Handles unlimited concurrent orders without linear cost increase
- **Consistency**: Every customer receives follow-up, reducing missed opportunities from human oversight
- **Measurable**: Email logs enable tracking of send rates, customer engagement, and recommendation effectiveness

**Estimated Impact:**
- Time savings: ~15-30 minutes per order × order volume
- Revenue opportunity: Enables order add-ons that would otherwise be missed due to timing
- Customer experience: Professional, personalized communication improves brand perception

## What I'd Improve Next

1. **Recommendation Quality**: Replace tag-based scoring with collaborative filtering or embedding-based similarity (e.g., product embeddings from purchase co-occurrence) for more nuanced recommendations

2. **A/B Testing Framework**: Instrument email content variations (AI-generated vs. templates) and recommendation algorithms to measure conversion rates and optimize over time

3. **Real-time Inventory**: Integrate live Shopify inventory API calls instead of daily BigQuery syncs to prevent recommending out-of-stock items

4. **Email Engagement Tracking**: Parse Gmail API responses for opens/clicks and feed back into recommendation scoring (e.g., deprioritize styles that customers consistently ignore)

5. **Multi-channel Delivery**: Add SMS or in-app notifications as fallback if email bounces, with preference learning based on customer engagement patterns

6. **Cost Optimization**: Cache AI-generated email templates by customer segment to reduce GPT-4 API calls while maintaining personalization

7. **Observability**: Add structured logging (e.g., Cloud Logging) with metrics dashboards for send rates, failure modes, and recommendation quality scores
