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

```python
import os
import dropbox
from dropbox.oauth import DropboxOAuth2FlowNoRedirect
from dropbox.exceptions import AuthError, ApiError

# =====================================================================
# CONFIGURATION
# =====================================================================
APP_KEY = "your app key"
APP_SECRET = "your app secret"

# OPTIONS: 
# "refresh"     -> For permanent backend pipelines / internal RAs
# "short_lived" -> For external journal replicators (Expires in 4 hours)
TOKEN_MODE = "short_lived" # Change to "refresh" for permanent token generation

# =====================================================================
# STEP 1: TOKEN GENERATION FLOW
# =====================================================================
access_type = "offline" if TOKEN_MODE == "refresh" else "online"
print(f"▶ Initializing token manager in [{TOKEN_MODE.upper()}] mode...")

auth_flow = DropboxOAuth2FlowNoRedirect(APP_KEY, APP_SECRET, token_access_type=access_type)
authorize_url = auth_flow.start()

print(f"\n1. Open this URL in your browser to authorize access:\n{authorize_url}")
auth_code = input("\n2. Enter the authorization code here: ").strip()

# The critical fix: This step exchanges the temporary authorization code 
# for actual credentials behind the scenes.
oauth_result = auth_flow.finish(auth_code)

print("\n=================== CREDENTIAL EXPORT ===================")
if TOKEN_MODE == "refresh":
    print(f'DROPBOX_REFRESH_TOKEN = "{oauth_result.refresh_token}"')
    print("\n✓ Save this permanent Refresh Token for automated data pipelines.")
else:
    print(f'DROPBOX_ACCESS_TOKEN = "{oauth_result.access_token}"')
    print("\n⚠️ Save this token. It will hard-expire in exactly 4 hours.")
print("=========================================================\n")

# =====================================================================
# STEP 2: VERIFY CONNECTION TO PARQUET DATA LAYER
# =====================================================================
print("Testing generated credentials against the data layer...")

try:
    if TOKEN_MODE == "short_lived":
        # Pass the resolved, functional short-lived token directly
        dbx = dropbox.Dropbox(oauth2_access_token=oauth_result.access_token)
    else:
        # Pass the permanent background refresh token
        dbx = dropbox.Dropbox(
            oauth2_refresh_token=oauth_result.refresh_token,
            app_key=APP_KEY,
            app_secret=APP_SECRET
        )
        
    # Execute metadata scan on the root app folder
    result = dbx.files_list_folder("")
    print("✅ Connection Successful! Verified data layer contents:")
    
    file_count = 0
    for entry in result.entries:
        if isinstance(entry, dropbox.files.FileMetadata):
            file_count += 1
            print(f"  📄 {entry.name} ({entry.size:,} bytes)")
            
    if file_count == 0:
        print("  (Folder authenticated successfully, but contains no files.)")

except AuthError as e:
    print(f"\n❌ Authentication Denied: {e}")
except ApiError as e:
    print(f"\n❌ Dropbox Storage Architecture Error: {e}")

```

- See R version
    
    ```r
# =====================================================================
# PACKAGES
# =====================================================================
library(httr)
library(jsonlite)

# =====================================================================
# CONFIGURATION
# =====================================================================
APP_KEY <- "fi2oqjuko41oi1r"
APP_SECRET <- "ozzb1af2uudrmtl"

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
import os
import dropbox
from dropbox.exceptions import AuthError, ApiError

# =====================================================================
# CONFIGURATION
# =====================================================================
# App credentials (Only required if you are using a permanent refresh token)
KEY = "your key"
SECRET = "your secret"

# Paste either token type here:
# - Paste an "sl.u..." token (Short-lived, 4-hour export for replicators)
# - Paste a "ds.u..." token (Permanent refresh token for pipelines/collaborators)
TOKEN = "your token"

# =====================================================================
# AUTOMATED CLIENT CONFIGURATION
# =====================================================================
print("Analyzing token signatures...")

if TOKEN.startswith("sl."):
    print("▶ Target: SHORT-LIVED Access Token detected (External Replicator Mode).")
    print("  Strategy: Disabling refresh loop. Initializing direct API context...")
    
    # Initialize using direct token validation without an token-exchange loop
    dbx = dropbox.Dropbox(oauth2_access_token=TOKEN)

elif TOKEN.startswith("ds.") or len(TOKEN) < 100:
    # Note: Depending on your app type, some older refresh tokens are short, 
    # but modern scoped refresh tokens typically lean on the "ds." prefix.
    print("▶ Target: PERMANENT Refresh Token detected (Pipeline / Collaborator Mode).")
    print("  Strategy: Activating background OAuth2 engine for automated rotations...")
    
    # Initialize using the token-exchange loop (requires App Key & Secret)
    dbx = dropbox.Dropbox(
        oauth2_refresh_token=TOKEN,
        app_key=KEY,
        app_secret=SECRET
    )

else:
    raise ValueError(
        "CRITICAL: Unrecognized token format. Your token must start with "
        "'sl.' (short-lived) or 'ds.' (permanent refresh token)."
    )

# =====================================================================
# FILE TREE
# =====================================================================
def list_dropbox_tree(folder_path="", prefix=""):
    """
    Recursively prints folders and files in a tree layout starting from folder_path.
    Returns a flat list of all file paths found.
    """
    all_files = []
    try:
        result = dbx.files_list_folder(folder_path)
        entries = list(result.entries)

        # Paginate if there are more entries
        while result.has_more:
            result = dbx.files_list_folder_continue(result.cursor)
            entries.extend(result.entries)

        # Folders first, then files
        folders = [e for e in entries if isinstance(e, dropbox.files.FolderMetadata)]
        files   = [e for e in entries if isinstance(e, dropbox.files.FileMetadata)]
        sorted_entries = sorted(folders, key=lambda e: e.name.lower()) + \
                         sorted(files,   key=lambda e: e.name.lower())

        for i, entry in enumerate(sorted_entries):
            is_last   = (i == len(sorted_entries) - 1)
            connector = "└── " if is_last else "├── "
            child_pfx = prefix + ("    " if is_last else "│   ")

            if isinstance(entry, dropbox.files.FolderMetadata):
                print(f"{prefix}{connector}📁 {entry.name}/")
                sub_files = list_dropbox_tree(entry.path_lower, child_pfx)
                all_files.extend(sub_files)
            else:
                size_kb = entry.size / 1024
                print(f"{prefix}{connector}📄 {entry.name}  ({size_kb:,.1f} KB)")
                all_files.append(entry.path_lower)

    except AuthError as e:
        print("\n❌ AUTHENTICATION FAILED!")
        if TOKEN.startswith("sl."):
            print("  - Reason: Your 4-hour replication window has elapsed.")
            print("  - Remedy: Please contact the corresponding author for a fresh session token.")
        else:
            print("  - Reason: App credentials (KEY/SECRET) mismatch or revoked refresh token.")
        print(f"  - Technical Log: {e}")
    except ApiError as e:
        print(f"\n❌ STORAGE ARCHITECTURE ERROR: {e}")

    return all_files

# =====================================================================
# DATA LAYER EXECUTION
# =====================================================================
print("Connecting to secure data layer and scanning storage layout...")
print("\n--- Dropbox File Tree ---")
print("📦 (root)")
all_files = list_dropbox_tree()
print(f"\n✓ Total files found: {len(all_files)}")

```

- See R version
    
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
    library(rawr)   # optional fallback for bytes handling
    
    # ============================================================
    # CONFIGURATION
    # ============================================================
    KEY    <- "your key"
    SECRET <- "your secret"
    
    TOKEN <- "your token"
    
    # ============================================================
    # AUTH HANDLING
    # ============================================================
    cat("Analyzing token signature...\n")
    
    if (startsWith(TOKEN, "sl.")) {
      cat("▶ SHORT-LIVED token detected (replicator mode)\n")
      
      token_header <- paste("Bearer", TOKEN)
    
    } else if (startsWith(TOKEN, "ds.")) {
      cat("▶ REFRESH token detected (pipeline mode)\n")
      
      # In R, refresh flow is handled via OAuth app flow (simplified here)
      token_header <- paste("Bearer", TOKEN)
    
    } else {
      stop("Unrecognized token format. Must start with 'sl.' or 'ds.'")
    }
    
    # ============================================================
    # LIST DROPBOX ROOT FILES
    # ============================================================
    cat("\nConnecting to Dropbox...\n")
    
    req <- request("https://api.dropboxapi.com/2/files/list_folder") |>
      req_method("POST") |>
      req_headers(
        Authorization = token_header,
        "Content-Type" = "application/json"
      ) |>
      req_body_json(list(path = ""))
    
    resp <- req_perform(req)
    data <- resp_body_json(resp)
    
    cat("\n--- Active Files ---\n")
    
    if (length(data$entries) == 0) {
      cat("✓ Authentication successful, but folder is empty\n")
    } else {
      for (entry in data$entries) {
        if (entry[".tag"] == "file") {
          cat("📄", entry$name, "\n")
        }
      }
    }
    
    # ============================================================
    # UNIVERSAL FILE READER
    # ============================================================
    read_dropbox_file <- function(api_path) {
      
      cat("📥 Streaming:", api_path, "\n")
      
      # Download file
      req <- request("https://content.dropboxapi.com/2/files/download") |>
        req_headers(
          Authorization = token_header,
          "Dropbox-API-Arg" = jsonlite::toJSON(list(path = api_path), auto_unbox = TRUE)
        )
      
      resp <- req_perform(req)
      file_bytes <- resp_body_raw(resp)
      
      ext <- tools::file_ext(api_path)
      
      if (ext == "csv") {
        return(read_csv(rawConnection(file_bytes)))
        
      } else if (ext %in% c("xlsx", "xls")) {
        tmp <- tempfile(fileext = paste0(".", ext))
        writeBin(file_bytes, tmp)
        return(read_excel(tmp))
        
      } else if (ext == "json") {
        return(fromJSON(rawToChar(file_bytes)))
        
      } else if (ext == "parquet") {
        tmp <- tempfile(fileext = ".parquet")
        writeBin(file_bytes, tmp)
        
        con <- dbConnect(duckdb::duckdb())
        df <- dbGetQuery(con, paste0("SELECT * FROM read_parquet('", tmp, "')"))
        dbDisconnect(con, shutdown = TRUE)
        
        return(df)
        
      } else {
        warning(paste("Unsupported format:", ext))
        return(file_bytes)
      }
    }
    
    # ============================================================
    # EXAMPLE READ
    # ============================================================
    df <- read_dropbox_file("/file.csv")
    ```
