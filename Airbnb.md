
## Airbnb Analysis

### Summary

### Data Processing

All data used in this analysis was downloaded from the [Inside Airbnb](http://insideairbnb.com/) website. Data files are available in the ['Get the Data'](http://insideairbnb.com/get-the-data.html) section of the website. This analysis uses the listings ('listings.csv.gz'), calendar ('calendar.csv.gz'), and reviews ('reviews.csv.gz') files.

Data files were downloaded from the [Inside Airbnb](http://insideairbnb.com/) website and placed in a local directory, with a child-folder structure of [Country]/[State]/[City]/[Data Scrape Date ('YYYY-MM-DD')]/[GZ File]. The directory structure used the city, state, and country names from the headers of each scraped city on the ['Get the Data'](http://insideairbnb.com/get-the-data.html) page of the [Inside Airbnb](http://insideairbnb.com/) website. All files for all scrapes of all cities were downloaded as of April 2, 2017, barring the December 2, 2015 scrape of New York City (this scrape contained a broken link for the calendar file). This data encompassed 136 data scrapes for 43 distinct cities. Older scrapes for each city can be removed to cut down on the data size substantially.

This code uses the directory structure of the data folder to construct the metadata for the scraped data. It is therefore imperative that the directory structure is properly set-up. The listings, calendar, and reviews files must retain their original file names.

For this first step in the data processing, the data is placed into a local SQLite database. The remaining analysis can either be performed on this SQLite database, or the data can be transferred to another store (e.g. a MySQL server on an AWS RDS instance). Total disk space for the gz files is about 7.95 GBs. Total disk space for the complete SQLite database is about 70 GBs, including the optional indices.


```r
library(RSQLite)
library(DBI)
```


```r
# Directory for data folder
data_directory <- 'C:/Airbnb'

# Directory and file for SQLite database
sqlite_directory <- 'C:/SQLite/Databases/Airbnb.sqlite3'
```

Connect to or create the SQLite database and build the tables if they do not exist


```r
# Connect to or create SQLite database
abnb_db <- dbConnect(SQLite(), sqlite_directory)
```


```r
# Create tables if they do not exist
calendar_create <- 'CREATE TABLE IF NOT EXISTS Calendar (
                        Calendar_ID INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL UNIQUE,
                        DataScrape_ID INTEGER NOT NULL,
                        ListingID INTEGER NOT NULL,
                        Date DATE NOT NULL,
                        Available VARCHAR NOT NULL,
                        Price REAL NOT NULL
                    );
                    '
dbExecute(abnb_db, calendar_create)
listings_create <- 'CREATE TABLE IF NOT EXISTS Listings (
                        Listings_ID INTEGER PRIMARY KEY AUTOINCREMENT UNIQUE NOT NULL,
                        DataScrape_ID INTEGER NOT NULL,
                        ID INTEGER NOT NULL,
                        ListingURL VARCHAR,
                        ScrapeID INTEGER,
                        LastSearched DATE,
                        LastScraped DATE,
                        Name TEXT,
                        Summary TEXT,
                        Space TEXT,
                        Description TEXT,
                        ExperiencesOffered TEXT,
                        NeighborhoodOverview TEXT,
                        Notes TEXT,
                        Transit TEXT,
                        Access TEXT,
                        Interaction TEXT,
                        HouseRules TEXT,
                        ThumbnailURL VARCHAR,
                        MediumURL VARCHAR,
                        PictureURL VARCHAR,
                        XLPictureURL VARCHAR,
                        HostID INTEGER,
                        HostURL VARCHAR,
                        HostName VARCHAR,
                        HostSince DATE,
                        HostLocation VARCHAR,
                        HostAbout TEXT,
                        HostResponseTime VARCHAR,
                        HostResponseRate REAL,
                        HostAcceptanceRate REAL,
                        HostIsSuperhost VARCHAR,
                        HostThumbnailURL VARCHAR,
                        HostPictureURL VARCHAR,
                        HostNeighborhood VARCHAR,
                        HostListingsCount INTEGER,
                        HostTotalListingsCount INTEGER,
                        HostVerifications VARCHAR,
                        HostHasProfilePic VARCHAR,
                        HostIdentityVerified VARCHAR,
                        Street VARCHAR,
                        Neighborhood VARCHAR,
                        NeighborhoodCleansed VARCHAR,
                        NeighborhoodGroupCleansed VARCHAR,
                        City VARCHAR,
                        State VARCHAR,
                        ZipCode VARCHAR,
                        Market VARCHAR,
                        SmartLocation VARCHAR,
                        CountryCode VARCHAR,
                        Country VARCHAR,
                        Latitude VARCHAR,
                        Longitude VARCHAR,
                        IsLocationExact VARCHAR,
                        PropertyType VARCHAR,
                        RoomType VARCHAR,
                        Accommodates INTEGER,
                        Bathrooms INTEGER,
                        Bedrooms INTEGER,
                        Beds INTEGER,
                        BedType VARCHAR,
                        Amenities VARCHAR,
                        SquareFeet REAL,
                        Price REAL,
                        WeeklyPrice REAL,
                        MonthlyPrice REAL,
                        SecurityDeposit REAL,
                        CleaningFee REAL,
                        GuestsIncluded INTEGER,
                        ExtraPeople REAL,
                        MinimumNights INTEGER,
                        MaximumNights INTEGER,
                        CalendarUpdated VARCHAR,
                        HasAvailability VARCHAR,
                        Availability30 INTEGER,
                        Availability60 INTEGER,
                        Availability90 INTEGER,
                        Availability365 INTEGER,
                        CalendarLastScraped DATE,
                        NumberOfReviews INTEGER,
                        FirstReview DATE,
                        LastReview DATE,
                        ReviewScoresRating INTEGER,
                        ReviewScoresAccuracy INTEGER,
                        ReviewScoresCleanliness INTEGER,
                        ReviewScoresCheckIn INTEGER,
                        ReviewScoresCommunication INTEGER,
                        ReviewScoresLocation INTEGER,
                        ReviewScoresValue INTEGER,
                        RequiresLicense VARCHAR,
                        License VARCHAR,
                        JurisdictionNames VARCHAR,
                        InstantBookable VARCHAR,
                        CancellationPolicy VARCHAR,
                        RequireGuestProfilePicture VARCHAR,
                        RequireGuestPhoneVerification VARCHAR,
                        RegionID INTEGER,
                        RegionName VARCHAR,
                        RegionParentID INTEGER,
                        CalculatedHostListingsCount INTEGER,
                        ReviewsPerMonth REAL
                    );
                    '
dbExecute(abnb_db, listings_create)
reviews_create <- 'CREATE TABLE IF NOT EXISTS Reviews (
                       Reviews_ID INTEGER PRIMARY KEY AUTOINCREMENT UNIQUE NOT NULL,
                       DataScrape_ID INTEGER NOT NULL,
                       ListingID INTEGER NOT NULL,
                       ID INTEGER NOT NULL,
                       Date DATE NOT NULL,
                       ReviewerID INTEGER NOT NULL,
                       ReviewerName VARCHAR NOT NULL,
                       Comments TEXT NOT NULL
                   );
                   '
dbExecute(abnb_db, reviews_create)
map_data_create <- 'CREATE TABLE IF NOT EXISTS Map_DataScrape (
                        DataScrape_ID INTEGER PRIMARY KEY AUTOINCREMENT UNIQUE NOT NULL,
                        Country VARCHAR NOT NULL,
                        State VARCHAR NOT NULL,
                        City VARCHAR NOT NULL,
                        DataScrapeDate DATE NOT NULL
                    );
                    '
dbExecute(abnb_db, map_data_create)
```

Add unique index to mapping table, to prevent duplication of scrapes in event of multiple runs


```r
idx_datascrape_unique <- 'CREATE UNIQUE INDEX IF NOT EXISTS idx_unique_scrape ON Map_DataScrape (
                              Country
                              ,State
                              ,City
                              ,DataScrapeDate
                          );
                          '
dbExecute(abnb_db, idx_datascrape_unique)
```

Investigate directory structure of data folder, and create scrape metadata from folder names


```r
dir_vec <- list.files(data_directory, recursive = TRUE)
dir_df <- t(data.frame(strsplit(dir_vec, split = '/'), stringsAsFactors = FALSE))
dir_df <- unique(dir_df[, 1:4])
colnames(dir_df) <- c('Country', 'State', 'City', 'DataScrapeDate')
rownames(dir_df) <- seq.int(1, nrow(dir_df))
head(dir_df, 10)
```

```
  Country         State        City        DataScrapeDate
1 "United States" "California" "San Diego" "2015-06-22"  
2 "United States" "California" "San Diego" "2016-07-07"  
```


```r
# Find max data scrape mapping ID already in the SQLite database
max_scrape <- dbExecute(abnb_db, 'SELECT MAX(DataScrape_ID) FROM Map_DataScrape;')
```

Define column name data frame for listings table (listings files differ from one city and scrape to another -- this step and several steps within the read code for the listings data must be completed in order to maintain consistency across different data files)


```r
# Possible column names for listings data from gz source files
listings_ds_colnames <- c('Listings_ID', 'DataScrape_ID', 'id', 'listing_url', 'scrape_id', 'last_searched',
                          'last_scraped', 'name', 'summary', 'space', 'description', 'experiences_offered',
                          'neighborhood_overview', 'notes', 'transit', 'access', 'interaction', 'house_rules',
                          'thumbnail_url', 'medium_url', 'picture_url', 'xl_picture_url', 'host_id',
                          'host_url', 'host_name', 'host_since', 'host_location', 'host_about',
                          'host_response_time', 'host_response_rate', 'host_acceptance_rate',
                          'host_is_superhost', 'host_thumbnail_url', 'host_picture_url', 'host_neighbourhood',
                          'host_listings_count', 'host_total_listings_count', 'host_verifications',
                          'host_has_profile_pic', 'host_identity_verified', 'street', 'neighbourhood',
                          'neighbourhood_cleansed', 'neighbourhood_group_cleansed', 'city', 'state',
                          'zipcode', 'market', 'smart_location', 'country_code', 'country', 'latitude',
                          'longitude', 'is_location_exact', 'property_type', 'room_type', 'accommodates',
                          'bathrooms', 'bedrooms', 'beds', 'bed_type', 'amenities', 'square_feet', 'price',
                          'weekly_price', 'monthly_price', 'security_deposit', 'cleaning_fee',
                          'guests_included', 'extra_people', 'minimum_nights', 'maximum_nights',
                          'calendar_updated', 'has_availability', 'availability_30', 'availability_60',
                          'availability_90', 'availability_365', 'calendar_last_scraped', 'number_of_reviews',
                          'first_review', 'last_review', 'review_scores_rating', 'review_scores_accuracy',
                          'review_scores_cleanliness', 'review_scores_checkin', 'review_scores_communication',
                          'review_scores_location', 'review_scores_value', 'requires_license', 'license',
                          'jurisdiction_names', 'instant_bookable', 'cancellation_policy',
                          'require_guest_profile_picture', 'require_guest_phone_verification', 'region_id',
                          'region_name', 'region_parent_id', 'calculated_host_listings_count',
                          'reviews_per_month')

# Column names for listings data from the SQLite database
listings_db_colnames <- c('Listings_ID', 'DataScrape_ID', 'ID', 'ListingURL', 'ScrapeID', 'LastSearched',
                          'LastScraped', 'Name', 'Summary', 'Space', 'Description', 'ExperiencesOffered',
                          'NeighborhoodOverview', 'Notes', 'Transit', 'Access', 'Interaction', 'HouseRules',
                          'ThumbnailURL', 'MediumURL', 'PictureURL', 'XLPictureURL', 'HostID',
                          'HostURL', 'HostName', 'HostSince', 'HostLocation', 'HostAbout',
                          'HostResponseTime', 'HostResponseRate', 'HostAcceptanceRate',
                          'HostIsSuperhost', 'HostThumbnailURL', 'HostPictureURL', 'HostNeighborhood',
                          'HostListingsCount', 'HostTotalListingsCount', 'HostVerifications',
                          'HostHasProfilePic', 'HostIdentityVerified', 'Street', 'Neighborhood',
                          'NeighborhoodCleansed', 'NeighborhoodGroupCleansed', 'City', 'State',
                          'ZipCode', 'Market', 'SmartLocation', 'CountryCode', 'Country', 'Latitude',
                          'Longitude', 'IsLocationExact', 'PropertyType', 'RoomType', 'Accommodates',
                          'Bathrooms', 'Bedrooms', 'Beds', 'BedType', 'Amenities', 'SquareFeet', 'Price',
                          'WeeklyPrice', 'MonthlyPrice', 'SecurityDeposit', 'CleaningFee',
                          'GuestsIncluded', 'ExtraPeople', 'MinimumNights', 'MaximumNights',
                          'CalendarUpdated', 'HasAvailability', 'Availability30', 'Availability60',
                          'Availability90', 'Availability365', 'CalendarLastScraped', 'NumberOfReviews',
                          'FirstReview', 'LastReview', 'ReviewScoresRating', 'ReviewScoresAccuracy',
                          'ReviewScoresCleanliness', 'ReviewScoresCheckIn', 'ReviewScoresCommunication',
                          'ReviewScoresLocation', 'ReviewScoresValue', 'RequiresLicense', 'License',
                          'JurisdictionNames', 'InstantBookable', 'CancellationPolicy',
                          'RequireGuestProfilePicture', 'RequireGuestPhoneVerification', 'RegionID',
                          'RegionName', 'RegionParentID', 'CalculatedHostListingsCount',
                          'ReviewsPerMonth')

# Combine gz file data source and SQLite column names for listings data into mapping data frame
listings_df_colnames <- data.frame(listings_ds_colnames, listings_db_colnames)
```

Loop through all directories (as defined in dir_df matrix), read all gz files, and populated data into the SQLite database


```r
# Read calendar data and populate into SQLite database
loop <- 1 + max_scrape
while (loop <= (max_scrape + nrow(dir_df))) {
    # Repair columns if broken
    calendar_temp <- readLines(paste(data_directory, '/', dir_df[loop, 1], '/',
                                     dir_df[loop, 2], '/', dir_df[loop, 3], '/',
                                     dir_df[loop, 4], '/calendar.csv.gz', sep = ''))
    # Replace two consecutive double quotes with one double quote and wrap currency figures in a double quote
    calendar_temp <- gsub('""', '"', gsub('([$][0-9,.]+)', '"\\1"', calendar_temp))
    # Write temp data to temporary file
    write(calendar_temp, file = 'temp.txt', ncolumns = 1)
    # Prepare calendar data frame
    calendar <- read.table('temp.txt', sep = ',', quote = '"', header = TRUE, comment.char = '',
                           stringsAsFactors = FALSE)
    # Remove NAs (interpreted as NULL)
    calendar[is.na(calendar)] <- ''
    # Create extra data rows
    cal_length <- nrow(calendar)
    id_vec_cal <- rep(NA, cal_length)
    map_id_vec_cal <- rep(loop, cal_length)
    # Bind data frame
    calendar <- cbind.data.frame(id_vec_cal, map_id_vec_cal, calendar)
    # Set column names
    names(calendar) <- c('Calendar_ID', 'DataScrape_ID', 'ListingID', 'Date', 'Available', 'Price')
    # Strip dollar signs and commas from prices
    calendar$Price <- gsub(',', '', gsub('\\$', '', calendar$Price))
    # Write calendar data frame to SQLite database
    dbWriteTable(abnb_db, name = 'Calendar', value = calendar, append = TRUE)
    
    # Increment loop
    loop <- loop + 1
}

# Remove temp file and large objects
if (file.exists('temp.txt')) {file.remove('temp.txt')}
if (exists('calendar_temp')) {rm(calendar_temp)}
if (exists('calendar')) {rm(calendar)}
```


```r
# Read listings data and populate into SQLite database
loop <- 1 + max_scrape
while (loop <= (max_scrape + nrow(dir_df))) {
    # Repair columns if broken
    listings_temp <- readLines(paste(data_directory, '/', dir_df[loop, 1], '/',
                                     dir_df[loop, 2], '/', dir_df[loop, 3], '/',
                                     dir_df[loop, 4], '/listings.csv.gz', sep = ''))
    # Fix quoting characters and replace two consecutive double quotes with a single quote
    listings_temp <- gsub(',\'"', ',"\'', gsub('""', '\'', listings_temp))
    # Write temp data to temporary file
    write(listings_temp, file = 'temp.txt', ncolumns = 1)
    # Prepare listings data frame
    listings <- read.table('temp.txt', sep = ',', quote = '"', header = TRUE, comment.char = '',
                           stringsAsFactors = FALSE)
    # Remove NAs (interpreted as NULL)
    listings[is.na(listings)] <- ''
    # Create extra data rows
    lst_length <- nrow(listings)
    id_vec_lst <- rep(NA, lst_length)
    map_id_vec_lst <- rep(loop, lst_length)
    # Add extra data rows to data frame
    listings['Listings_ID'] <- id_vec_lst
    listings['DataScrape_ID'] <- map_id_vec_lst
    # Set column names
    i <- 1
    while (i <= length(names(listings))) {
        names(listings)[i] <- as.character(listings_df_colnames$listings_db_colnames
                                           [match(names(listings)[i],
                                                  listings_df_colnames$listings_ds_colnames)])
        i <- i + 1
    }
    # Fill in missing columns with NA
    i <- 1
    while (i <= length(listings_df_colnames$listings_db_colnames)) {
        if (!is.element(listings_df_colnames$listings_db_colnames[i], names(listings))) {
            temp_vec <- rep(NA, lst_length)
            listings[as.character(listings_df_colnames$listings_db_colnames[i])] <- temp_vec
        }
        i <- i + 1
    }
    # Reorder data frame
    i <- 1
    name_vec <- vector(mode = 'integer')
    while (i <= length(listings_df_colnames$listings_db_colnames)) {
        name_vec[i] <- match(listings_df_colnames$listings_db_colnames[i], names(listings))
        i <- i + 1
    }
    listings <- listings[name_vec]
    # Strip dollar signs and commas from prices
    listings$Price <- gsub(',', '', gsub('\\$', '', listings$Price))
    listings$WeeklyPrice <- gsub(',', '', gsub('\\$', '', listings$WeeklyPrice))
    listings$MonthlyPrice <- gsub(',', '', gsub('\\$', '', listings$MonthlyPrice))
    listings$SecurityDeposit <- gsub(',', '', gsub('\\$', '', listings$SecurityDeposit))
    listings$CleaningFee <- gsub(',', '', gsub('\\$', '', listings$CleaningFee))
    listings$ExtraPeople <- gsub(',', '', gsub('\\$', '', listings$ExtraPeople))
    # Strip percent signs
    listings$HostResponseRate <- gsub('%', '', listings$HostResponseRate)
    listings$HostAcceptanceRate <- gsub('%', '', listings$HostAcceptanceRate)
    # Write listings data frame to SQLite database
    dbWriteTable(abnb_db, name = 'Listings', value = listings, append = TRUE)
    
    # Increment loop
    loop <- loop + 1
}

# Remove temp file and large objects
if (file.exists('temp.txt')) {file.remove('temp.txt')}
if (exists('listings_temp')) {rm(listings_temp)}
if (exists('listings')) {rm(listings)}
```


```r
# Read reviews data and populate into SQLite database
loop <- 1 + max_scrape
while (loop <= (max_scrape + nrow(dir_df))) {
    # Prepare reviews data frame
    reviews <- read.table(paste(data_directory, '/', dir_df[loop, 1], '/',
                                dir_df[loop, 2], '/', dir_df[loop, 3], '/',
                                dir_df[loop, 4], '/reviews.csv.gz', sep = ''),
                          sep = ',', quote = '"', header = TRUE, comment.char = '',
                          stringsAsFactors = FALSE)
    reviews[is.na(reviews)] <- ''
    # Create extra data rows
    rvw_length <- nrow(reviews)
    id_vec_rvw <- rep(NA, rvw_length)
    map_id_vec_rvw <- rep(loop, rvw_length)
    # Bind data frame
    reviews <- cbind.data.frame(id_vec_rvw, map_id_vec_rvw, reviews)
    # Set column names
    names(reviews) <- c('Reviews_ID', 'DataScrape_ID', 'ListingID', 'ID', 'Date', 'ReviewerID',
                        'ReviewerName', 'Comments')
    # Write reviews data frame to SQLite database
    dbWriteTable(abnb_db, name = 'Reviews', value = reviews, append = TRUE)
    
    # Increment loop
    loop <- loop + 1
}

# Remove large objects
if (exists('reviews')) {rm(reviews)}
```


```r
# Create data scrape mapping table
scrape_seq <- seq(1 + max_scrape, (max_scrape + nrow(dir_df)))
data_scrape <- cbind.data.frame(scrape_seq, dir_df)
# Set column names
colnames(data_scrape) <- c('DataScrape_ID', 'Country', 'State', 'City', 'DataScrapeDate')
# Write data scrape mapping data frame to SQLite database
dbWriteTable(abnb_db, name = 'Map_DataScrape', value = data_scrape, append = TRUE, row.names = FALSE)
```

**Optional:** Create database indices for faster querying


```r
# Data scrape ID indices
idx_datascrape_id <- 'CREATE INDEX IF NOT EXISTS idx_DataScrape_ID ON Map_DataScrape (
                          DataScrape_ID
                      );
                      '
dbExecute(abnb_db, idx_datascrape_id)
idx_datascrape_id_cal <- 'CREATE INDEX IF NOT EXISTS idx_DataScrape_ID_cal ON Calendar (
                              DataScrape_ID
                          );
                          '
dbExecute(abnb_db, idx_datascrape_id_cal)
idx_datascrape_id_lst <- 'CREATE INDEX IF NOT EXISTS idx_DataScrape_ID_lst ON Listings (
                              DataScrape_ID
                          );
                          '
dbExecute(abnb_db, idx_datascrape_id_lst)
idx_datascrape_id_rvw <- 'CREATE INDEX IF NOT EXISTS idx_DataScrape_ID_rvw ON Reviews (
                              DataScrape_ID
                          );
                          '
dbExecute(abnb_db, idx_datascrape_id_rvw)

# Listing ID indices
idx_listing_id_cal <- 'CREATE INDEX IF NOT EXISTS idx_Listing_ID_cal ON Calendar (
                           ListingID
                       );
                       '
dbExecute(abnb_db, idx_listing_id_cal)
idx_listing_id_lst <- 'CREATE INDEX IF NOT EXISTS idx_Listing_ID_lst ON Listings (
                           ID
                       );
                       '
dbExecute(abnb_db, idx_listing_id_lst)
idx_listing_id_rvw <- 'CREATE INDEX IF NOT EXISTS idx_Listing_ID_rvw ON Reviews (
                           ListingID
                       );
                       '
dbExecute(abnb_db, idx_listing_id_rvw)
```

Disconnect from the SQLite database


```r
# Disconnect from SQLite database
dbDisconnect(abnb_db)
```

### Analysis
