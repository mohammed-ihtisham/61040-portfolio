# Assignment 2 — Functional Design

## Problem Statement

**Domain: Nearby Place Discovery**

I love exploring new places, especially aesthetic nature spots, scenic overlooks, and unique food spots. Finding a hidden trail with a great view or a small restaurant that becomes a favorite feels exciting and memorable. I often go on spontaneous drives with friends or family just to see what we can discover, and those trips usually become some of my favorite memories. For me, nearby place discovery is about making exploration feel easy and fun without having to travel far or visit another city or country.

**Problem: Overwhelm from Generic Reviews and Ratings**

Many people like me struggle when trying to find new nearby places. Platforms like Google Maps, Yelp, and Instagram flood us with endless lists and repetitive reviews that all start to look the same. Even with ratings and photos, it is hard to tell which spots are truly worth visiting. Instead of feeling inspired, we spend too much time scrolling and often end up defaulting to the same familiar places. What should be a fun and energizing process of discovery becomes a stressful chore that discourages exploration and takes away the sense of adventure.

**Stakeholders**

- ***Explorer***: The primary user looking for places to visit who experiences frustration and decision fatigue from review overload.
- ***Platform Operator***: The company hosting the discovery service that loses user trust and engagement when the discovery process feels tedious and unhelpful.
- ***Local Business Owner***: The venue operator whose unique qualities get lost in generic rating systems and may be overlooked despite offering great experiences.
  
**Evidence & Comparables**

- [Consumers abandon purchases due to review overload (***Evidence***)](https://www.customerexperiencedive.com/news/information-overload-reviews-overwhelm-shopping-expereience/720658/): Nearly three-quarters of consumers walk away from purchases because too many reviews made decision-making difficult, showing how review overwhelm directly leads to user abandonment.
- [Published research shows generic reviews diminish impact (***Evidence***)](https://www.customerexperiencedive.com/news/information-overload-reviews-overwhelm-shopping-expereience/720658/): Studies find that when reviews are too numerous or repetitive, they lose practical value. Instead of aiding decisions, they create cognitive dissonance and make choices harder.
- [Choice overload reduces decision satisfaction (***Evidence***)](https://www.researchgate.net/publication/265170803_Choice_Overload_A_Conceptual_Review_and_Meta-Analysis): A meta-analysis of 99 studies found that excessive options reduce confidence and satisfaction, paralleling how long review lists overwhelm people choosing places to visit.
- [Yelp • Local business reviews (***Comparable***)](https://www.yelp.com): Yelp provides massive coverage of local spots through crowdsourced reviews. However, its long lists of repetitive text often cause decision fatigue instead of simplifying discovery.  
- [Google Maps • Crowdsourced ratings (***Comparable***)](https://maps.google.com): Google Maps integrates ratings with directions, making it convenient. But its brevity and prevalence of fake or generic reviews make it unreliable for nuanced decisions.  
- [Tripadvisor • Travel-focused reviews (***Comparable***)](https://www.tripadvisor.com): Tripadvisor helps tourists find attractions with detailed reviews. However, its skew toward out-of-town voices makes it less useful for locals seeking everyday places.

## Application Pitch

**Name: Unwindr**

**Motivation:** Unwindr solves the problem of decision fatigue by replacing long, repetitive review lists with fast, visual discovery that lets users get an instant sense of what nearby places are like.

**Key Features:**

***Interactive Media Gallery:*** Instead of scrolling through text-heavy reviews, users browse an interactive map where each place has a gallery of authentic photos and media. This feature lets users visually assess whether a place matches their vibe within seconds. It benefits explorers by speeding up decisions, local businesses by showcasing their atmosphere directly, and platform operators by increasing engagement through a more intuitive browsing experience.

***Smart Interest Filtering:*** Users set personal preferences like "quiet study spots," "lively group dining," or "Instagram-worthy views" to see only relevant places. This feature reduces cognitive overload by eliminating irrelevant options before users even see them. Explorers save time and feel more confident about their choices, local businesses get matched with their ideal customers, and platform operators see higher conversion rates as users find places that truly fit their needs.

***Quality-Based Recommendations:*** Places are ranked not only by popularity but also by how engaged users are with their media relative to visit volume. This allows under-the-radar but well-loved places to rise to the top, helping users discover authentic local favorites before they become overexposed. This rewards venues that deliver a great experience, keeps discovery fresh for repeat users.

## Concept Design 

### Concept: `UserAuth`
```markdown
concept UserAuth [User]
purpose authenticate users and manage permissions so only authorized users can contribute or moderate content
principle users register and sign in to access actions; moderators have special privileges for
  content quality control

state
  a set of Users with
    a username String
    a password String
    a canModerate Flag

actions
  registerUser (username: String, password: String) : (user: User)
    requires username is unique
    effect creates a new user with default permissions (cannot moderate)
  
  login (username: String, password: String) : (user: User)
    requires user exists and password matches
    effect returns authenticated user
  
  grantModerator (user: User, admin: User)
    requires admin has moderation privileges and user exists
    effect sets canModerate to true for user
```

### Concept: `PlaceCatalog `
```markdown
concept PlaceCatalog [User]
purpose maintain essential information about discoverable places
principle can be preloaded from a provider or added by users to ensure comprehensive coverage

state
  a set of Places with
    a name String
    a address String
    a category String
    a verified Flag
    a addedBy User
    a location Location
    a source String // "provider" or "user_added"
  
  a set of Locations with
    a latitude Number
    a longitude Number

actions
  seedPlaces (centerLat: Number, centerLng: Number, radius: Number)
    requires coordinates are valid and radius > 0
    effect loads places from provider within specified area
  
  addPlace (user: User, name: String, address: String, category: String, lat: Number, lng: Number) : (place: Place)
    requires user is authenticated and name is not empty and coordinates are valid
    effect creates user-added place
  
  verifyPlace (place: Place, user: User)
    requires place exists and user has moderation privileges
    effect marks place as verified
  
  updatePlace (place: Place, name: String, address: String, user: User)
    requires place exists and user is authenticated
    effect updates place details
  
  getPlacesInArea (centerLat: Number, centerLng: Number, radius: Number) : (places: set Place)
    requires coordinates are valid and radius > 0
    effect returns nearby places
```
Note: The system proactively populates areas with places from Google Maps API in `seedPlaces()` to ensure comprehensive venue coverage from launch, eliminating the cold start problem where users would find empty maps.

### Concept: `MediaGallery`
```markdown
concept MediaGallery [User, Place]
purpose enable visual discovery of places with authentic photos organized spatially
principle the system starts with places seeded from Google Maps API to provide comprehensive coverage
  though it algorithmically surfaces most-engaging user photos first for instant visual context

state
  a set of MediaItems with
    a place Place
    a contributor User
    a timestamp Date
    a imageUrl String
    a source String // "provider" or "user_contributed"
    a engagementScore Number
  
  a set of MediaInteractions with
    a mediaItem MediaItem
    a user User
    a interactionType String // "view", "tap", "like", "share"
    a timestamp Date

actions
  seedMedia (place: Place, images: set String)
    requires place exists and images is not empty
    effect creates provider-sourced media
  
  addMedia (user: User, place: Place, imageUrl: String) : (mediaItem: MediaItem)
    requires user is authenticated and place exists and imageUrl is valid
    effect user contributes a new media item
  
  recordInteraction (user: User, mediaItem: MediaItem, interactionType: String)
    requires user is authenticated and mediaItem exists
    effect logs engagement and updates score
  
  reactToMedia (user: User, mediaItem: MediaItem, interaction: interactionType)
    requires user is authenticated and mediaItem exists and user has not already liked
    effect boosts engagement score
  
  getRankedMedia (place: Place) : (media: set MediaItem)
    requires place exists
    effect returns media sorted by engagement score
```
Note: The system automatically populates each place with photos from Google Maps API in `seedMedia()` to ensure immediate visual content. The recommendation algorithm combines engagement metrics (views, taps, likes, shares) with user interaction patterns to surface the most compelling media first.

### Concept: `InterestFilter`
```markdown
concept  InterestFilter [User, Place]
purpose allow users to express their interests so places can be filtered to match their preferences
principle users pick preference tags, and only matching places are shown to minimize irrelevant options

state
  a set of UserPreferences with
    a user User
    a tags set String
  
  a set of PlaceTags with
    a place Place
    a tags set String

actions
  setPreferences (user: User, tags: set String)
    requires user is authenticated and tags is not empty
    effect updates user's selected interest tags
  
  tagPlace (place: Place, tag: String)
    requires place exists and tag is valid
    effect associates tag with place
  
  getMatchingPlaces (user: User, places: set Place) : (matches: set Place)
    requires user has preferences set
    effect filters places by overlapping tags with user preferences
```

### Concept: `QualityRanking`
```markdown
concept  QualityRanking [Place]
purpose surface lesser-known places that have high engagement but low mainstream visibility
principle compute a score that promotes well-loved places regardless of popularity,
  helping users discover authentic local favorites

state
  a set of RankingMetrics with
    a place Place
    a engagementRatio Number
    a visitorVolume Number
    a qualityScore Number
    a lastUpdated Date
  
  a set of RankingPreferences with
    a user User
    a prefersEmergent Flag
    a explorationRadius Number

actions
  calculateQualityScore (place: Place) : (score: Number)
    requires place exists and has engagement data
    effect computes engagement-to-visit ratio
  
  updateMetrics (place: Place, visits: Number, engagement: Number)
    requires place exists and visits >= 0 and engagement >= 0
    effect refreshes ranking metrics
  
  setPreferences (user: User, prefersEmergent: Boolean, radius: Number)
    requires user is authenticated and radius > 0
    effect enables discovery of hidden spots
  
  getRecommendedPlaces (user: User, centerLat: Number, centerLng: Number) : (places: set Place)
    requires user has ranking preferences and coordinates are valid
    effect returns places within radius ranked by quality score
```

## Synchronizations

```markdown
sync authenticationEnablement
when UserAuth.login (username, password) succeeds
then enable PlaceCatalog.addPlace and MediaGallery.addMedia for user

sync placeAddition
when PlaceCatalog.addPlace (user, name, address, category, lat, lng)
then MediaGallery.seedMedia (place, getProviderImages(place.location))
     QualityRanking.updateMetrics (place, 0, 0)

sync mediaEngagement
when MediaGallery.reactToMedia (user, mediaItem, interaction)
then QualityRanking.updateMetrics (mediaItem.place, getVisits(mediaItem.place), mediaItem.engagementScore)

sync personalizedDiscovery
when InterestFilter.getMatchingPlaces (user, places)
then QualityRanking.getRecommendedPlaces (user, centerLat, centerLng) using filtered results
```

## Concept Integration Notes:

The UserAuth concept ensures that all user actions are gated behind authentication, so only logged-in users can contribute places or media. It also grants moderators elevated permissions to verify places and remove inappropriate media, maintaining data quality and trust.

The PlaceCatalog concept provides the foundational set of discoverable places. It ensures users always have a populated map by seeding locations from a provider, while allowing users to add new venues when something is missing. This decouples the data layer from visual presentation, allowing other concepts to build on consistent place information.

The MediaGallery concept attaches visual media to each place and ranks media items based on engagement. It powers the core user experience of visually exploring places and gives immediate context about what a location feels like without needing to read reviews. The engagement tracking enables other features to leverage this data independently.

The InterestFilter concept personalizes discovery results using tags to filter places. It operates independently of MediaGallery and PlaceCatalog internals, keeping filtering logic generic and reusable.

The QualityRanking concept solves a critical gap in traditional discovery platforms by computing scores based on engagement relative to visitor volume rather than raw popularity. This allows the system to surface "hidden gems" - places that locals love but haven't been overwhelmed by mainstream attention. Users can opt into emergent place discovery through ranking preferences, ensuring they find authentic experiences before they become overcrowded.

## UI Sketches
### Main Discovery View — Interactive Map
![Note Sep 28, 2025](https://github.com/user-attachments/assets/ec4ff0e2-3342-43ab-b7cd-3ce5193a4531)

### Filter & Results Sidebar — Personalized Discovery
![IMG_84F26C0B65D7-1](https://github.com/user-attachments/assets/9f8c3382-6c7f-4f22-94b5-060379e7a204)

### Distance Filter — Adjustable Radius
![IMG_D6987010C919-1](https://github.com/user-attachments/assets/cf408a6e-7f3c-49c6-a8ab-9670b5c7a321)

### Interest Filter — Tag Selection
![IMG_1BD72C00E29A-1](https://github.com/user-attachments/assets/96f7ebc4-e47c-4066-8a58-920f97a7d66d)

### Place Detail View — Media Gallery Focus
![Note Sep 28, 2026](https://github.com/user-attachments/assets/b17b6d17-03b7-4416-a057-5e290e2d49cc)

### Full-Screen Media Viewer
![IMG_878A6D0C0362-1](https://github.com/user-attachments/assets/da8ae5c1-c968-43de-9f0b-9382c319c6d2)

## User journey

A user is planning a Saturday afternoon outing and wants to find a place filled with natural scenery, somewhere outside of New York City but not too far to feel like a peaceful escape. They are not interested in well-known parks or crowded tourist spots, but rather a quiet destination where they can enjoy fresh air, open views, and a slower pace. They want a place where they can relax, take in the scenery, and feel like they have discovered something special.

When they open Unwindr, they are greeted with a full-screen interactive map centered on their current location, with colorful media gallery thumbnails marking different destinations nearby. They tap the filter icon and adjust the search radius to include areas slightly outside the city. Then they choose interest tags such as “nature walks,” “waterfront views,” and “quiet spaces.” Finally, they toggle on the hidden gem discovery mode to focus on lesser-known places that locals appreciate but are not crowded with visitors. The map updates instantly, dimming locations that do not match and highlighting a smaller set of destinations that fit their preferences.

One gallery thumbnail near Larchmont catches their eye because it shows a glimpse of the coastline. They tap on it, and a preview panel opens on the right side of the screen showing a ranked media gallery. The images include grassy picnic areas, winding walking paths under tall trees, and a view of the Long Island Sound. They swipe through the photos and notice candid images of people sitting with books, children exploring the shoreline, and sailboats in the distance. The overall feeling is calm and inviting, exactly what they had been hoping for.

Because Unwindr ranks locations by engagement quality instead of just popularity, Larchmont Manor Park appears near the top of the recommendations. The user shares it with friends using the share icon, which generates a Google Maps link. Later that day, they spend a few hours walking the paths, watching the sunset over the water, and taking photos of their favorite spots. When they return home, they like the destination by hearting and upload a few of those photos into Unwindr to help other explorers discover this quiet, scenic location in the future.
