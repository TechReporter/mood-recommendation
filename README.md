# Architecture Diagrams - Mood-Based Recommendation System

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "User Interface Layer"
        UI[Streamlit Web App<br/>app.py]
        COMP[UI Components<br/>ui_components.py]
        STYLE[Styling<br/>styles.py]
        STATE[State Manager<br/>state_manager.py]
    end
    
    subgraph "Application Logic Layer"
        QUERY[Query Builder<br/>utils.py]
        MOOD[Mood Taxonomy<br/>mood_taxonomy.py]
        WHY[Why Watch Generator<br/>why_watch_generator.py]
    end
    
    subgraph "Search Engine Core"
        SEARCH[Search Engine<br/>search_engine.py]
        BM25[BM25 Index<br/>Keyword Matching]
        SEM[Semantic Index<br/>Vector Embeddings]
        FAISS[FAISS Vector DB<br/>Fast Similarity Search]
    end
    
    subgraph "Ranking & Scoring"
        RRF[Reciprocal Rank Fusion<br/>RRF Algorithm]
        SCORE[Multi-Signal Scoring<br/>BM25 + Semantic + Boosts]
        RERANK[Two-Stage Re-Ranking]
    end
    
    subgraph "Data & External Services"
        CSV[Content Catalog<br/>CSV Database]
        TMDB[TMDb API<br/>Genre Mapping & Details]
        CACHE[Genre Cache<br/>JSON File]
    end
    
    UI --> COMP
    UI --> STYLE
    UI --> STATE
    UI --> QUERY
    QUERY --> MOOD
    UI --> SEARCH
    
    SEARCH --> BM25
    SEARCH --> SEM
    SEM --> FAISS
    
    BM25 --> RRF
    SEM --> RRF
    RRF --> SCORE
    SCORE --> RERANK
    RERANK --> COMP
    
    SEARCH --> CSV
    SEARCH --> TMDB
    TMDB --> CACHE
    COMP --> WHY
    COMP --> TMDB
    
    style UI fill:#667eea,stroke:#764ba2,color:#fff
    style SEARCH fill:#f093fb,stroke:#4facfe,color:#fff
    style RRF fill:#fa709a,stroke:#fee140,color:#fff
    style CSV fill:#30cfd0,stroke:#330867,color:#fff
```

## 2. Detailed Query Processing Flow

```mermaid
flowchart TD
    START([User Input]) --> INPUT{Input Type?}
    
    INPUT -->|Free Text| TEXT[User Text Query]
    INPUT -->|Mood Chip| MOOD[Mood Chip Selected]
    INPUT -->|Both| COMBINE[Combine Text + Mood]
    
    MOOD --> EXPAND[Mood Expansion<br/>mood_taxonomy.py]
    EXPAND --> QUERY[Build Effective Query]
    TEXT --> QUERY
    COMBINE --> QUERY
    
    QUERY --> PREPROC[Query Preprocessing<br/>Stop Word Removal<br/>Tokenization]
    PREPROC --> EXPAND_Q[Query Expansion<br/>Add Synonyms]
    
    EXPAND_Q --> PARALLEL[Parallel Score Computation]
    
    PARALLEL --> BM25_PROC[BM25 Processing]
    PARALLEL --> SEM_PROC[Semantic Processing]
    PARALLEL --> BOOST[Boost Calculation]
    
    BM25_PROC --> BM25_TOKEN[Tokenize Query]
    BM25_TOKEN --> BM25_SCORE[BM25 Score<br/>Keyword Matching]
    BM25_SCORE --> BM25_NORM[Normalize to 0-1]
    
    SEM_PROC --> SEM_ENCODE[Encode Query to Vector<br/>SentenceTransformer]
    SEM_ENCODE --> SEM_SEARCH[Vector Similarity Search<br/>FAISS or NumPy]
    SEM_SEARCH --> SEM_NORM[Normalize to 0-1]
    
    BOOST --> TITLE_BOOST[Title Boost<br/>Jaccard Similarity]
    BOOST --> DESC_BOOST[Description Boost<br/>Jaccard Similarity]
    BOOST --> FRESH_BOOST[Freshness Boost<br/>Release Date Based]
    
    BM25_NORM --> DETECT[Query Type Detection]
    SEM_NORM --> DETECT
    TITLE_BOOST --> DETECT
    DESC_BOOST --> DETECT
    FRESH_BOOST --> DETECT
    
    DETECT --> TYPE{Query Type?}
    
    TYPE -->|Mood-Based<br/>3+ mood keywords| MOOD_WEIGHT[Adaptive Weights<br/>20% BM25, 50% Semantic<br/>15% Title, 5% Desc, 10% Fresh]
    TYPE -->|Free-Text<br/>Fewer mood keywords| TEXT_WEIGHT[Adaptive Weights<br/>30% BM25, 40% Semantic<br/>15% Title, 5% Desc, 10% Fresh]
    
    MOOD_WEIGHT --> COMBINE_SCORE[Calculate Combined Scores]
    TEXT_WEIGHT --> COMBINE_SCORE
    
    COMBINE_SCORE --> RRF_STAGE1[Stage 1: RRF Fusion<br/>k=20.0]
    
    RRF_STAGE1 --> TOP100[Get Top 100 Candidates]
    TOP100 --> RERANK_STAGE2[Stage 2: Re-Rank by<br/>Combined Scores]
    
    RERANK_STAGE2 --> FILTER[Result Filtering]
    
    FILTER --> THRESHOLD{Score > 0.1?}
    THRESHOLD -->|No| SKIP[Skip Item]
    THRESHOLD -->|Yes| TYPE_FILTER{Type Match?}
    
    TYPE_FILTER -->|No| SKIP
    TYPE_FILTER -->|Yes| DUP{Duplicate Title?}
    
    DUP -->|Yes| SKIP
    DUP -->|No| ADD[Add to Results]
    
    ADD --> TOPK{Top 10<br/>Reached?}
    TOPK -->|No| FILTER
    TOPK -->|Yes| RESULTS[Return Results<br/>Top 10 Movies + Top 10 Shows]
    
    SKIP --> FILTER
    
    RESULTS --> DISPLAY[Display Results<br/>Posters, Titles, Genres]
    DISPLAY --> END([End])
    
    style START fill:#667eea,stroke:#764ba2,color:#fff
    style PARALLEL fill:#f093fb,stroke:#4facfe,color:#fff
    style RRF_STAGE1 fill:#fa709a,stroke:#fee140,color:#fff
    style RESULTS fill:#30cfd0,stroke:#330867,color:#fff
    style END fill:#667eea,stroke:#764ba2,color:#fff
```

## 3. Data Flow Architecture

```mermaid
flowchart LR
    subgraph "Data Sources"
        TMDB_API[TMDb API<br/>External Service]
        CSV_FILE[CSV Catalog<br/>data/hbo_sample_titles.csv]
    end
    
    subgraph "Data Ingestion"
        FETCH[tmdb_fetch.py<br/>Data Fetcher]
        PARSE[CSV Parser<br/>Pandas DataFrame]
    end
    
    subgraph "Data Processing"
        ITEMS[CatalogItem Objects<br/>Structured Data]
        BM25_CORPUS[BM25 Corpus<br/>Weighted Text]
        SEM_CORPUS[Semantic Corpus<br/>Text for Embeddings]
    end
    
    subgraph "Index Building"
        BM25_IDX[BM25 Index<br/>rank-bm25]
        EMBED[Vector Embeddings<br/>768-dim Vectors]
        FAISS_IDX[FAISS Index<br/>Vector Database]
    end
    
    subgraph "Runtime Query"
        QUERY[User Query]
        BM25_SEARCH[BM25 Search]
        SEM_SEARCH[Semantic Search]
        FUSION[Score Fusion]
    end
    
    subgraph "Results"
        RANKED[Ranked Results]
        UI_DISPLAY[UI Display]
    end
    
    TMDB_API --> FETCH
    FETCH --> CSV_FILE
    CSV_FILE --> PARSE
    PARSE --> ITEMS
    
    ITEMS --> BM25_CORPUS
    ITEMS --> SEM_CORPUS
    
    BM25_CORPUS --> BM25_IDX
    SEM_CORPUS --> EMBED
    EMBED --> FAISS_IDX
    
    QUERY --> BM25_SEARCH
    QUERY --> SEM_SEARCH
    
    BM25_IDX --> BM25_SEARCH
    FAISS_IDX --> SEM_SEARCH
    
    BM25_SEARCH --> FUSION
    SEM_SEARCH --> FUSION
    FUSION --> RANKED
    RANKED --> UI_DISPLAY
    
    style TMDB_API fill:#ff6b6b,stroke:#ee5a6f,color:#fff
    style CSV_FILE fill:#4ecdc4,stroke:#44a08d,color:#fff
    style BM25_IDX fill:#ffe66d,stroke:#ff6b6b,color:#000
    style FAISS_IDX fill:#95e1d3,stroke:#f38181,color:#000
    style FUSION fill:#aa96da,stroke:#c7ceea,color:#fff
    style UI_DISPLAY fill:#667eea,stroke:#764ba2,color:#fff
```

## 4. Component Interaction Sequence

```mermaid
sequenceDiagram
    participant User
    participant UI as Streamlit UI
    participant Query as Query Builder
    participant Search as Search Engine
    participant BM25 as BM25 Index
    participant Semantic as Semantic Index
    participant RRF as RRF Fusion
    participant Ranker as Re-Ranker
    participant Display as UI Components
    
    User->>UI: Enter Query / Select Mood
    UI->>Query: Build Effective Query
    Query->>Query: Expand Mood Terms
    Query->>Query: Preprocess Query
    Query->>Query: Expand Synonyms
    Query-->>Search: Processed Query
    
    Search->>BM25: Get BM25 Scores
    BM25-->>Search: BM25 Scores (normalized)
    
    Search->>Semantic: Get Semantic Scores
    Semantic->>Semantic: Encode Query Vector
    Semantic->>Semantic: FAISS/NumPy Search
    Semantic-->>Search: Semantic Scores (normalized)
    
    Search->>Search: Calculate Boosts<br/>(Title, Desc, Freshness)
    Search->>Search: Detect Query Type
    Search->>Search: Apply Adaptive Weights
    
    Search->>RRF: Fuse Rankings
    RRF->>RRF: Calculate RRF Scores
    RRF-->>Search: Top 100 Candidates
    
    Search->>Ranker: Re-Rank Top 100
    Ranker->>Ranker: Sort by Combined Scores
    Ranker-->>Search: Final Rankings
    
    Search->>Search: Filter Results<br/>(Threshold, Type, Duplicates)
    Search-->>UI: Top 10 Movies + Top 10 Shows
    
    UI->>Display: Render Results
    Display->>Display: Get Genre Names
    Display->>Display: Generate Why Watch
    Display-->>User: Display Posters, Titles, Details
```

## 5. Scoring Algorithm Flow

```mermaid
flowchart TD
    START([Query Input]) --> PREP[Preprocess & Expand Query]
    
    PREP --> BM25[BM25 Score Calculation]
    PREP --> SEM[Semantic Score Calculation]
    PREP --> BOOSTS[Boost Calculations]
    
    BM25 --> BM25_NORM[Normalize BM25<br/>Range: 0-1]
    
    SEM --> SEM_ENCODE[Encode to 768-dim Vector]
    SEM_ENCODE --> SEM_SIM[Cosine Similarity Search]
    SEM_SIM --> SEM_NORM[Normalize Semantic<br/>Range: 0-1]
    
    BOOSTS --> TITLE[Title Boost<br/>Jaccard Similarity<br/>+0.3 for exact phrase]
    BOOSTS --> DESC[Description Boost<br/>Jaccard Similarity × 0.5]
    BOOSTS --> FRESH[Freshness Boost<br/>Based on Release Date]
    
    BM25_NORM --> DETECT[Query Type Detection]
    SEM_NORM --> DETECT
    TITLE --> DETECT
    DESC --> DETECT
    FRESH --> DETECT
    
    DETECT --> CHECK{3+ Mood Keywords<br/>OR 5+ Words?}
    
    CHECK -->|Yes| MOOD_WEIGHTS[Mood-Based Weights<br/>20% BM25<br/>50% Semantic<br/>15% Title<br/>5% Description<br/>10% Freshness]
    
    CHECK -->|No| TEXT_WEIGHTS[Free-Text Weights<br/>30% BM25<br/>40% Semantic<br/>15% Title<br/>5% Description<br/>10% Freshness]
    
    MOOD_WEIGHTS --> COMBINE[Weighted Combination<br/>Final Score = Σ(weight × score)]
    TEXT_WEIGHTS --> COMBINE
    
    COMBINE --> RRF[RRF Fusion<br/>k=20.0]
    
    RRF --> RANK1[Rank by BM25]
    RRF --> RANK2[Rank by Semantic]
    RANK1 --> RRF_CALC[RRF Score = Σ(1/(k+rank+1))]
    RANK2 --> RRF_CALC
    
    RRF_CALC --> TOP100[Get Top 100 by RRF]
    TOP100 --> RERANK[Re-Rank by Combined Scores]
    
    RERANK --> FILTER[Filter Results]
    FILTER --> THRESHOLD{Score ≥ 0.1?}
    THRESHOLD -->|No| SKIP[Skip]
    THRESHOLD -->|Yes| TYPE{Type Match?}
    
    TYPE -->|No| SKIP
    TYPE -->|Yes| DUP{Duplicate?}
    
    DUP -->|Yes| SKIP
    DUP -->|No| RESULT[Add to Results]
    
    RESULT --> TOPK{Top 10?}
    TOPK -->|No| FILTER
    TOPK -->|Yes| OUTPUT([Final Results])
    
    SKIP --> FILTER
    
    style START fill:#667eea,stroke:#764ba2,color:#fff
    style COMBINE fill:#f093fb,stroke:#4facfe,color:#fff
    style RRF fill:#fa709a,stroke:#fee140,color:#fff
    style OUTPUT fill:#30cfd0,stroke:#330867,color:#fff
```

## 6. System Initialization Flow

```mermaid
flowchart TD
    START([App Startup]) --> CONFIG[Page Configuration<br/>Streamlit Setup]
    
    CONFIG --> INIT_STATE[Initialize Session State<br/>search_triggered, selected_mood, etc.]
    
    INIT_STATE --> LOAD_ENGINE[Load Search Engine<br/>@st.cache_resource]
    
    LOAD_ENGINE --> READ_CSV[Read CSV Catalog<br/>data/hbo_sample_titles.csv]
    
    READ_CSV --> PARSE[Parse to CatalogItem Objects]
    
    PARSE --> BUILD_BM25[Build BM25 Index]
    BUILD_BM25 --> BM25_CORPUS[Create Weighted Corpus<br/>Title×3 + Desc×2 + Genre + Mood]
    BM25_CORPUS --> BM25_TOKEN[Tokenize Corpus]
    BM25_TOKEN --> BM25_INDEX[Initialize BM25Okapi]
    
    PARSE --> BUILD_SEM[Build Semantic Index]
    BUILD_SEM --> SEM_CORPUS[Create Semantic Corpus<br/>Title×2 + Desc + Genre + Mood]
    SEM_CORPUS --> LOAD_MODEL[Load SentenceTransformer<br/>all-mpnet-base-v2]
    LOAD_MODEL --> ENCODE[Encode All Items<br/>768-dim Vectors]
    ENCODE --> NORMALIZE[Normalize Embeddings<br/>L2 Normalization]
    
    NORMALIZE --> CHECK_SIZE{Items > 10k?}
    
    CHECK_SIZE -->|Yes| BUILD_IVF[Build FAISS IVF Index<br/>Inverted File Index]
    CHECK_SIZE -->|No| BUILD_FLAT[Build FAISS Flat Index<br/>Exact Search]
    
    BUILD_IVF --> STORE[Store Embeddings & Index]
    BUILD_FLAT --> STORE
    
    STORE --> PRE_COMPUTE[Pre-compute Freshness Scores<br/>Based on Release Dates]
    
    PRE_COMPUTE --> GENRE_INIT[Initialize Genre Mappings]
    GENRE_INIT --> CHECK_CACHE{Cache Exists?}
    
    CHECK_CACHE -->|Yes| LOAD_CACHE[Load from Cache<br/>genre_cache.json]
    CHECK_CACHE -->|No| FETCH_GENRE[Fetch from TMDb API]
    
    FETCH_GENRE --> SAVE_CACHE[Save to Cache]
    LOAD_CACHE --> READY[Search Engine Ready]
    SAVE_CACHE --> READY
    
    READY --> UI_READY[UI Ready for User Input]
    
    style START fill:#667eea,stroke:#764ba2,color:#fff
    style BUILD_BM25 fill:#f093fb,stroke:#4facfe,color:#fff
    style BUILD_SEM fill:#fa709a,stroke:#fee140,color:#fff
    style READY fill:#30cfd0,stroke:#330867,color:#fff
    style UI_READY fill:#667eea,stroke:#764ba2,color:#fff
```

## 7. Similar Content Discovery Flow

```mermaid
flowchart TD
    START([User Clicks<br/>Find Similar]) --> GET_ID[Get Item ID<br/>from Session State]
    
    GET_ID --> FIND_ITEM[Find Item in Catalog<br/>by ID]
    
    FIND_ITEM --> CHECK{Item Found?}
    CHECK -->|No| ERROR[Return Empty Results]
    CHECK -->|Yes| GET_EMBED[Get Item's Embedding<br/>from Embeddings Array]
    
    GET_EMBED --> SEARCH_SIM[Search for Similar Items]
    
    SEARCH_SIM --> CHECK_FAISS{FAISS Available?}
    
    CHECK_FAISS -->|Yes| FAISS_SEARCH[FAISS Vector Search<br/>IndexFlatIP or IndexIVFFlat]
    CHECK_FAISS -->|No| NUMPY_SEARCH[NumPy Dot Product<br/>Slower but Works]
    
    FAISS_SEARCH --> GET_TOP[Get Top k×3 Candidates<br/>k = desired results]
    NUMPY_SEARCH --> GET_TOP
    
    GET_TOP --> FILTER_SELF[Filter Out Original Item]
    FILTER_SELF --> FILTER_TYPE{Type Filter<br/>Applied?}
    
    FILTER_TYPE -->|Yes| CHECK_TYPE[Check Item Type<br/>movie or show]
    FILTER_TYPE -->|No| ADD_RESULT[Add to Results]
    
    CHECK_TYPE --> MATCH{Type Matches?}
    MATCH -->|Yes| ADD_RESULT
    MATCH -->|No| SKIP[Skip Item]
    
    ADD_RESULT --> COUNT{Reached<br/>Top k?}
    COUNT -->|No| GET_TOP
    COUNT -->|Yes| RETURN[Return Similar Items<br/>List of CatalogItem, score]
    
    SKIP --> COUNT
    ERROR --> END([End])
    RETURN --> DISPLAY[Display Similar Content<br/>Grid Layout]
    DISPLAY --> END
    
    style START fill:#667eea,stroke:#764ba2,color:#fff
    style SEARCH_SIM fill:#f093fb,stroke:#4facfe,color:#fff
    style RETURN fill:#30cfd0,stroke:#330867,color:#fff
    style DISPLAY fill:#fa709a,stroke:#fee140,color:#fff
```

## 8. Module Dependency Graph

```mermaid
graph TD
    subgraph "Core Modules"
        APP[app.py<br/>Main Application]
        SEARCH[search_engine.py<br/>Search & Ranking]
        MOOD[mood_taxonomy.py<br/>Mood Definitions]
    end
    
    subgraph "UI Modules"
        UI_COMP[ui_components.py<br/>UI Rendering]
        STYLES[styles.py<br/>CSS Styling]
        STATE[state_manager.py<br/>State Management]
    end
    
    subgraph "Utility Modules"
        UTILS[utils.py<br/>Query Building]
        SEARCH_UTILS[search_utils.py<br/>Engine Init]
        GENRE[genre_mapper.py<br/>Genre Mapping]
        WHY[why_watch_generator.py<br/>Explanations]
        TMDB_DET[tmdb_details.py<br/>TMDb API]
    end
    
    subgraph "Data"
        CSV[CSV Catalog]
        CACHE[Genre Cache]
    end
    
    APP --> UI_COMP
    APP --> STYLES
    APP --> STATE
    APP --> UTILS
    APP --> SEARCH_UTILS
    APP --> MOOD
    
    UI_COMP --> SEARCH
    UI_COMP --> GENRE
    UI_COMP --> WHY
    UI_COMP --> TMDB_DET
    
    SEARCH --> GENRE
    SEARCH --> CSV
    
    SEARCH_UTILS --> SEARCH
    
    UTILS --> MOOD
    
    GENRE --> CACHE
    GENRE --> TMDB_DET
    
    TMDB_DET --> CSV
    
    WHY --> MOOD
    WHY --> GENRE
    
    style APP fill:#667eea,stroke:#764ba2,color:#fff
    style SEARCH fill:#f093fb,stroke:#4facfe,color:#fff
    style UI_COMP fill:#fa709a,stroke:#fee140,color:#fff
    style CSV fill:#30cfd0,stroke:#330867,color:#fff
```

## Usage Instructions

These Mermaid diagrams can be used in:
- **Markdown documentation** (GitHub, GitLab, etc.)
- **Documentation sites** (MkDocs, Sphinx, etc.)
- **Thesis papers** (with appropriate Mermaid renderer)
- **Presentation slides** (exported as images)
- **Online tools** (Mermaid Live Editor: https://mermaid.live)

### Rendering Options:
1. **Online**: Copy diagram code to https://mermaid.live
2. **VS Code**: Install "Markdown Preview Mermaid Support" extension
3. **GitHub/GitLab**: Native Mermaid support in markdown files
4. **Export**: Use Mermaid CLI to export as PNG/SVG

### Diagram Types Included:
1. **High-Level Architecture**: System overview
2. **Query Processing Flow**: Detailed step-by-step flow
3. **Data Flow**: Data movement through system
4. **Sequence Diagram**: Component interactions
5. **Scoring Algorithm**: Scoring calculation flow
6. **Initialization Flow**: System startup process
7. **Similar Content Flow**: Similar item discovery
8. **Module Dependencies**: Code structure

