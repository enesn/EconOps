# Step 6: Providing authenticated data access to co-authors, RAs, and external researchers [DL]
Some datasets are proprietary, which restricts redistribution, and even for semi-public sources like PSID, researchers are typically not allowed to share raw microdata. So replication repositories cannot include full input datasets or direct links to them. Instead, access can be granted to researchers who meet data use agreement requirements.

A simple implementation is to store the data on Dropbox and have each project access it through a shared link that is available only to authenticated users, with the option to revoke the link once the work is completed. However, this approach still has important limitations.

This approach treats access control as something we can manage with a simple shared link, when in reality research data governance requires auditable and revocable permissions. Dropbox links are too coarse: once a file is shared, it can be copied, forwarded, or cached outside your control. More importantly, it also lacks strong identity binding, so we cannot reliably enforce who is accessing what, when, or under which agreement terms—something that datasets like PSID explicitly require. 

So what we need instead is a small authentication system that ties access to verified user identities, enforces time-bounded and purpose-specific permissions, and so access is both controlled and accountable rather than informally distributed through URLs.

### How it works

- We create a dedicated Dropbox app for each dataset in our data layer. Each app crates a specific folder inside our Dropbox account, and access to that folder is governed through authentication.
- Using the app’s control panel, we can define and adjust the permissions associated with each token, and revoke access at any time if needed.
- We maintain a small Python/R script that can generate either a short-lived access token (valid for four hours) or a refresh token that remains active until explicitly revoked from the Dropbox panel.
- Anyone with a valid token is able to reproduce and run the project within the project layer.

### Creating a Dropbox app

You create a Dropbox app through the Dropbox Developer Console: https://www.dropbox.com/developers/apps. In brief, you sign in to Dropbox, create a new app, and define its scope (such as full Dropbox access or a single app folder as in our case). Dropbox then generates an app key and secret, which you use to implement OAuth-based authentication in your system. After that, you configure permissions, optionally set redirect URIs for web flows, and use the OAuth flow to let users grant access so your application can securely read/write to the designated folder.

### Authentication script

**This script is used by the corresponding author.** It sets up Dropbox OAuth authentication and a data-loading pipeline. It lets us choose between a short-lived access token (for external replicators) or a long-lived refresh token (for RAs, collaborators), then runs a browser-based OAuth flow to generate an authorization code. As the owner of the account, you use this authorization code to generate token credentials and prints them for reuse. After authentication, it tests access by listing files in the user’s Dropbox app folder and handles any authentication or API errors:

    
 ```r
    # =====================================================================
# PACKAGES
# =====================================================================
library(httr)
library(jsonlite)

# =====================================================================
# CONFIGURATION
# =====================================================================
APP_KEY <- "key"
APP_SECRET <- "secret"

# OPTIONS:
# "refresh"     -> permanent backend pipelines
# "short_lived" -> external replicators (4-hour expiry)
TOKEN_MODE <- "refresh"

# =====================================================================
# STEP 1: TOKEN GENERATION FLOW
# =====================================================================

access_type <- ifelse(TOKEN_MODE == "refresh", "offline", "online")

cat(sprintf("▶ Initializing token manager in [%s] mode...\n", toupper(TOKEN_MODE)))

# Dropbox OAuth endpoints
auth_url <- "https://www.dropbox.com/oauth2/authorize"

query <- list(
  client_id = APP_KEY,
  response_type = "code",
  token_access_type = access_type
)

authorize_url <- modify_url(auth_url, query = query)

cat("\n1. Open this URL in your browser to authorize access:\n")
cat(authorize_url, "\n")

auth_code <- readline("\n2. Enter the authorization code here: ")
auth_code <- trimws(auth_code)

# Exchange authorization code for token
token_response <- POST(
  url = "https://api.dropboxapi.com/oauth2/token",
  authenticate(APP_KEY, APP_SECRET),
  body = list(
    code = auth_code,
    grant_type = "authorization_code"
  ),
  encode = "form"
)

oauth_result <- content(token_response, as = "parsed", type = "application/json")

# =====================================================================
# CREDENTIAL EXPORT
# =====================================================================
cat("\n=================== CREDENTIAL EXPORT ===================\n")

if (TOKEN_MODE == "refresh") {
  cat(sprintf('DROPBOX_REFRESH_TOKEN = "%s"\n', oauth_result$refresh_token))
  cat("\n✓ Save this permanent Refresh Token for automated pipelines.\n")
} else {
  cat(sprintf('DROPBOX_ACCESS_TOKEN = "%s"\n', oauth_result$access_token))
  cat("\n⚠️ Save this token. It will expire in ~4 hours.\n")
}

cat("=========================================================\n\n")

# =====================================================================
# STEP 2: VERIFY CONNECTION
# =====================================================================
cat("Testing generated credentials against the data layer...\n")

tryCatch({

  if (TOKEN_MODE == "short_lived") {

    dbx_token <- oauth_result$access_token

  } else {

    # A refresh token cannot authenticate API calls directly; it must be
    # exchanged for a short-lived access token first (same as a pipeline would).
    refresh_response <- POST(
      url = "https://api.dropboxapi.com/oauth2/token",
      authenticate(APP_KEY, APP_SECRET),
      body = list(
        grant_type = "refresh_token",
        refresh_token = oauth_result$refresh_token
      ),
      encode = "form"
    )

    refresh_result <- content(refresh_response, as = "parsed", type = "application/json")

    if (is.null(refresh_result$access_token)) {
      stop(sprintf("Refresh token exchange failed: %s",
                   paste(unlist(refresh_result), collapse = " ")))
    }

    dbx_token <- refresh_result$access_token
  }

  # Call Dropbox API: list folder
  res <- POST(
    url = "https://api.dropboxapi.com/2/files/list_folder",
    add_headers(
      Authorization = paste("Bearer", dbx_token),
      "Content-Type" = "application/json"
    ),
    body = toJSON(list(path = ""), auto_unbox = TRUE)
  )

  parsed <- content(res, as = "parsed", type = "application/json")

  if (!is.null(parsed$entries)) {

    cat("✅ Connection Successful! Verified data layer contents:\n\n")

    file_count <- 0

    for (entry in parsed$entries) {
      if (!is.null(entry$`.tag`) && entry$`.tag` == "file") {
        file_count <- file_count + 1
        cat(sprintf("  📄 %s\n", entry$name))
      }
    }

    if (file_count == 0) {
      cat("  (Folder authenticated successfully, but contains no files.)\n")
    }

  } else {
    cat("\n❌ Dropbox API returned an error:\n")
    cat(sprintf("  %s\n", ifelse(is.null(parsed$error_summary),
                                 paste(unlist(parsed), collapse = " "),
                                 parsed$error_summary)))
  }

}, error = function(e) {
  cat("\n❌ Error during Dropbox API call:\n")
  cat(e$message, "\n")
})
```
    

### User script to read data

**This script is the data import script included in the project repository (see Step 2) .** It connects Python/R to Dropbox using either a short-lived access token or a long-lived refresh token, then uses that authentication to inspect and read files stored in a Dropbox app folder. It first checks the token format: if it starts with `sl.`, it treats it as a temporary access token and connects directly; if it starts with `ds.` (or looks like a refresh token), it uses the app key and secret to authenticate through OAuth refresh flow. After establishing the client, it lists files in the root Dropbox directory and prints metadata like filename, size, and path, while handling authentication and API errors in a structured way.

It also defines a universal `read_dropbox_file()` function that streams files directly from Dropbox into memory and converts them into usable data formats. Depending on the file extension, it loads Parquet files via DuckDB, CSV files via pandas with automatic delimiter detection, Excel files via `read_excel`, and JSON files via `read_json`. If the format is unsupported, it returns raw bytes. Finally, it demonstrates usage by reading a CSV file directly from Dropbox, enabling zero-disk, authenticated access to research data.


    
```r

# ============================================================
# LIBRARIES
# ============================================================
library(httr2)
library(jsonlite)
library(readr)
library(readxl)
library(arrow)
library(duckdb)
library(DBI)

# ============================================================
# CONFIGURATION
# ============================================================
KEY    <- "fi2oqjuko41oi1r"
SECRET <- "ozzb1af2uudrmtl"

TOKEN <- "EA-pJnCLy_IAAAAAAAAAAdorIg2xmQkg24_kJDjpTEBUkh7_K2pHQ53a-NeWFtlb"

# ============================================================
# AUTH HANDLING
# ============================================================
cat("Analyzing token signature...\n")

# Token type is determined by behavior, not by format: Dropbox does not
# guarantee token prefixes, so we probe the API instead of parsing the string.
works_as_access_token <- function(tok) {
  resp <- request("https://api.dropboxapi.com/2/check/user") |>
    req_method("POST") |>
    req_headers(
      Authorization = paste("Bearer", tok),
      "Content-Type" = "application/json"
    ) |>
    req_body_json(list(query = "ping")) |>
    req_error(is_error = function(resp) FALSE) |>
    req_perform()

  resp_status(resp) == 200
}

if (works_as_access_token(TOKEN)) {
  cat("▶ Token accepted as ACCESS token (replicator mode)\n")

  token_header <- paste("Bearer", TOKEN)

} else {
  cat("▶ Direct auth failed — attempting REFRESH exchange (pipeline mode)...\n")

  refresh_resp <- request("https://api.dropboxapi.com/oauth2/token") |>
    req_auth_basic(KEY, SECRET) |>
    req_body_form(
      grant_type = "refresh_token",
      refresh_token = TOKEN
    ) |>
    req_error(is_error = function(resp) FALSE) |>
    req_perform()

  refresh_data <- resp_body_json(refresh_resp)

  if (resp_status(refresh_resp) != 200 || is.null(refresh_data$access_token)) {
    stop(paste(
      "TOKEN is neither a valid access token nor a valid refresh token.",
      "Check TOKEN, KEY, and SECRET (token may be expired or revoked)."
    ))
  }

  token_header <- paste("Bearer", refresh_data$access_token)
}

# ============================================================
# LIST DROPBOX ROOT FILES
# ============================================================
cat("\nConnecting to Dropbox...\n")

list_dropbox_entries <- function(path = "", recursive = TRUE) {

  resp <- request("https://api.dropboxapi.com/2/files/list_folder") |>
    req_method("POST") |>
    req_headers(
      Authorization = token_header,
      "Content-Type" = "application/json"
    ) |>
    req_body_json(list(path = path, recursive = recursive)) |>
    req_perform()

  data <- resp_body_json(resp)
  entries <- data$entries

  # Dropbox paginates results: keep fetching until has_more is FALSE
  while (isTRUE(data$has_more)) {
    resp <- request("https://api.dropboxapi.com/2/files/list_folder/continue") |>
      req_method("POST") |>
      req_headers(
        Authorization = token_header,
        "Content-Type" = "application/json"
      ) |>
      req_body_json(list(cursor = data$cursor)) |>
      req_perform()

    data <- resp_body_json(resp)
    entries <- c(entries, data$entries)
  }

  entries
}

entries <- list_dropbox_entries("", recursive = FALSE)

cat("\n--- Root Contents ---\n")

if (length(entries) == 0) {
  cat("✓ Authentication successful, but root is empty\n")
} else {
  for (entry in entries) {
    tag <- entry$`.tag`
    if (identical(tag, "folder")) {
      cat("📁", entry$name, "\n")
    } else if (identical(tag, "file")) {
      cat(sprintf("📄 %s  (uploaded: %s)\n",
                  entry$name, sub("Z", "", sub("T", " ", entry$server_modified))))
    }
  }
}

# ============================================================
# FOLDER NAVIGATION
# ============================================================
# Interactive browser over the Dropbox tree. Type an entry's number to
# open a folder or select a file, ".." to go up one level, "q" to quit.
# Returns the API path of the selected file (or NULL if quit), so the
# result can be passed directly to read_dropbox_file().
navigate_dropbox <- function(start_path = "") {

  current <- start_path

  repeat {

    entries <- list_dropbox_entries(current, recursive = FALSE)

    cat(sprintf("\n📂 %s\n", ifelse(current == "", "/ (root)", current)))

    if (length(entries) == 0) {
      cat("  (empty folder)\n")
    } else {
      for (i in seq_along(entries)) {
        icon <- ifelse(identical(entries[[i]]$`.tag`, "folder"), "📁", "📄")
        stamp <- ""
        if (!is.null(entries[[i]]$server_modified)) {
          stamp <- sprintf("  (uploaded: %s)",
                           sub("Z", "", sub("T", " ", entries[[i]]$server_modified)))
        }
        cat(sprintf("  [%d] %s %s%s\n", i, icon, entries[[i]]$name, stamp))
      }
    }

    choice <- trimws(readline("Number to open, '..' = up, 'q' = quit: "))

    if (tolower(choice) == "q") {
      return(invisible(NULL))
    }

    if (choice == "..") {
      if (current == "") {
        cat("Already at root.\n")
      } else {
        # Drop the last path segment; "" means back at root
        current <- sub("/[^/]+$", "", current)
      }
      next
    }

    idx <- suppressWarnings(as.integer(choice))

    if (is.na(idx) || idx < 1 || idx > length(entries)) {
      cat("Invalid selection, try again.\n")
      next
    }

    entry <- entries[[idx]]

    if (identical(entry$`.tag`, "folder")) {
      current <- entry$path_lower
    } else {
      cat("✔ Selected:", entry$path_display, "\n")
      return(entry$path_lower)
    }
  }
}

# ============================================================
# LOCAL CACHE
# ============================================================
CACHE_DIR <- "cached-input-data"
dir.create(CACHE_DIR, showWarnings = FALSE, recursive = TRUE)

dropbox_metadata <- function(api_path) {
  resp <- request("https://api.dropboxapi.com/2/files/get_metadata") |>
    req_method("POST") |>
    req_headers(
      Authorization = token_header,
      "Content-Type" = "application/json"
    ) |>
    req_body_json(list(path = api_path)) |>
    req_perform()

  resp_body_json(resp)
}

cache_path_for <- function(api_path) {
  file.path(CACHE_DIR, gsub("[^A-Za-z0-9._-]+", "_", sub("^/+", "", api_path)))
}

# ============================================================
# UNIVERSAL FILE READER
# ============================================================
# col_select (tidy-select, e.g. c(ID, YEAR)) reads only those columns
# into memory; it applies to csv and parquet and is ignored (with a
# warning) for other formats. Extra arguments in ... are forwarded to
# the format's reader.
#
# Caching: downloads are stored in CACHE_DIR. The cached copy is reused
# when it is newer than the file's server_modified time on Dropbox, or
# when Dropbox cannot be reached. Set use_cache = FALSE to force a
# fresh download.
read_dropbox_file <- function(api_path, col_select = NULL, ..., use_cache = TRUE) {

  local_file <- cache_path_for(api_path)

  fetch_needed <- TRUE

  if (use_cache && file.exists(local_file)) {

    remote_time <- tryCatch({
      meta <- dropbox_metadata(api_path)
      as.POSIXct(meta$server_modified, format = "%Y-%m-%dT%H:%M:%SZ", tz = "UTC")
    }, error = function(e) NULL)

    if (is.null(remote_time)) {
      warning("Could not check Dropbox version — using cached copy: ", local_file)
      fetch_needed <- FALSE
    } else if (file.mtime(local_file) >= remote_time) {
      fetch_needed <- FALSE
    }
  }

  if (fetch_needed) {
    cat("📥 Downloading:", api_path, "\n")

    resp <- request("https://content.dropboxapi.com/2/files/download") |>
      req_headers(
        Authorization = token_header,
        "Dropbox-API-Arg" = jsonlite::toJSON(list(path = api_path), auto_unbox = TRUE)
      ) |>
      req_perform()

    writeBin(resp_body_raw(resp), local_file)
  } else {
    cat("🗂️ Using cache:", local_file, "\n")
  }

  ext <- tolower(tools::file_ext(api_path))

  if (ext == "csv") {
    read_csv(local_file, col_select = {{ col_select }}, ...)

  } else if (ext %in% c("xlsx", "xls")) {
    if (!missing(col_select)) warning("col_select is ignored for Excel files")
    read_excel(local_file, ...)

  } else if (ext == "json") {
    if (!missing(col_select)) warning("col_select is ignored for JSON files")
    fromJSON(local_file, ...)

  } else if (ext == "parquet") {
    # arrow restores R attributes stored in the file's metadata (haven
    # variable/value labels, labelled classes); DuckDB drops them.
    arrow::read_parquet(local_file, col_select = {{ col_select }}, ...)

  } else {
    warning(paste("Unsupported format:", ext))
    readBin(local_file, "raw", file.size(local_file))
  }
}

# ============================================================
# SELECT & READ
# ============================================================
# The interactively chosen path is remembered in the cache dir. On later
# runs, if that file is already cached, the navigator is skipped and the
# data is read directly (still version-checked against Dropbox). Delete
# the .last_path file or run navigate_dropbox() manually to pick anew.
LAST_PATH_FILE <- file.path(CACHE_DIR, ".last_path")

selected_path <- NULL

if (file.exists(LAST_PATH_FILE)) {
  remembered <- readLines(LAST_PATH_FILE, n = 1)
  if (file.exists(cache_path_for(remembered))) {
    cat("↩️ Reusing cached selection:", remembered, "\n")
    selected_path <- remembered
  }
}

if (is.null(selected_path)) {
  selected_path <- navigate_dropbox()
  if (!is.null(selected_path)) writeLines(selected_path, LAST_PATH_FILE)
}

if (!is.null(selected_path)) df <- read_dropbox_file(selected_path, col_select = c(ID, YEAR, TAXABLE_INCOME_ND_RC, LABOR_TOT_INCOME_ND_RP))


# ============================================================
# CACHE STATUS
# ============================================================
cat(sprintf("\n--- Local Cache (%s) ---\n", normalizePath(CACHE_DIR)))

cached_files <- list.files(CACHE_DIR, full.names = TRUE)

if (length(cached_files) == 0) {
  cat("(cache is empty)\n")
} else {
  info <- file.info(cached_files)
  for (i in seq_along(cached_files)) {
    cat(sprintf("🗂️ %-40s %8.1f MB   cached: %s\n",
                basename(cached_files[i]),
                info$size[i] / 1e6,
                format(info$mtime[i], "%Y-%m-%d %H:%M")))
  }
}

# ============================================================
# SESSION CLEANUP
# ============================================================
# Drop everything this script created except the data itself — most
# importantly the credentials (KEY, SECRET, TOKEN, token_header), which
# should not linger in the workspace or get saved into .RData.
rm(list = setdiff(ls(), "df"))
invisible(gc())

cat("\n🧹 Session cleaned .\n")



```
