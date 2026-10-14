
# Fields and their meanings in *listings.csv*


## 📝 Listing Identification & Scrape Metadata

| Field | Meaning |
|---|---|
| `id` | Airbnb's unique identifier for the listing. |
| `listing_url` | URL of the listing page on Airbnb. |
| `scrape_id` | Unique identifier for the scrape run (e.g., `20260920135913` = date/time of scrape). |
| `last_scraped` | Date the listing data was last scraped (e.g., `2026-09-25`). |
| `source` | Source of the listing record - typically `city scrape` (found in the current scrape) or `previous scrape` (carried over from an earlier scrape). |

## 👤 Host Information

| Field | Meaning |
|---|---|
| `host_id` | Airbnb's unique identifier for the host/user. |
| `host_url` | URL to the host's Airbnb profile. |
| `host_profile_id` | Unique identifier for the host's public profile (distinct from the internal user ID). |
| `host_profile_url` | URL to the host's public profile page. |
| `host_name` | Host's first name (free text). |
| `host_since` | Date the host registered on Airbnb (or became a host). |
| `hosts_time_as_user_years` / `hosts_time_as_user_months` | Length of time the host has been an Airbnb user (years + months). |
| `hosts_time_as_host_years` / `hosts_time_as_host_months` | Length of time the host has been hosting (years + months). |
| `host_location` | Host's self-reported location. |
| `host_about` | Host's self-description. |
| `host_response_time` | Time interval between a guest inquiry and the host's response. |
| `host_response_rate` | Percentage of inquiries the host responds to. |
| `host_acceptance_rate` | Rate at which the host accepts booking requests. |
| `host_is_superhost` | Whether the host is a Superhost (`t`/`f`). |
| `host_thumbnail_url` | URL to a thumbnail of the host's profile picture. |
| `host_picture_url` | URL to the full-size host profile picture. |
| `host_neighbourhood` | Host's stated neighbourhood. |
| `host_listings_count` | Number of listings the host has (self-reported). |
| `host_total_listings_count` | Total number of listings the host has (may include other platforms or historical counts). |
| `host_verifications` | Verification methods the host has completed (e.g., email, phone). |
| `host_has_profile_pic` | Whether the host has a profile picture (`t`/`f`). |
| `host_identity_verified` | Whether the host's identity has been verified (`t`/`f`). |

## 📍 Location & Neighbourhood

| Field | Meaning |
|---|---|
| `neighbourhood` | Free-text neighbourhood name (often empty). |
| `neighbourhood_cleansed` | Neighbourhood derived by Inside Airbnb by geocoding the listing's coordinates against public shapefiles. |
| `neighbourhood_group_cleansed` | Higher-level neighbourhood group (e.g., district). |
| `latitude` / `longitude` | Geographic coordinates of the listing (WGS84). |

## 🏠 Property Details

| Field | Meaning |
|---|---|
| `property_type` | Self-selected property type (e.g., "Entire rental unit"). |
| `room_type` | Category of room: `Entire home/apt`, `Private room`, `Shared room`, or `Hotel room`. |
| `accommodates` | Maximum number of guests the listing can host. |
| `bathrooms` | Number of bathrooms (may be blank if only `bathrooms_text` is provided). |
| `bathrooms_text` | Text description of bathrooms (e.g., "1 bath"). |
| `bedrooms` | Number of bedrooms. |
| `beds` | Number of beds. |
| `amenities` | JSON-like list of amenities provided (e.g., Washer, Wifi). |

## 💰 Pricing & Booking Rules

| Field | Meaning |
|---|---|
| `price` | Nightly price in local currency (whole dollars). |
| `price_quote_checkin_date` | Check-in date used for the price quote (usually the scrape date). |
| `price_quote_checkout_date` | Check-out date used for the price quote. |
| `price_quote_total_price` | Total price for the quoted stay (including fees). |
| `price_quote_price_per_night` | Price per night derived from the quote. |
| `price_quote_raw` | Raw price data from the quote (often JSON). |
| `minimum_nights` | Minimum nights required per booking. |
| `maximum_nights` | Maximum nights allowed per booking. |
| `minimum_minimum_nights` | Smallest `minimum_nights` value found in the calendar over the next 365 nights. |
| `maximum_minimum_nights` | Largest `minimum_nights` value in the calendar over the next 365 nights. |
| `minimum_maximum_nights` | Smallest `maximum_nights` value in the calendar over the next 365 nights. |
| `maximum_maximum_nights` | Largest `maximum_nights` value in the calendar over the next 365 nights. |
| `minimum_nights_avg_ntm` | Average `minimum_nights` over the next 365 nights. |
| `maximum_nights_avg_ntm` | Average `maximum_nights` over the next 365 nights. |
| `instant_bookable` | Whether guests can book instantly without host approval (`t`/`f`). |

## 📅 Availability

| Field | Meaning |
|---|---|
| `has_availability` | Whether the listing has any availability (`t`/`f`). |
| `availability_30` | Number of nights available in the next 30 days. |
| `availability_60` | Number of nights available in the next 60 days. |
| `availability_90` | Number of nights available in the next 90 days. |
| `availability_365` | Number of nights available in the next 365 days. |
| `calendar_updated` | Date the calendar was last updated (often blank). |
| `calendar_last_scraped` | Date the calendar data was last scraped. |
| `availability_eoy` | Availability at end of year (derived from calendar). |

## ⭐ Reviews & Ratings

| Field | Meaning |
|---|---|
| `number_of_reviews` | Total number of reviews the listing has received. |
| `number_of_reviews_ltm` | Reviews received in the last 12 months. |
| `number_of_reviews_l30d` | Reviews received in the last 30 days. |
| `number_of_reviews_ly` | Reviews received in the last year (may differ from `_ltm`). |
| `first_review` | Date of the first (oldest) review. |
| `last_review` | Date of the most recent review. |
| `review_scores_rating` | Overall rating score (out of 5). |
| `review_scores_accuracy` | Accuracy rating. |
| `review_scores_cleanliness` | Cleanliness rating. |
| `review_scores_checkin` | Check-in experience rating. |
| `review_scores_communication` | Communication rating. |
| `review_scores_location` | Location rating. |
| `review_scores_value` | Value-for-money rating. |
| `reviews_per_month` | Average number of reviews per month over the listing's lifetime. |

## 📊 Derived & Calculated Fields

These are not scraped from Airbnb but computed by Inside Airbnb.

| Field | Meaning |
|---|---|
| `neighbourhood_cleansed` | Neighbourhood derived from coordinates. |
| `estimated_occupancy_l365d` | Estimated number of nights booked in the last 365 days (based on review frequency). |
| `estimated_revenue_l365d` | Estimated revenue over the last 365 days (occupancy × nightly price). |
| `calculated_host_listings_count` | Number of listings the host has in the current scrape (city-wide). |
| `calculated_host_listings_count_entire_homes` | Number of entire-home listings the host has. |
| `calculated_host_listings_count_private_rooms` | Number of private-room listings the host has. |
| `calculated_host_listings_count_shared_rooms` | Number of shared-room listings the host has. |
| `reviews_per_month` | Average reviews per month (derived from review history). |

## 📜 Legal & Administrative

| Field | Meaning |
|---|---|
| `license` | License or registration number, if provided by the host (often blank). |


# Fields and their meanings in *reviews.csv*

## 📝 Review Identification

| Field | Meaning |
|---|---|
| `listing_id` | Airbnb listing ID that the review belongs to. This is a foreign key linking to the `id` field in `listings.csv`. |
| `id` | Airbnb's unique identifier for the review itself. |
| `date` | Date the review was posted, in `YYYY-MM-DD` format. |

## 👤 Reviewer Information

| Field | Meaning |
|---|---|
| `reviewer_id` | Airbnb's unique identifier for the user who wrote the review. |
| `reviewer_name` | First name of the reviewer (free text). |

## 💬 Review Content

| Field | Meaning |
|---|---|
| `comments` | The full text of the review left by the guest. May contain line breaks, punctuation, and non-English characters. |

## 🔗 Relationship to Other Files

| Field | Meaning |
|---|---|
| `listing_id` | Use this to join reviews to listings. It matches `id` in `listings.csv`. |
| `reviewer_id` | Can be used to identify repeat reviewers or link to other Airbnb user data, if available. |

## 🧾 Example Record Interpretation

| Column | Value |
|---|---|
| `listing_id` | `72571` |
| `id` | `188791` |
| `date` | `2011-02-26` |
| `reviewer_id` | `380778` |
| `reviewer_name` | `Katherine` |
| `comments` | `Brenden is really helpful to us. We had a really pleasant stay in this place, the location is excellent, convenient, and has real character. We definitely will come back to stay there again next time when we go to HongKong!` |

This record means that reviewer Katherine (`reviewer_id = 380778`) posted review `188791` on `2011-02-26` for listing `72571`. The review text praises the host Brendan, the location, convenience, and character of the place.
