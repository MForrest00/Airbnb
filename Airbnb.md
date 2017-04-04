
## Airbnb Analysis

### Summary

### Data Processing (Overview and SQLite Data Transfer)

Investigate directory structure of data folder, and create scrape metadata from folder names


```r
# Directory for data folder
data_directory <- 'C:/Airbnb'

# Create data frame for data scrape metadata from folders in the data directory
dir_vec <- list.files(data_directory, recursive = TRUE)
dir_df <- t(data.frame(strsplit(dir_vec, split = '/'), stringsAsFactors = FALSE))
dir_df <- unique(dir_df[, 1:4])
colnames(dir_df) <- c('Country', 'State', 'City', 'DataScrapeDate')
rownames(dir_df) <- seq.int(1, nrow(dir_df))
```

All data used in this analysis was downloaded from the [Inside Airbnb](http://insideairbnb.com/) website. Data files are available in the ['Get the Data'](http://insideairbnb.com/get-the-data.html) section of the website. This analysis uses the listings ('listings.csv.gz'), calendar ('calendar.csv.gz'), and reviews ('reviews.csv.gz') files.

Data files were downloaded from the [Inside Airbnb](http://insideairbnb.com/) website and placed in a local directory, with a child-folder structure of [Country]/[State]/[City]/[Data Scrape Date ('YYYY-MM-DD')]/[GZ File]. The directory structure used the city, state, and country names from the headers of each scraped city on the ['Get the Data'](http://insideairbnb.com/get-the-data.html) page of the [Inside Airbnb](http://insideairbnb.com/) website. All files for all scrapes of all cities were downloaded as of March 15, 2017, barring the December 02, 2015 scrape of New York City (this scrape contained a broken link for the calendar file). This data encompassed 136 data scrapes for 43 distinct cities. Older scrapes for each city can be removed to cut down on the data size substantially.

This code uses the directory structure of the data folder to construct the metadata for the scraped data. It is therefore imperative that the directory structure is properly set-up. The listings, calendar, and reviews files must retain their original file names.

For this first step in the data processing, the data is placed into a local SQLite database. The remaining analysis can either be performed on this SQLite database, or the data can be transferred to another store (e.g. a MySQL server on an AWS RDS instance). Total disk space for the gz files (408 files total) is about 6.95 GBs. Total disk space for the complete SQLite database should end up around 71.6 GBs, including the optional indices.


```r
library(RSQLite)
library(DBI)
```


```r
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

Investigate the data scrape metadata


```r
# Return top 20 rows of data scrape metadata
head(dir_df, 20)
```

```
   Country     State              City              DataScrapeDate
1  "Australia" "New South Wales"  "Northern Rivers" "2016-04-02"  
2  "Australia" "New South Wales"  "Sydney"          "2015-05-10"  
3  "Australia" "New South Wales"  "Sydney"          "2015-09-03"  
4  "Australia" "New South Wales"  "Sydney"          "2015-10-02"  
5  "Australia" "New South Wales"  "Sydney"          "2015-12-03"  
6  "Australia" "New South Wales"  "Sydney"          "2016-01-03"  
7  "Australia" "New South Wales"  "Sydney"          "2016-12-04"  
8  "Australia" "New South Wales"  "Sydney"          "2017-03-03"  
9  "Australia" "Victoria"         "Melbourne"       "2015-07-18"  
10 "Australia" "Victoria"         "Melbourne"       "2015-09-03"  
11 "Australia" "Victoria"         "Melbourne"       "2015-10-02"  
12 "Australia" "Victoria"         "Melbourne"       "2015-12-03"  
13 "Australia" "Victoria"         "Melbourne"       "2016-01-03"  
14 "Australia" "Victoria"         "Melbourne"       "2016-09-04"  
15 "Australia" "Victoria"         "Melbourne"       "2016-12-05"  
16 "Austria"   "Vienna"           "Vienna"          "2015-07-18"  
17 "Belgium"   "Brussels"         "Brussels"        "2015-10-03"  
18 "Belgium"   "Flemish Region"   "Antwerp"         "2015-10-03"  
19 "Canada"    "British Columbia" "Vancouver"       "2015-11-07"  
20 "Canada"    "British Columbia" "Vancouver"       "2015-12-03"  
```


```r
# Find max data scrape mapping ID already in the SQLite database
max_scrape_temp <- dbSendQuery(abnb_db, 'SELECT MAX(DataScrape_ID) FROM Map_DataScrape;')
max_scrape <- dbFetch(max_scrape_temp)
dbClearResult(max_scrape_temp)
if (is.na(as.integer(max_scrape))) {max_scrape <- 0} else {max_scrape <- as.integer(max_scrape)}
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

Loop through all directories (as defined in dir_df matrix), read all gz files, and populate data into the SQLite database


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

### Data Processing (AWS MySQL RDS Instance Data Transfer)

Although merging all bz files into a SQLite database with the proper metadata enables significantly streamlined analysis, this approach towards data management carries significant downsides. The data takes up a large amount of space and is stored twice on disk (once in the assortment of gz files and once in the SQLite database). The analysis is limited by the processing power and memory of the user's computer. Finally, the SQLite database is serverless and therefore must be transferred to another user's computer in order to facilitate data sharing.

Transferring the data to a server-based RDBMS solves these problems. Below, the data from the SQLite database will be moved to a MySQL server on an AWS RDS instance owned by the analysis author. There is no need for the SQLite database to server as an intermediary, however. In this analysis, the data will be moved out of the SQLite database and into the MySQL server simply to reduce duplication of code. Below are documented the steps required to alter the code in the 'Data Processing' sections of this analysis to move the data straight from the bz files into the MySQL server.

1. Replace all table and index creation statements with the equivalent MySQL statements documented below.
2. Replace the statement used to create the max_scrape variable with the equivalent MySQL statement documented below.
3. Perform the following modifications to the read and populate blocks for the calendar, listings, reviews, and data scrape code blocks:
    + Remove all calls to dbWriteTable.
    + In place of the dbWriteTable calls, write the final dataframes to a text file. Make sure to replace all new line, carriage return, and tab characters in fields where these can occur (see statements below to identify the applicable fields).
    + Execute LOAD DATA LOCAL INFILE statements on the MySQL server using these files (see statements below for examples).
    + Some fields will require replacement logic to make the output compatible with MySQL data types (see statements below to identify the applicable fields).
    + Additional work may need to be done to handle differences in how MySQL handles NULL fields (concerning how R writes NA values to text files and how MySQL reads the resulting data, and how MySQL handles NULL values for auto-increment NOT NULL columns).


```r
library(RSQLite)
library(RMySQL)
library(DBI)
```

Connect to the SQLite and MySQL databases


```r
# Directory and file for SQLite database
sqlite_directory <- 'C:/SQLite/Databases/Airbnb.sqlite3'

# Connect to SQLite database
abnb_db_slt <- dbConnect(SQLite(), sqlite_directory)

# Load MySQL server address and log-in credentials (replace with your server address and log-in credentials)
mysql_server_address <- readLines('C:/Credentials/AWS MySQL Airbnb Database/serverAddress.txt')
mysql_user_name <- readLines('C:/Credentials/AWS MySQL Airbnb Database/userName.txt')
mysql_password <- readLines('C:/Credentials/AWS MySQL Airbnb Database/password.txt')

# Connect to MySQL database -- this instance has been launched with a database named 'airbnb'
abnb_db_mys <- dbConnect(MySQL(), dbname = 'airbnb', username = mysql_user_name, password = mysql_password,
                         host = mysql_server_address)
```

Create the MySQL tables if they do not exist


```r
# Create tables if they do not exist
calendar_create <- 'CREATE TABLE IF NOT EXISTS airbnb.Calendar (
                        Calendar_ID INT NOT NULL AUTO_INCREMENT,
                        DataScrape_ID INT NOT NULL,
                        ListingID INT NOT NULL,
                        Date DATE NOT NULL,
                        Available VARCHAR(1) NOT NULL,
                        Price DECIMAL(8, 2),
                        PRIMARY KEY (Calendar_ID),
                        UNIQUE INDEX Calendar_ID_UNIQUE (Calendar_ID ASC)
                    );
                    '
dbExecute(abnb_db_mys, calendar_create)
listings_create <- 'CREATE TABLE IF NOT EXISTS airbnb.Listings (
                        Listings_ID INT NOT NULL AUTO_INCREMENT,
                        DataScrape_ID INT NOT NULL,
                        ID INT NOT NULL,
                        ListingURL VARCHAR(45),
                        ScrapeID INT,
                        LastSearched DATE,
                        LastScraped DATE,
                        Name VARCHAR(120),
                        Summary VARCHAR(1500),
                        Space VARCHAR(2000),
                        Description VARCHAR(2500),
                        ExperiencesOffered VARCHAR(45),
                        NeighborhoodOverview VARCHAR(2000),
                        Notes VARCHAR(1200),
                        Transit VARCHAR(1500),
                        Access VARCHAR(1200),
                        Interaction VARCHAR(1200),
                        HouseRules VARCHAR(1500),
                        ThumbnailURL VARCHAR(150),
                        MediumURL VARCHAR(150),
                        PictureURL VARCHAR(150),
                        XLPictureURL VARCHAR(150),
                        HostID INT,
                        HostURL VARCHAR(120),
                        HostName VARCHAR(120),
                        HostSince DATE,
                        HostLocation VARCHAR(200),
                        HostAbout VARCHAR(6000),
                        HostResponseTime VARCHAR(45),
                        HostResponseRate INT,
                        HostAcceptanceRate INT,
                        HostIsSuperhost VARCHAR(1),
                        HostThumbnailURL VARCHAR(150),
                        HostPictureURL VARCHAR(150),
                        HostNeighborhood VARCHAR(200),
                        HostListingsCount INT,
                        HostTotalListingsCount INT,
                        HostVerifications VARCHAR(120),
                        HostHasProfilePic VARCHAR(1),
                        HostIdentityVerified VARCHAR(1),
                        Street VARCHAR(120),
                        Neighborhood VARCHAR(250),
                        NeighborhoodCleansed VARCHAR(100),
                        NeighborhoodGroupCleansed VARCHAR(45),
                        City VARCHAR(100),
                        State VARCHAR(45),
                        ZipCode VARCHAR(45),
                        Market VARCHAR(45),
                        SmartLocation VARCHAR(100),
                        CountryCode VARCHAR(2),
                        Country VARCHAR(45),
                        Latitude VARCHAR(45),
                        Longitude VARCHAR(45),
                        IsLocationExact VARCHAR(1),
                        PropertyType VARCHAR(45),
                        RoomType VARCHAR(45),
                        Accommodates INT,
                        Bathrooms INT,
                        Bedrooms INT,
                        Beds INT,
                        BedType VARCHAR(45),
                        Amenities VARCHAR(1000),
                        SquareFeet DECIMAL(8, 2),
                        Price DECIMAL(8, 2),
                        WeeklyPrice DECIMAL(8, 2),
                        MonthlyPrice DECIMAL(8, 2),
                        SecurityDeposit DECIMAL(8, 2),
                        CleaningFee DECIMAL(8, 2),
                        GuestsIncluded INT,
                        ExtraPeople DECIMAL(8, 2),
                        MinimumNights INT,
                        MaximumNights INT,
                        CalendarUpdated VARCHAR(45),
                        HasAvailability VARCHAR(1),
                        Availability30 INT,
                        Availability60 INT,
                        Availability90 INT,
                        Availability365 INT,
                        CalendarLastScraped DATE,
                        NumberOfReviews INT,
                        FirstReview DATE,
                        LastReview DATE,
                        ReviewScoresRating INT,
                        ReviewScoresAccuracy INT,
                        ReviewScoresCleanliness INT,
                        ReviewScoresCheckIn INT,
                        ReviewScoresCommunication INT,
                        ReviewScoresLocation INT,
                        ReviewScoresValue INT,
                        RequiresLicense VARCHAR(1),
                        License VARCHAR(45),
                        JurisdictionNames VARCHAR(45),
                        InstantBookable VARCHAR(1),
                        CancellationPolicy VARCHAR(45),
                        RequireGuestProfilePicture VARCHAR(1),
                        RequireGuestPhoneVerification VARCHAR(1),
                        RegionID INT,
                        RegionName VARCHAR(45),
                        RegionParentID INT,
                        CalculatedHostListingsCount INT,
                        ReviewsPerMonth DECIMAL(4, 2),
                        PRIMARY KEY (Listings_ID),
                        UNIQUE INDEX Listings_ID_UNIQUE (Listings_ID ASC)
                    );
                    '
dbExecute(abnb_db_mys, listings_create)
reviews_create <- 'CREATE TABLE IF NOT EXISTS airbnb.Reviews (
                       Reviews_ID INT NOT NULL AUTO_INCREMENT,
                       DataScrape_ID INT NOT NULL,
                       ListingID INT NOT NULL,
                       ID INT NOT NULL,
                       Date DATE NOT NULL,
                       ReviewerID INT NOT NULL,
                       ReviewerName VARCHAR(300) NOT NULL,
                       Comments VARCHAR(14000) NOT NULL,
                       PRIMARY KEY (Reviews_ID),
                       UNIQUE INDEX Reviews_ID_UNIQUE (Reviews_ID ASC)
                   );
                   '
dbExecute(abnb_db_mys, reviews_create)
map_data_create <- 'CREATE TABLE IF NOT EXISTS airbnb.Map_DataScrape (
                        DataScrape_ID INT NOT NULL AUTO_INCREMENT,
                        Country VARCHAR(25) NOT NULL,
                        State VARCHAR(35) NOT NULL,
                        City VARCHAR(25) NOT NULL,
                        DataScrapeDate DATE NOT NULL,
                        PRIMARY KEY (DataScrape_ID),
                        UNIQUE INDEX DataScrape_ID_UNIQUE (DataScrape_ID ASC)
                    );
                    '
dbExecute(abnb_db_mys, map_data_create)
```

Add unique index to mapping table, to prevent duplication of scrapes in event of multiple runs  
MySQL has no CREATE INDEX IF NOT EXISTS functionality, so do not run this code chunk if the index already exists.


```r
idx_datascrape_unique <- 'CREATE UNIQUE INDEX idx_unique_scrape ON airbnb.Map_DataScrape (
                              Country
                              ,State
                              ,City
                              ,DataScrapeDate
                          );
                          '
dbExecute(abnb_db_mys, idx_datascrape_unique)
```


```r
# Find max data scrape mapping ID already in the MySQL database
max_scrape_temp <- dbSendQuery(abnb_db_mys, 'SELECT MAX(DataScrape_ID) FROM airbnb.Map_DataScrape;')
max_scrape <- dbFetch(max_scrape_temp)
dbClearResult(max_scrape_temp)
if (is.na(as.integer(max_scrape))) {max_scrape <- 0} else {max_scrape <- as.integer(max_scrape)}
```

The process of moving data from the SQLite database into the MySQL server will involve writing the SQLite data to a text file and then calling a LOAD DATA LOCAL INFILE command on the MySQL server which will read the data in the text file. The data will be transferred in batches to allow us to pick up where we left off if we only want to transfer some of the data or if the transfer process fails in the middle of the loop.

First, we must find how many rows are in each of the three large tables (Calendar, Listings, and Reviews) in the SQLite database. This will tell us how long to run our loops. Then we can begin writing the data from the SQLite database and reading it into the MySQL server.


```r
# Find row count of large tables
table_rows_temp <- dbSendQuery(abnb_db_slt, 'SELECT COUNT(*) AS RowCount FROM Calendar
                                             UNION ALL
                                             SELECT COUNT(*) AS RowCount FROM Listings
                                             UNION ALL
                                             SELECT COUNT(*) AS RowCount FROM Reviews;')
table_rows <- dbFetch(table_rows_temp)
invisible(dbClearResult(table_rows_temp))
rownames(table_rows) <- c('Calendar', 'Listings', 'Reviews')
table_rows
```

```
          RowCount
Calendar 702590690
Listings   1924970
Reviews   25220866
```

Because the Calendar table is narrow, we can transfer it in batches of half a million. The wider Listings and Reviews tables will be transferred in batches of 100,000.


```r
work_dir_stem <- getwd()
```


```r
loop <- 0
batch_size <- 500000
work_dir <- paste(work_dir_stem, '/calendar_transf.txt', sep = '')
while (loop < table_rows[1, 1]) {
    # Query Calendar table
    calendar_sql <- 'SELECT
                         Calendar_ID
                         ,DataScrape_ID
                         ,ListingID
                         ,Date
                         ,Available
                         ,CASE WHEN Price = \'\' THEN \'NULL\' ELSE CAST(Price AS TEXT) END AS Price
                     FROM Calendar
                     LIMIT '
    calendar_rows_temp <- dbSendQuery(abnb_db_slt, paste(calendar_sql, format(loop, scientific = FALSE), ', ',
                                                         format(batch_size, scientific = FALSE), ';',
                                                         sep = ''))
    calendar_rows <- dbFetch(calendar_rows_temp)
    dbClearResult(calendar_rows_temp)
    
    # Write results to temp file
    write.table(calendar_rows, file = 'calendar_transf.txt', quote = FALSE, sep = '|', row.names = FALSE,
                col.names = FALSE)
    
    # Read results into MySQL from temp file
    calendar_load_stem1 <- 'LOAD DATA LOCAL INFILE \''
    calendar_load_stem2 <- '\' INTO TABLE airbnb.Calendar
                            FIELDS TERMINATED BY \'|\'
                            LINES TERMINATED BY \'\r\n\'
                            (Calendar_ID, DataScrape_ID, ListingID, Date, Available, @var1)
                            SET Price = IF(@var1 = \'NULL\', NULL, @var1);'
    calendar_load <- dbSendStatement(abnb_db_mys, paste(calendar_load_stem1, work_dir, calendar_load_stem2,
                                                        sep = ''))
    dbClearResult(calendar_load)
    
    loop <- loop + batch_size
}

# Remove temp file and large objects
if (file.exists('calendar_transf.txt')) {file.remove('calendar_transf.txt')}
if (exists('calendar_rows')) {rm(calendar_rows)}
```









```r
loop <- 0
batch_size <- 100000
work_dir <- paste(work_dir_stem, '/reviews_transf.txt', sep = '')
while (loop < table_rows[3, 1]) {
    # Query Reviews table -- replace all characters that might interfere with data transfer
    reviews_sql <- 'SELECT
                        Reviews_ID
                        ,DataScrape_ID
                        ,ListingID
                        ,ID
                        ,Date
                        ,ReviewerID
                        ,REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(ReviewerName,
                            CHAR(9), \' \'), CHAR(10), \' \'), CHAR(11), \'\'), CHAR(13), \' \'),
                            CHAR(8), \'\'), \'|\', \'-\'), CHAR(92), \'-\') AS ReviewerName
                        ,REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(REPLACE(Comments,
                            CHAR(9), \' \'), CHAR(10), \' \'), CHAR(11), \'\'), CHAR(13), \' \'),
                            CHAR(8), \'\'), \'|\', \'-\'), CHAR(92), \'-\') AS Comments
                    FROM Reviews
                    LIMIT '
    reviews_rows_temp <- dbSendQuery(abnb_db_slt, paste(reviews_sql, format(loop, scientific = FALSE), ', ',
                                                         format(batch_size, scientific = FALSE), ';',
                                                         sep = ''))
    reviews_rows <- dbFetch(reviews_rows_temp)
    dbClearResult(reviews_rows_temp)
    
    # Write results to temp file
    write.table(reviews_rows, file = 'reviews_transf.txt', quote = FALSE, sep = '|', row.names = FALSE,
                col.names = FALSE)
    
    # Read results into MySQL from temp file
    reviews_load_stem1 <- 'LOAD DATA LOCAL INFILE \''
    reviews_load_stem2 <- '\' INTO TABLE airbnb.Reviews
                           FIELDS TERMINATED BY \'|\'
                           LINES TERMINATED BY \'\r\n\';'
    reviews_load <- dbSendStatement(abnb_db_mys, paste(reviews_load_stem1, work_dir, reviews_load_stem2,
                                                       sep = ''))
    dbClearResult(reviews_load)
    
    loop <- loop + batch_size
}

# Remove temp file and large objects
if (file.exists('reviews_transf.txt')) {file.remove('reviews_transf.txt')}
if (exists('reviews_rows')) {rm(reviews_rows)}
```


```r
# Query all content from data scrape mapping table in SQLite
scrape_temp <- dbSendQuery(abnb_db_slt, 'SELECT * FROM Map_DataScrape;')
scrape <- dbFetch(scrape_temp)
dbClearResult(scrape_temp)

# Write results to temp file
write.table(scrape, file = 'scrape_transf.txt', quote = FALSE, sep = '|', row.names = FALSE,
            col.names = FALSE)
work_dir <- paste(work_dir_stem, '/scrape_transf.txt', sep = '')

# Read results into MySQL from temp file
scrape_load_stem1 <- 'LOAD DATA LOCAL INFILE \''
scrape_load_stem2 <- '\' INTO TABLE airbnb.Map_DataScrape
                      FIELDS TERMINATED BY \'|\'
                      LINES TERMINATED BY \'\n\';'
scrape_load <- dbSendStatement(abnb_db_mys, paste(scrape_load_stem1, work_dir, scrape_load_stem2, sep = ''))
dbClearResult(scrape_load)

# Remove temp file
if (file.exists('scrape_transf.txt')) {file.remove('scrape_transf.txt')}
```










Disconnect from the SQLite and MySQL databases


```r
# Disconnect from SQLite database
dbDisconnect(abnb_db_slt)
dbDisconnect(abnb_db_mys)
```

### Analysis
