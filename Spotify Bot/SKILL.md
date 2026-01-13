# Spotify Library Agent Skill

## Overview

This skill enables an AI agent to interact with the Spotify Web API to analyze a user's music library, generate insights and visualizations, and manage playlists intelligently. The agent can categorize music, create smart playlists, surface hidden recommendations, and provide detailed listening analytics.

## Session Initialization

At the start of every session, ask the user:

1. **Cache preference**: "Would you like to start with a fresh cache or load an existing one?"
   - If existing: Look for `spotify_cache.json` in the working directory
   - If fresh: Create new cache file, warn user that initial analysis will require more API calls

2. **Verify credentials**: Confirm the user has valid Spotify API credentials before proceeding

## Authentication Setup

### User Credential Requirements

The user must provide the following credentials:

```
SPOTIFY_CLIENT_ID=<client_id>
SPOTIFY_CLIENT_SECRET=<client_secret>
SPOTIFY_REDIRECT_URI=http://localhost:8888/callback
SPOTIFY_ACCESS_TOKEN=<access_token>      # After OAuth flow
SPOTIFY_REFRESH_TOKEN=<refresh_token>    # After OAuth flow
```

### Obtaining Credentials - Guide for Users

Provide these instructions when a user needs to set up credentials:

#### Step 1: Create a Spotify Developer Application

1. Go to https://developer.spotify.com/dashboard
2. Log in with your Spotify account
3. Click "Create App"
4. Fill in the details:
   - App name: Any name (e.g., "My Library Agent")
   - App description: Any description
   - Redirect URI: `http://localhost:8888/callback`
5. Check the Web API checkbox
6. Accept the terms and click "Save"
7. Click "Settings" to find your Client ID and Client Secret

#### Step 2: OAuth Authorization Flow

The agent should help the user complete OAuth by:

1. **Generate Authorization URL**:
```
https://accounts.spotify.com/authorize?client_id={CLIENT_ID}&response_type=code&redirect_uri=http://localhost:8888/callback&scope={SCOPES}&state={RANDOM_STATE}
```

2. **Required Scopes** (space-separated in URL, use %20):
```
user-library-read
user-library-modify
user-top-read
user-read-recently-played
playlist-read-private
playlist-read-collaborative
playlist-modify-public
playlist-modify-private
user-read-private
user-read-email
```

3. **User visits URL**, authorizes, gets redirected to callback with `?code=XXX`

4. **Exchange code for tokens**:
```bash
curl -X POST https://accounts.spotify.com/api/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Basic $(echo -n 'CLIENT_ID:CLIENT_SECRET' | base64)" \
  -d "grant_type=authorization_code" \
  -d "code=AUTHORIZATION_CODE" \
  -d "redirect_uri=http://localhost:8888/callback"
```

Response contains `access_token`, `refresh_token`, and `expires_in` (3600 seconds).

#### Step 3: Token Refresh

Access tokens expire after 1 hour. Refresh with:

```bash
curl -X POST https://accounts.spotify.com/api/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Basic $(echo -n 'CLIENT_ID:CLIENT_SECRET' | base64)" \
  -d "grant_type=refresh_token" \
  -d "refresh_token=REFRESH_TOKEN"
```

**Always check token validity before making requests. If a request returns 401, refresh the token and retry.**

---

## API Reference

### Base URL
```
https://api.spotify.com/v1
```

### Authentication Header
```
Authorization: Bearer {access_token}
```

### Core Endpoints

#### User Library

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/me/tracks` | GET | Get user's saved tracks | Standard |
| `/me/tracks` | PUT | Save tracks to library | Standard |
| `/me/tracks` | DELETE | Remove tracks from library | Standard |
| `/me/tracks/contains` | GET | Check if tracks are saved | Standard |
| `/me/albums` | GET | Get user's saved albums | Standard |
| `/me/following` | GET | Get followed artists | Standard |

**Pagination for `/me/tracks`**:
- `limit`: 1-50 (default 20)
- `offset`: 0-based position
- Maximum offset: 10,000 items (API limitation)

**Response structure for `/me/tracks`**:
```json
{
  "href": "https://api.spotify.com/v1/me/tracks",
  "items": [
    {
      "added_at": "2024-01-15T10:30:00Z",
      "track": {
        "id": "track_id",
        "name": "Track Name",
        "artists": [{"id": "artist_id", "name": "Artist Name"}],
        "album": {"id": "album_id", "name": "Album Name", "release_date": "2023-05-01"},
        "duration_ms": 240000,
        "popularity": 75,
        "explicit": false
      }
    }
  ],
  "limit": 50,
  "next": "https://api.spotify.com/v1/me/tracks?offset=50&limit=50",
  "offset": 0,
  "previous": null,
  "total": 1500
}
```

#### User Top Items

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/me/top/artists` | GET | Get user's top artists | Standard |
| `/me/top/tracks` | GET | Get user's top tracks | Standard |

**Parameters**:
- `time_range`: `short_term` (4 weeks), `medium_term` (6 months), `long_term` (years)
- `limit`: 1-50
- `offset`: 0-based

#### Recently Played

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/me/player/recently-played` | GET | Get recently played tracks | Standard |

**Parameters**:
- `limit`: 1-50
- `before`: Unix timestamp (cursor pagination)
- `after`: Unix timestamp (cursor pagination)

**Note**: Only returns last 50 unique tracks played.

#### Playlists

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/me/playlists` | GET | Get current user's playlists | Standard |
| `/users/{user_id}/playlists` | POST | Create playlist | Standard |
| `/playlists/{playlist_id}` | GET | Get playlist details | Standard |
| `/playlists/{playlist_id}` | PUT | Update playlist details | Standard |
| `/playlists/{playlist_id}/tracks` | GET | Get playlist tracks | Standard |
| `/playlists/{playlist_id}/tracks` | POST | Add tracks to playlist | Standard |
| `/playlists/{playlist_id}/tracks` | PUT | Replace playlist tracks | Standard |
| `/playlists/{playlist_id}/tracks` | DELETE | Remove tracks from playlist | Standard |

**Creating a playlist**:
```json
POST /users/{user_id}/playlists
{
  "name": "Playlist Name",
  "description": "Playlist description",
  "public": false
}
```

**Adding tracks** (max 100 per request):
```json
POST /playlists/{playlist_id}/tracks
{
  "uris": ["spotify:track:id1", "spotify:track:id2"],
  "position": 0  // Optional, adds to end if omitted
}
```

**Playlist response includes**:
```json
{
  "id": "playlist_id",
  "name": "Playlist Name",
  "description": "Description",
  "owner": {"id": "user_id", "display_name": "User"},
  "public": false,
  "collaborative": false,
  "tracks": {"total": 50},
  "snapshot_id": "snapshot_id",
  "images": [{"url": "image_url"}]
}
```

**Note**: `snapshot_id` changes when playlist is modified. Use it for conditional updates.

#### Track Audio Features

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/audio-features/{id}` | GET | Get audio features for track | Standard |
| `/audio-features` | GET | Get audio features for multiple tracks | Standard |

**Batch endpoint** (`/audio-features?ids=id1,id2,...`):
- Maximum 100 track IDs per request

**Audio features response**:
```json
{
  "acousticness": 0.00242,      // 0.0 - 1.0
  "danceability": 0.585,        // 0.0 - 1.0
  "energy": 0.842,              // 0.0 - 1.0
  "instrumentalness": 0.00686,  // 0.0 - 1.0
  "key": 9,                     // 0-11 (pitch class)
  "liveness": 0.0866,           // 0.0 - 1.0
  "loudness": -5.883,           // dB, typically -60 to 0
  "mode": 0,                    // 0 = minor, 1 = major
  "speechiness": 0.0556,        // 0.0 - 1.0
  "tempo": 118.211,             // BPM
  "time_signature": 4,          // beats per bar
  "valence": 0.428              // 0.0 - 1.0 (musical positiveness)
}
```

#### Artists

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/artists/{id}` | GET | Get artist details | Standard |
| `/artists` | GET | Get multiple artists | Standard |
| `/artists/{id}/related-artists` | GET | Get related artists | Standard |

**Batch endpoint** (`/artists?ids=id1,id2,...`):
- Maximum 50 artist IDs per request

**Artist response includes genres**:
```json
{
  "id": "artist_id",
  "name": "Artist Name",
  "genres": ["jazz", "soul", "r&b"],
  "popularity": 82,
  "followers": {"total": 5000000}
}
```

#### Recommendations

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/recommendations` | GET | Get recommendations | Standard |

**Parameters**:
- `seed_artists`: Comma-separated artist IDs (max 5 total seeds)
- `seed_tracks`: Comma-separated track IDs (max 5 total seeds)
- `seed_genres`: Comma-separated genres (max 5 total seeds)
- `limit`: 1-100
- Target/min/max parameters for audio features (e.g., `target_energy`, `min_tempo`)

**Available seed genres**: Use `/recommendations/available-genre-seeds` to get the list.

#### Search

| Endpoint | Method | Description | Rate Category |
|----------|--------|-------------|---------------|
| `/search` | GET | Search for items | Standard |

**Parameters**:
- `q`: Search query
- `type`: Comma-separated: `album`, `artist`, `playlist`, `track`
- `limit`: 1-50
- `offset`: 0-based

---

## Rate Limiting

### Spotify Rate Limits

Spotify uses a sliding window rate limit. The exact limits are not publicly documented but are approximately:

- **Standard requests**: ~180 requests per minute per user
- **Batch requests count as 1 request** regardless of items fetched

### Rate Limit Response

When rate limited, Spotify returns:
```
HTTP 429 Too Many Requests
Retry-After: {seconds}
```

### Agent Rate Limit Strategy

Use a **hybrid approach** based on request type:

#### Fast Mode (Default for user requests)
- Direct single requests without artificial delays
- Handle 429s reactively with exponential backoff
- Use for: Single track lookups, playlist info, quick searches

```python
# Pseudocode for fast mode
def fast_request(endpoint):
    response = make_request(endpoint)
    if response.status == 429:
        wait_time = response.headers['Retry-After']
        sleep(wait_time)
        return fast_request(endpoint)
    return response
```

#### Batched Mode (Default for analytics)
- Group requests to use batch endpoints
- Add 100-200ms delay between batch requests
- Use for: Library analysis, audio feature collection, genre mapping

```python
# Pseudocode for batched mode
def batch_audio_features(track_ids):
    results = []
    for chunk in chunks(track_ids, 100):  # Max 100 per request
        response = make_request(f"/audio-features?ids={','.join(chunk)}")
        results.extend(response['audio_features'])
        sleep(0.15)  # 150ms delay
    return results
```

#### Adaptive Mode (For complex operations)
- Start fast, switch to batched if approaching limits
- Track request count in sliding window
- Use for: Multi-step operations, playlist creation with analysis

```python
# Pseudocode for adaptive mode
class AdaptiveRateLimiter:
    def __init__(self):
        self.requests = []  # Timestamps
        self.window = 60    # 1 minute
        self.threshold = 150  # Switch to batched at 150 req/min
    
    def should_batch(self):
        now = time.now()
        self.requests = [t for t in self.requests if now - t < self.window]
        return len(self.requests) > self.threshold
    
    def record_request(self):
        self.requests.append(time.now())
```

### Batch Endpoint Maximums

| Endpoint | Max Items per Request |
|----------|----------------------|
| `/audio-features` | 100 track IDs |
| `/artists` | 50 artist IDs |
| `/tracks` | 50 track IDs |
| `/me/tracks/contains` | 50 track IDs |
| `/playlists/{id}/tracks` (POST) | 100 URIs |
| `/playlists/{id}/tracks` (DELETE) | 100 URIs |

---

## Caching Strategy

### Cache Structure

Store cache as `spotify_cache.json`:

```json
{
  "metadata": {
    "created_at": "2024-01-15T10:00:00Z",
    "last_updated": "2024-01-15T12:30:00Z",
    "user_id": "user_spotify_id",
    "library_snapshot": {
      "total_tracks": 1500,
      "last_synced": "2024-01-15T12:00:00Z"
    }
  },
  "tracks": {
    "track_id_1": {
      "name": "Track Name",
      "artists": ["Artist 1", "Artist 2"],
      "artist_ids": ["artist_id_1", "artist_id_2"],
      "album": "Album Name",
      "album_id": "album_id",
      "added_at": "2024-01-10T08:00:00Z",
      "duration_ms": 240000,
      "popularity": 75,
      "cached_at": "2024-01-15T10:00:00Z"
    }
  },
  "audio_features": {
    "track_id_1": {
      "danceability": 0.585,
      "energy": 0.842,
      "valence": 0.428,
      "tempo": 118.211,
      "acousticness": 0.00242,
      "instrumentalness": 0.00686,
      "speechiness": 0.0556,
      "liveness": 0.0866,
      "key": 9,
      "mode": 0,
      "cached_at": "2024-01-15T10:05:00Z"
    }
  },
  "artists": {
    "artist_id_1": {
      "name": "Artist Name",
      "genres": ["jazz", "soul"],
      "popularity": 82,
      "cached_at": "2024-01-15T10:10:00Z"
    }
  },
  "genre_mappings": {
    "track_id_1": {
      "genres": ["jazz", "soul"],
      "primary_genre": "jazz",
      "derived_from": "artist_id_1",
      "cached_at": "2024-01-15T10:15:00Z"
    }
  },
  "playlists": {
    "playlist_id_1": {
      "name": "My Jazz Playlist",
      "description": "Auto-generated jazz collection",
      "track_count": 45,
      "track_ids": ["track_id_1", "track_id_2"],
      "created_by_agent": true,
      "snapshot_id": "snapshot_abc123",
      "cached_at": "2024-01-15T11:00:00Z"
    }
  }
}
```

### Cache Invalidation Rules

| Data Type | Cache Duration | Invalidation Trigger |
|-----------|---------------|---------------------|
| Track metadata | 30 days | Manual refresh |
| Audio features | Indefinite | Never (immutable) |
| Artist genres | 7 days | Manual refresh |
| Genre mappings | 7 days | Artist cache invalidation |
| Playlist contents | 1 hour | Snapshot ID change |
| User library | On demand | User requests sync |

### Cache Operations

**On session start with existing cache**:
1. Load cache file
2. Check `metadata.last_updated`
3. Ask user if they want to sync library (fetch new tracks since `library_snapshot.last_synced`)

**During operation**:
1. Always check cache before API call
2. For track lookups: Use cache if present and < 30 days old
3. For audio features: Always use cache (immutable data)
4. For artist genres: Use cache if < 7 days old
5. Update cache after each API response

**Cache maintenance**:
- Provide "rebuild cache" option for users
- Warn when cache is > 7 days old at session start
- Save cache after significant operations (every 50 API calls or on session end)

---

## Genre Classification

### Genre Hierarchy

Spotify has hundreds of micro-genres. Map them to broad categories for playlist organization:

```json
{
  "Rock": ["rock", "hard rock", "soft rock", "classic rock", "alternative rock", "indie rock", "punk rock", "post-punk", "garage rock", "psychedelic rock", "progressive rock", "grunge", "emo", "pop rock", "art rock", "blues rock", "southern rock", "roots rock"],
  
  "Pop": ["pop", "dance pop", "electropop", "synth-pop", "indie pop", "art pop", "chamber pop", "power pop", "teen pop", "k-pop", "j-pop", "europop", "bubblegum pop"],
  
  "Hip-Hop/Rap": ["hip hop", "rap", "trap", "southern hip hop", "east coast hip hop", "west coast hip hop", "gangsta rap", "conscious hip hop", "underground hip hop", "dirty south", "crunk", "grime", "drill"],
  
  "R&B/Soul": ["r&b", "soul", "neo soul", "contemporary r&b", "funk", "motown", "quiet storm", "new jack swing", "disco"],
  
  "Electronic/Dance": ["electronic", "edm", "house", "deep house", "tech house", "progressive house", "techno", "trance", "dubstep", "drum and bass", "ambient", "downtempo", "chillwave", "synthwave", "electronica", "idm", "trip hop"],
  
  "Jazz": ["jazz", "smooth jazz", "jazz fusion", "bebop", "cool jazz", "free jazz", "latin jazz", "acid jazz", "jazz funk", "big band", "swing"],
  
  "Classical": ["classical", "baroque", "romantic", "contemporary classical", "opera", "orchestral", "chamber music", "piano", "symphony", "minimalism", "neoclassical"],
  
  "Country": ["country", "contemporary country", "country rock", "outlaw country", "country pop", "americana", "bluegrass", "folk country", "honky tonk", "nashville sound"],
  
  "Folk/Acoustic": ["folk", "indie folk", "contemporary folk", "folk rock", "acoustic", "singer-songwriter", "traditional folk", "freak folk"],
  
  "Metal": ["metal", "heavy metal", "thrash metal", "death metal", "black metal", "doom metal", "power metal", "progressive metal", "nu metal", "metalcore", "deathcore", "symphonic metal", "folk metal"],
  
  "Blues": ["blues", "electric blues", "delta blues", "chicago blues", "blues rock", "soul blues", "contemporary blues"],
  
  "Reggae/Caribbean": ["reggae", "dancehall", "dub", "ska", "roots reggae", "reggaeton", "soca", "calypso"],
  
  "Latin": ["latin", "salsa", "bachata", "merengue", "cumbia", "bossa nova", "latin pop", "tropical", "latin rock", "norteño", "banda", "mariachi"],
  
  "World": ["world", "afrobeat", "afropop", "highlife", "african", "middle eastern", "indian", "asian", "celtic", "flamenco", "fado"],
  
  "Punk": ["punk", "punk rock", "pop punk", "hardcore punk", "post-hardcore", "skate punk", "anarcho-punk"],
  
  "Indie/Alternative": ["indie", "alternative", "indie rock", "indie pop", "alternative rock", "lo-fi", "shoegaze", "dream pop", "noise rock", "post-rock", "math rock", "slowcore", "sadcore"]
}
```

### Genre Detection Strategy

1. **Get track's artist(s)**
2. **Fetch artist genres** from `/artists/{id}` (use cache/batch)
3. **Map micro-genres to broad categories** using hierarchy above
4. **Handle multiple genres**: Assign to primary category (first match in hierarchy order) but store all matches
5. **Handle unknown genres**: Flag for manual review or assign to "Other"

**Algorithm**:
```python
def classify_track(track, artist_genres):
    matched_categories = []
    for category, genre_list in GENRE_HIERARCHY.items():
        for artist_genre in artist_genres:
            # Fuzzy match: check if any genre_list item is contained in artist_genre
            for known_genre in genre_list:
                if known_genre in artist_genre.lower():
                    matched_categories.append(category)
                    break
    
    if matched_categories:
        return {
            "primary": matched_categories[0],
            "all": list(set(matched_categories))
        }
    return {"primary": "Other", "all": ["Other"]}
```

---

## Core Agent Operations

### 1. Library Analysis

**Trigger**: User asks about their music library, listening habits, or wants an overview.

**Process**:
1. Sync library (if not recently synced):
   - Fetch all saved tracks with pagination (limit=50, batched mode)
   - Store in cache
2. Fetch audio features for uncached tracks (batch endpoint, 100 at a time)
3. Fetch artist info for uncached artists (batch endpoint, 50 at a time)
4. Build genre mappings
5. Generate analysis

**Output metrics**:
- Total tracks, total duration
- Genre distribution
- Top artists by track count
- Audio feature averages and ranges
- Tracks by decade/year
- Recently added vs older tracks ratio

### 2. Smart Playlist Creation

**Trigger**: User asks to organize library, create playlists by genre/mood/criteria, or agent detects 5+ songs matching a category.

**Process**:
1. Identify tracks matching criteria (genre, audio features, date range, etc.)
2. Check if matching playlist exists:
   - Search user playlists by name pattern (e.g., "Auto: Jazz")
   - Use cache to check agent-created playlists
3. **If 5+ tracks match and no existing playlist**:
   - Prompt user for confirmation with preview (track names, count)
   - Create playlist if confirmed
4. **If existing playlist found**:
   - Provide playlist details (see Confirmation Protocol below)
   - Ask to add new tracks, replace, or skip
5. Add tracks (batch mode, 100 per request)
6. Update cache with playlist info

**Naming Convention**:
- Genre playlists: "Auto: {Genre}" (e.g., "Auto: Jazz")
- Mood playlists: "Mood: {Mood}" (e.g., "Mood: Chill")
- Custom criteria: "Custom: {Description}" or user-specified name

### 3. Recommendations

**Trigger**: User asks for music recommendations, new music discovery, or "what should I listen to?"

**Process**:
1. Analyze user's top tracks/artists (use `/me/top/tracks` and `/me/top/artists`)
2. Select seeds based on context:
   - General recommendations: Mix of top artists and tracks
   - Genre-specific: Filter seeds to matching genre
   - Mood-based: Use audio feature targets
3. Call `/recommendations` with appropriate seeds and targets
4. Filter out tracks already in user's library (use `/me/tracks/contains`)
5. Present recommendations with option to:
   - Save to library
   - Add to playlist
   - Get more like specific track

### 4. Criteria-Based Track Addition

**Trigger**: User says "add jazz songs from the past month to playlist X" or similar.

**Process**:
1. Parse criteria:
   - Genre/category filter
   - Date range (added_at)
   - Audio feature thresholds
   - Artist filters
2. Query library (use cache for performance)
3. Apply filters
4. Find target playlist:
   - Search by name
   - Present confirmation with details
5. Check for duplicates (tracks already in playlist)
6. Add non-duplicate tracks (with confirmation)

---

## Confirmation Protocol

### When to Confirm

**Always confirm**:
- Creating new playlists
- Adding tracks to existing playlists
- Removing tracks from playlists
- Modifying playlist details (name, description)

**Batch mode exception**:
- User explicitly enables batch mode
- Confirm once for operation type, then proceed
- Provide summary at end

### Playlist Identification Details

When asking for confirmation on an existing playlist, provide:

```
Found playlist matching your request:

📁 **{Playlist Name}**
   - ID: {playlist_id}
   - Created: {approximate date or "Unknown" if not tracked}
   - Last modified: {snapshot change date from cache, or "Unknown"}
   - Tracks: {total count}
   - Sample tracks:
     • "{Track 1}" by {Artist 1}
     • "{Track 2}" by {Artist 2}
     • "{Track 3}" by {Artist 3}
   - {Public/Private}
   - {Created by this agent / Created manually}

Is this the correct playlist? (yes/no/show more tracks)
```

### Batch Mode

**Activation**: User says "batch mode", "do all of them", "yes to all", or similar.

**Behavior**:
1. Confirm batch mode activation
2. Describe all pending operations
3. Ask for single confirmation
4. Execute all operations
5. Provide detailed summary:
   - Operations completed
   - Any failures
   - Changes made

**Example summary**:
```
Batch operation complete:

✅ Created 3 new playlists:
   - "Auto: Jazz" (45 tracks)
   - "Auto: Electronic" (78 tracks)  
   - "Auto: Folk" (23 tracks)

✅ Updated 1 existing playlist:
   - "Auto: Rock" (+12 tracks, now 156 total)

⚠️ 1 issue:
   - "Auto: Classical" - 4 tracks only (below 5-track threshold, skipped)

Total: 158 tracks organized into 4 playlists
```

---

## Chart Generation

### Available Chart Types

#### 1. Genre Distribution (Pie/Donut Chart)

**Data**: Count of tracks per broad genre category
**Visualization**: Pie chart or donut chart
**Include**: Percentage labels, legend

```python
# Data structure
genre_data = {
    "labels": ["Rock", "Pop", "Hip-Hop", "Jazz", "Electronic", "Other"],
    "values": [245, 189, 156, 89, 201, 120],
    "colors": ["#e74c3c", "#3498db", "#9b59b6", "#f1c40f", "#1abc9c", "#95a5a6"]
}
```

#### 2. Audio Features Radar Chart

**Data**: Average audio features across library or selection
**Metrics**: Danceability, Energy, Valence, Acousticness, Instrumentalness, Speechiness, Liveness
**Scale**: 0-1 (normalized)

```python
# Data structure
radar_data = {
    "features": ["Danceability", "Energy", "Valence", "Acousticness", "Instrumentalness", "Speechiness", "Liveness"],
    "values": [0.65, 0.72, 0.58, 0.23, 0.15, 0.08, 0.18]
}
```

#### 3. Tempo Distribution (Histogram)

**Data**: BPM distribution across library
**Bins**: 60-80, 80-100, 100-120, 120-140, 140-160, 160+
**Show**: Count per bin, average BPM line

#### 4. Top Artists Bar Chart

**Data**: Track count per artist
**Show**: Top 10-20 artists
**Orientation**: Horizontal bars

#### 5. Listening Timeline (Line/Area Chart)

**Data**: Tracks added over time
**X-axis**: Months/years
**Y-axis**: Cumulative or per-period count
**Options**: By genre overlay

#### 6. Energy vs Valence Scatter Plot

**Data**: Each track as a point
**X-axis**: Energy (0-1)
**Y-axis**: Valence (0-1)
**Color**: By genre
**Quadrants**: 
- High energy + High valence = "Happy/Upbeat"
- High energy + Low valence = "Angry/Turbulent"
- Low energy + High valence = "Peaceful/Relaxed"
- Low energy + Low valence = "Sad/Depressed"

#### 7. Decade Distribution (Bar Chart)

**Data**: Tracks by release decade
**Show**: 1960s through 2020s

### Chart Implementation

Use Python with matplotlib, seaborn, or plotly for generation. Save as PNG or interactive HTML.

**Example code structure**:
```python
import matplotlib.pyplot as plt
import numpy as np

def create_genre_pie_chart(genre_counts, output_path):
    """Generate genre distribution pie chart."""
    fig, ax = plt.subplots(figsize=(10, 8))
    
    labels = list(genre_counts.keys())
    sizes = list(genre_counts.values())
    colors = plt.cm.Set3(np.linspace(0, 1, len(labels)))
    
    wedges, texts, autotexts = ax.pie(
        sizes, 
        labels=labels, 
        autopct='%1.1f%%',
        colors=colors,
        pctdistance=0.8
    )
    
    ax.set_title('Genre Distribution in Your Library', fontsize=16, fontweight='bold')
    
    plt.tight_layout()
    plt.savefig(output_path, dpi=150, bbox_inches='tight')
    plt.close()
    
    return output_path

def create_audio_features_radar(features_avg, output_path):
    """Generate audio features radar chart."""
    categories = list(features_avg.keys())
    values = list(features_avg.values())
    
    # Close the radar chart
    values += values[:1]
    angles = np.linspace(0, 2 * np.pi, len(categories), endpoint=False).tolist()
    angles += angles[:1]
    
    fig, ax = plt.subplots(figsize=(8, 8), subplot_kw=dict(polar=True))
    ax.plot(angles, values, 'o-', linewidth=2, color='#3498db')
    ax.fill(angles, values, alpha=0.25, color='#3498db')
    ax.set_xticks(angles[:-1])
    ax.set_xticklabels(categories)
    ax.set_ylim(0, 1)
    ax.set_title('Your Music Audio Profile', fontsize=14, fontweight='bold', y=1.08)
    
    plt.tight_layout()
    plt.savefig(output_path, dpi=150, bbox_inches='tight')
    plt.close()
    
    return output_path
```

---

## Error Handling

### Common Errors

| Error Code | Meaning | Action |
|------------|---------|--------|
| 400 | Bad Request | Check request parameters, report to user |
| 401 | Unauthorized | Refresh access token, retry |
| 403 | Forbidden | Scope missing or resource not accessible, inform user |
| 404 | Not Found | Resource doesn't exist, inform user |
| 429 | Rate Limited | Wait for Retry-After seconds, then retry |
| 500 | Server Error | Retry after 5 seconds, max 3 attempts |
| 502/503 | Service Unavailable | Retry after 10 seconds, max 3 attempts |

### Error Recovery

```python
def spotify_request(endpoint, method='GET', data=None, retry_count=0):
    """Make Spotify API request with error handling."""
    MAX_RETRIES = 3
    
    try:
        response = make_request(endpoint, method, data)
        
        if response.status_code == 200:
            return response.json()
        
        elif response.status_code == 401:
            # Token expired - refresh and retry
            refresh_token()
            return spotify_request(endpoint, method, data, retry_count)
        
        elif response.status_code == 429:
            # Rate limited
            wait_time = int(response.headers.get('Retry-After', 30))
            print(f"Rate limited. Waiting {wait_time} seconds...")
            sleep(wait_time)
            return spotify_request(endpoint, method, data, retry_count)
        
        elif response.status_code >= 500:
            # Server error - retry with backoff
            if retry_count < MAX_RETRIES:
                sleep(5 * (retry_count + 1))
                return spotify_request(endpoint, method, data, retry_count + 1)
            else:
                raise Exception(f"Server error after {MAX_RETRIES} retries")
        
        else:
            # Client error - don't retry
            raise Exception(f"API error {response.status_code}: {response.text}")
            
    except ConnectionError:
        if retry_count < MAX_RETRIES:
            sleep(5)
            return spotify_request(endpoint, method, data, retry_count + 1)
        raise
```

### User Communication

- Always inform user of errors in plain language
- Offer solutions or alternatives when possible
- For rate limits during analysis: "Processing your library is taking a moment due to API limits. Currently at X/Y tracks..."
- For auth errors: Guide user through re-authentication

---

## Sample Workflows

### Workflow 1: Full Library Analysis

```
User: "Analyze my Spotify library"

Agent:
1. Check cache status → Ask about fresh/existing cache
2. Sync library (batched mode):
   - Fetch all saved tracks (paginate with limit=50)
   - Report progress: "Syncing library... 500/1247 tracks"
3. Fetch audio features (batched, 100 per request)
4. Fetch artist genres (batched, 50 per request)  
5. Build genre mappings
6. Generate analysis report:
   - Total stats
   - Genre breakdown
   - Audio profile
   - Time-based trends
7. Generate charts (genre pie, audio radar, etc.)
8. Present findings with visualizations
```

### Workflow 2: Auto-Organize by Genre

```
User: "Organize my liked songs into playlists by genre"

Agent:
1. Ensure library is synced and genre-mapped
2. Group tracks by primary genre
3. For each genre with 5+ tracks:
   - Check for existing "Auto: {Genre}" playlist
   - If exists: Show details, ask to add new tracks
   - If not: Propose new playlist creation
4. Ask: "Would you like to review each playlist, or use batch mode?"
5. If batch mode:
   - Summarize all proposed actions
   - Execute on confirmation
   - Provide final summary
6. If individual:
   - Present each playlist proposal
   - Wait for confirmation
   - Execute and move to next
```

### Workflow 3: Criteria-Based Addition

```
User: "Add all the jazz songs I saved this month to my 'Late Night Jazz' playlist"

Agent:
1. Parse criteria:
   - Genre: Jazz
   - Date range: This month (added_at filter)
   - Target playlist: "Late Night Jazz"
2. Query library (use cache):
   - Filter by genre mapping = "Jazz"
   - Filter by added_at >= start of month
3. Find "Late Night Jazz" playlist:
   - Search user playlists
   - Present details for confirmation
4. Check for duplicates:
   - Get playlist tracks
   - Compare with filtered tracks
5. Present summary:
   - "Found 12 jazz tracks from this month"
   - "3 are already in 'Late Night Jazz'"
   - "Add 9 new tracks?"
6. On confirmation: Add tracks
7. Update cache
```

### Workflow 4: Discovery Recommendations

```
User: "Recommend some new music based on my jazz collection"

Agent:
1. Identify jazz tracks in library
2. Analyze patterns:
   - Top jazz artists
   - Audio feature averages for jazz tracks
3. Select seeds:
   - 2-3 top jazz artists
   - 2 representative jazz tracks
4. Set targets:
   - target_acousticness based on library average
   - target_instrumentalness
5. Call /recommendations:
   - Seeds + targets
   - Limit: 20
6. Filter results:
   - Remove tracks already in library
   - Remove tracks by artists heavily represented
7. Present:
   - "Here are 15 jazz recommendations based on your taste:"
   - List with artist, track, and why recommended
8. Offer actions:
   - "Would you like to save any to your library?"
   - "Create a 'Jazz Discovery' playlist?"
```

---

## Security Notes

- Never log or display full access tokens
- Store refresh tokens securely
- Clear tokens from memory after session
- Warn users not to share their credentials
- Cache files may contain personal data - remind users

---

## Limitations

- Library size limit: Can only access first ~10,000 saved tracks via offset pagination
- Recently played: Only last 50 unique tracks available
- Genre data: Not available for all artists; may need fallback strategies
- Recommendations: Limited to 5 seeds total per request
- Playlist track limit: 10,000 tracks per playlist
- Audio features: Not available for all tracks (local files, some regional content)

---

## Quick Reference

### Essential Endpoints

| Task | Endpoint | Max Batch |
|------|----------|-----------|
| Get saved tracks | GET /me/tracks | 50/req |
| Get audio features | GET /audio-features | 100 IDs |
| Get artists | GET /artists | 50 IDs |
| Create playlist | POST /users/{id}/playlists | - |
| Add to playlist | POST /playlists/{id}/tracks | 100 URIs |
| Get recommendations | GET /recommendations | 100 results |
| Get top items | GET /me/top/{type} | 50/req |

### Rate Limit Quick Guide

- User requests: Go fast, handle 429s
- Analysis: Batch + 150ms delays
- Complex ops: Adaptive (start fast, slow if needed)
- Target: Stay under 180 req/min

### Cache Priority

1. Audio features (immutable, always use cache)
2. Artist genres (7-day TTL)
3. Track metadata (30-day TTL)
4. Playlist contents (1-hour TTL or snapshot check)