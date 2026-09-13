# 🏠 NYC Airbnb End-to-End Data Engineering & Business Analytics Project

This project encompasses an end-to-end data cleaning, validation, engineering, and business analytics workflow on the New York City (NYC) Airbnb dataset. Designed with a data-integrity-first approach, the process resolves spatial inconsistencies without introducing synthetic bias, manages non-standard missing values, eliminates logical outliers, and standardizes data schemas. The cleaned dataset was exported in Parquet format and ingested into Power BI, where a 4-page interactive dashboard was built to model market dynamics, host portfolio structures, and customer satisfaction metrics.

---

## 🛠️ Part 1: Data Cleaning, Standardization & Engineering (Python & EDA)

The data pipeline enforces a rigorous data integrity methodology to transform raw, noisy data into an analysis-ready asset.

### Key Engineering & Cleaning Highlights
* **Data Type Optimization & Schema Standardization:** Essential analytical columns were cast to precise types (dates: `datetime64`, ratings/counts: `integer`, categorical attributes: `string`) to enforce schema compliance during Power BI ingestion.

* **Outlier Analysis & Cell-Level Cleaning Strategy:**
  * `minimum_nights` Feature: 48 rows containing physically impossible values (< 1 or > 365 nights) were permanently removed from the dataset.
  * `availability_365` Feature: 3,191 invalid records were identified, including negative values (426 rows) and entries exceeding 365 days (2,765 rows). A cell-level cleaning approach was applied: invalid values were set to `NaN` to prevent data loss while preserving other valid dimensions (price, location, room type) of the listings.

* **Missing Data Imputation & Validation Methodology:**
  * **Metadata Cleanup:** The `license` column, which was 100% missing, was dropped entirely.
  * **Deterministic Borough Correction:** Typographical errors in `neighbourhood_group` (e.g., manhatan, brookln) were corrected, and missing borough values were deterministically imputed with 100% accuracy using inner-neighbourhood mapping.
  * **Spatial Proximity Analysis (k-NN):** Missing sub-neighbourhoods were evaluated via Euclidean distance calculations on latitude and longitude coordinates. For 15 records without exact spatial matches, values were intentionally left as `NaN` to avoid introducing artificial bias.
  * **Review & Interaction Logic:** Records with `NaN` in `last_review` and `reviews_per_month` alongside `number_of_reviews == 0` were confirmed as unreviewed listings. Entries with valid `review_rate_number` ratings but missing textual reviews (star-only ratings) were validated as legitimate data and retained to preserve dimensional context.

* **Deduplication & Data Hygiene:** Primary key (`id`) and full row-level deduplication were performed; categorical text fields were verified free of hidden spaces or empty string anomalies.

* **High-Performance BI Integration:** In addition to CSV, the cleaned dataset was serialized into Parquet format to enforce exact data types, deliver high columnar compression, and optimize I/O performance in Power BI reporting.

---

## 📊 Part 2: Power BI Business Analytics & Interactive Dashboard

The cleaned Parquet data was ingested into Power BI to construct a 4-page interactive dashboard analyzing market dynamics, host strategies, and review metrics.

### 📐 Principle of Strict Methodological Isolation
> 📝 **Critical Analytical Note:**  
> Host portfolio segmentation (*Host Listing Group*) relies on the platform-level global metadata field (`calculated_host_listings_count`). Conversely, the Estimated Revenue Potential metric strictly isolates financial logic to physical listings located within New York City borders by multiplying row-level metrics, excluding out-of-region revenues.

#### 🟢 Page 1: Market Overview & Snapshot

<img width="906" height="506" alt="overview   market snapshot" src="https://github.com/user-attachments/assets/99b9e143-a975-4a4d-9cda-d4eb9e6a8617" />

* **Market Scale & Supply Structure:** The NYC Airbnb market comprises 101.61K listings and 101.61K unique hosts, with an average night price of $625.36. Total market estimated annual revenue potential reaches $14.92 Billion.
* **Regional Distribution:** Manhattan leads the market with 43,366 listings (42.7%) and $6.36B revenue potential, followed by Brooklyn with 41,464 listings (40.8%). Outer boroughs (Queens, Bronx, Staten Island) represent the remaining supply.
* **Room Type Breakdown:** The supply is predominantly Entire Home/Apt (52.3%) and Private Room (45.4%). Shared Room (2.2%) and Hotel Room (0.1%) represent niche segments.

#### 🔵 Page 2: Pricing & Revenue Analytics

<img width="884" height="497" alt="Market   Pricing Dynamics" src="https://github.com/user-attachments/assets/59a3ab82-0783-4f30-8f51-c70b84c38a4b" />


* **Price Distribution & Premium Segment:** Manhattan aligns with the market average at $622.69 average night price and leads in total volume. In contrast, Queens accounts for 13,149 listings with an average price of $630.06, generating $1.74 Billion in estimated revenue potential.
* **Revenue Potential Dynamics:** The highest average revenue potential resides in the Entire Home/Apt segment, directly driven by night constraints and availability days.

#### 🟣 Page 3: Host & Listing Analytics

<img width="886" height="496" alt="Host   Listing Insights" src="https://github.com/user-attachments/assets/a454140b-e6fa-4ed5-9edb-0431b4db1a4d" />


* **Dominance of Single-Property Hosts:** The market is overwhelmingly operated by individual hosts with a single listing. Hosts with 1 Listing (62,848 hosts) account for 61.8% of listing volume and generate $10.32 Billion (69.2%) in revenue potential.
* **Commercial Operations:** Multi-listing or commercial operators (groups ranging from 2 to 50+ properties) manage 38,762 listings and generate $4.60 Billion (30.8%) in revenue.
* **Verification Profile:** Host verification status is evenly balanced between Verified (49.9%) and Unverified (49.9%).
* **Top Individual Listing:** The top-grossing individual listing (ID: 29531702698) achieves an estimated annual revenue potential of $480,160.00 at a 58% occupancy rate.

#### 🟡 Page 4: Reviews & Rating Analytics

<img width="887" height="497" alt="Rating   Reviews Insights" src="https://github.com/user-attachments/assets/3717b49a-0045-44e5-ad86-fbb48eddd0f8" />


* **High Data Coverage:** 84.5% of active listings possess valid review data (Listings with Reviews), capturing 2.79 Million reviews with an average rating of 3.28.
* **Volume vs. Satisfaction Divergence:** While Brooklyn (1.18M) and Manhattan (1.05M) receive 79.7% of all reviews, higher average ratings are observed in lower-volume outer boroughs — Staten Island (3.40), Bronx (3.33), and Queens (3.33).
* **Rating Distribution:** Listings rated 2, 3, 4, and 5 stars are evenly distributed at ~23,000 listings each, while 1-star listings remain at 9,138, reflecting maintained platform quality baselines.
* **Room Type Satisfaction Trend:** Entire Home and Private Room options average 3.28 stars. As review volume decreases, the professional Hotel Room segment achieves the highest satisfaction rating at 3.55 stars.

---

## 📐 Core DAX Measures & Calculated Columns

📐 Core DAX Measures & Calculated Columns¶

🛠️ Calculated Columns (Row-Context Rules)
// 1. Availability Grouping (Inventory Strategy & Missing Data Handling)
// Segments annual listing availability into operational buckets while explicitly categorizing null/invalid values as "Unknown".

Availability Group = 
SWITCH(
    TRUE(),
    ISBLANK(veri[availability_365]), "Unknown",  
    veri[availability_365] <= 30, "0–30 Days",  
    veri[availability_365] <= 90, "31–90 Days",  
    veri[availability_365] <= 180, "91–180 Days",  
    veri[availability_365] <= 270, "181–270 Days",  
    veri[availability_365] <= 365, "271–365 Days",  
    "Unknown"  
)

// 2. Host Listing Grouping (Portfolio Scale Segmentation)
// Categorizes hosts based on their platform-level listing counts to isolate single, multi-property, and commercial operators.

Host Listing Group = 
SWITCH(
    TRUE(),
    'veri'[calculated_host_listings_count] = 1, "1 Listing",  
    'veri'[calculated_host_listings_count] <= 5, "2-5 Listings",  
    'veri'[calculated_host_listings_count] <= 10, "6-10 Listings",  
    'veri'[calculated_host_listings_count] <= 50, "11-50 Listings",  
    'veri'[calculated_host_listings_count] > 50, "50+ Listings"  
)
