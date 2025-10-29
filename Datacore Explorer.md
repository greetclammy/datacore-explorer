---
created: 2025-10-28T16:55+03:00
tags:
cssclasses:
  - max
  - hide-yaml
title: ​return function View() {​
disabled rules:
  - all
no rename: true
---

```datacorejsx
return function View() {
    // Load persisted values from localStorage
    const getPersistedValue = (key, defaultValue) => {
        try {
            const value = localStorage.getItem(key);
            return value !== null ? value : defaultValue;
        } catch (e) {
            return defaultValue;
        }
    };

    // State for sort method, search, view mode, query, and settings
    const [sortMethod, setSortMethod] = dc.useState(getPersistedValue('datacore-masonry-sortMethod', 'mtime-desc'));
    const [searchQuery, setSearchQuery] = dc.useState(getPersistedValue('datacore-masonry-searchQuery', ''));
    const [viewMode, setViewMode] = dc.useState(getPersistedValue('datacore-masonry-viewMode', 'card'));
    const [query, setQuery] = dc.useState(getPersistedValue('datacore-masonry-query', ''));
    const [isShuffled, setIsShuffled] = dc.useState(false);
    const [shuffledOrder, setShuffledOrder] = dc.useState([]);
    const [showQueryEditor, setShowQueryEditor] = dc.useState(false);
    const [showSettings, setShowSettings] = dc.useState(false);
    const [showSortDropdown, setShowSortDropdown] = dc.useState(false);
    const [showSearchBar, setShowSearchBar] = dc.useState(false);
    const [showViewDropdown, setShowViewDropdown] = dc.useState(false);
    const [settings, setSettings] = dc.useState({
        titleProperty: "title",
        descriptionProperty: "description",
        imageProperty: "cover",
        cardBottomDisplay: "tags",
        listMarker: "bullet"
    });

    // Persist state to localStorage whenever it changes
    dc.useEffect(() => {
        try {
            localStorage.setItem('datacore-masonry-query', query);
        } catch (e) {
            console.error('Failed to persist query:', e);
        }
    }, [query]);

    dc.useEffect(() => {
        try {
            localStorage.setItem('datacore-masonry-sortMethod', sortMethod);
        } catch (e) {
            console.error('Failed to persist sortMethod:', e);
        }
    }, [sortMethod]);

    dc.useEffect(() => {
        try {
            localStorage.setItem('datacore-masonry-searchQuery', searchQuery);
        } catch (e) {
            console.error('Failed to persist searchQuery:', e);
        }
    }, [searchQuery]);

    dc.useEffect(() => {
        try {
            localStorage.setItem('datacore-masonry-viewMode', viewMode);
        } catch (e) {
            console.error('Failed to persist viewMode:', e);
        }
    }, [viewMode]);

    // Handle empty query to avoid parsing errors - always call hook with safe fallback
    const pages = dc.useQuery(query && query.trim() ? query : "@page and FALSE");

    // Apply sorting and filtering based on selected method and search
    const sorted = dc.useMemo(() => {
        const pagesArray = Array.isArray(pages) ? [...pages] : [];

        // Filter by search query (case-insensitive)
        let filtered = pagesArray;
        if (searchQuery.trim()) {
            const terms = searchQuery.toLowerCase().trim().split(/\s+/);

            // Separate positive and negative terms
            const positiveTerms = terms.filter(t => !t.startsWith("-"));
            const negativeTerms = terms.filter(t => t.startsWith("-")).map(t => t.slice(1));

            // Split by tags and names
            const posTagTerms = positiveTerms.filter(t => t.startsWith("#"));
            const posNameTerms = positiveTerms.filter(t => !t.startsWith("#"));
            const negTagTerms = negativeTerms.filter(t => t.startsWith("#"));
            const negNameTerms = negativeTerms.filter(t => !t.startsWith("#"));

            filtered = pagesArray.filter(p => {
                const fileName = (p.$name || "").toLowerCase();
                const fileTags = (p.$tags || []).map(t => t.toLowerCase());

                // Check positive matches (case-insensitive)
                const posNameMatch = posNameTerms.every(term => fileName.includes(term));
                const posTagMatch = posTagTerms.every(term =>
                    fileTags.some(fileTag => fileTag === term)
                );

                // Check negative matches (must NOT match, case-insensitive)
                const negNameMatch = negNameTerms.some(term => fileName.includes(term));
                const negTagMatch = negTagTerms.some(term =>
                    fileTags.some(fileTag => fileTag === term)
                );

                return posNameMatch && posTagMatch && !negNameMatch && !negTagMatch;
            });
        }

        // Sort the filtered results
        let sorted;
        if (isShuffled) {
            // Use shuffled order if active
            sorted = filtered.sort((a, b) => {
                const indexA = shuffledOrder.indexOf(a.$path);
                const indexB = shuffledOrder.indexOf(b.$path);
                return indexA - indexB;
            });
        } else {
            // Regular sorting
            switch (sortMethod) {
                case "name-asc":
                    sorted = filtered.sort((a, b) => (a.$name || "").localeCompare(b.$name || ""));
                    break;
                case "name-desc":
                    sorted = filtered.sort((a, b) => (b.$name || "").localeCompare(a.$name || ""));
                    break;
                case "mtime-asc":
                    sorted = filtered.sort((a, b) => (a.$mtime?.toMillis() || 0) - (b.$mtime?.toMillis() || 0));
                    break;
                case "mtime-desc":
                    sorted = filtered.sort((a, b) => (b.$mtime?.toMillis() || 0) - (a.$mtime?.toMillis() || 0));
                    break;
                case "ctime-asc":
                    sorted = filtered.sort((a, b) => (a.$ctime?.toMillis() || 0) - (b.$ctime?.toMillis() || 0));
                    break;
                case "ctime-desc":
                    sorted = filtered.sort((a, b) => (b.$ctime?.toMillis() || 0) - (a.$ctime?.toMillis() || 0));
                    break;
                default:
                    sorted = filtered.sort((a, b) => (b.$mtime?.toMillis() || 0) - (a.$mtime?.toMillis() || 0));
            }
        }
        return sorted;
    }, [pages, sortMethod, searchQuery, isShuffled, shuffledOrder]);

    // Shuffle handler
    const handleShuffle = () => {
        const paths = sorted.map(p => p.$path);
        // Fisher-Yates shuffle algorithm
        const shuffled = [...paths];
        for (let i = shuffled.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
        }
        setShuffledOrder(shuffled);
        setIsShuffled(true);
    };

    // Wrapper for setSortMethod that clears shuffle
    const handleSetSortMethod = (method) => {
        setSortMethod(method);
        setIsShuffled(false);
    };

    // State to store file snippets and images
    const [snippets, setSnippets] = dc.useState({});
    const [images, setImages] = dc.useState({});
    const [staticGifs, setStaticGifs] = dc.useState({}); // Store static first frames of GIFs

    // Close dropdowns when clicking outside
    dc.useEffect(() => {
        const handleClickOutside = (event) => {
            // Close sort dropdown if clicking outside
            if (showSortDropdown) {
                const sortDropdown = event.target.closest('.sort-dropdown-wrapper');
                if (!sortDropdown) {
                    setShowSortDropdown(false);
                }
            }
            // Close view dropdown if clicking outside
            if (showViewDropdown) {
                const viewDropdown = event.target.closest('.view-dropdown-wrapper');
                if (!viewDropdown) {
                    setShowViewDropdown(false);
                }
            }
        };

        if (showSortDropdown || showViewDropdown) {
            // Use setTimeout to avoid immediate closing on the same click that opened
            setTimeout(() => {
                document.addEventListener('click', handleClickOutside);
            }, 0);
            return () => document.removeEventListener('click', handleClickOutside);
        }
    }, [showSortDropdown, showViewDropdown]);

    // State for masonry layout refs
    const [columnHeights, setColumnHeights] = dc.useState([]);
    const [columnCount, setColumnCount] = dc.useState(3);
    const containerRef = dc.useRef(null);
    const updateLayoutRef = dc.useRef(null);

    // Reset card styles when switching away from masonry mode
    dc.useEffect(() => {
        if (viewMode !== 'masonry' && containerRef.current) {
            // Wait for next tick to ensure DOM is updated
            setTimeout(() => {
                if (!containerRef.current) return;
                const cards = containerRef.current.querySelectorAll('.writing-card');
                cards.forEach((card) => {
                    card.style.position = '';
                    card.style.left = '';
                    card.style.top = '';
                    card.style.width = '';
                });
                if (containerRef.current) {
                    containerRef.current.style.height = '';
                }
            }, 0);
        }
    }, [viewMode]);

    // Calculate masonry positions
    dc.useEffect(() => {
        if (viewMode !== 'masonry' || !containerRef.current) return;

        const updateLayout = () => {
            const container = containerRef.current;
            if (!container) return;

            // Use clientWidth which respects the actual visible width
            const containerWidth = container.clientWidth;
            const cardMinWidth = 350;
            const gap = 16;

            // Calculate number of columns based on visible width
            const cols = Math.max(1, Math.floor((containerWidth + gap) / (cardMinWidth + gap)));
            setColumnCount(cols);

            // Calculate actual card width to fill space
            const totalGapWidth = (cols - 1) * gap;
            const cardWidth = (containerWidth - totalGapWidth) / cols;

            // First pass: set widths only (allows content to reflow)
            const cards = container.querySelectorAll('.writing-card');
            if (cards.length === 0) return;

            cards.forEach((card) => {
                card.style.width = `${cardWidth}px`;
            });

            // Force reflow to get accurate heights
            container.offsetHeight;

            // Second pass: calculate positions with accurate heights
            const heights = new Array(cols).fill(0);
            cards.forEach((card, index) => {
                // Find shortest column
                const shortestCol = heights.indexOf(Math.min(...heights));

                // Calculate position
                const left = shortestCol * (cardWidth + gap);
                const top = heights[shortestCol];

                // Apply position
                card.style.position = 'absolute';
                card.style.left = `${left}px`;
                card.style.top = `${top}px`;

                // Update column height with actual card height
                heights[shortestCol] += card.offsetHeight + gap;
            });

            // Set container height
            container.style.height = `${Math.max(...heights)}px`;
            setColumnHeights(heights);
        };

        // Store update function in ref so images can trigger it
        updateLayoutRef.current = updateLayout;

        // Initial layout with multiple attempts to handle async content
        const initialLayout = () => {
            updateLayout();
            // Retry after images might have loaded
            setTimeout(updateLayout, 100);
            setTimeout(updateLayout, 300);
            setTimeout(updateLayout, 600);
        };
        initialLayout();

        // Use ResizeObserver to detect container width changes (including sidebar)
        const resizeObserver = new ResizeObserver(() => {
            updateLayout();
        });
        resizeObserver.observe(containerRef.current);

        // Also listen for window resize as backup
        let resizeTimeout;
        const handleResize = () => {
            clearTimeout(resizeTimeout);
            resizeTimeout = setTimeout(updateLayout, 100);
        };
        window.addEventListener('resize', handleResize);

        return () => {
            resizeObserver.disconnect();
            window.removeEventListener('resize', handleResize);
            clearTimeout(resizeTimeout);
        };
    }, [sorted, viewMode, snippets, images]);

    // Load file contents asynchronously
    dc.useEffect(() => {
        const loadSnippets = async () => {
            const newSnippets = {};
            const newImages = {};
            const newStaticGifs = {};

            for (const p of sorted) {
                try {
                    const file = app.vault.getAbstractFileByPath(p.$path);
                    if (file) {
                        const text = await app.vault.read(file);

                        // Try to get description from property first
                        let description = p.value(settings.descriptionProperty);
                        if (!description) {
                            // Fallback: extract first lines from file content
                            const cleaned = text.replace(/^---[\s\S]*?---/, "").trim();
                            const lines = cleaned.split("\n").filter(l => l.trim() !== "");
                            const snippetLines = lines.slice(1, 4);
                            description = snippetLines.length > 0 ? snippetLines.join(" ") : "";
                        }
                        newSnippets[p.$path] = description;

                        // Try to get image from property first
                        let imagePath = p.value(settings.imageProperty);

                        if (!imagePath) {
                            // Fallback: extract first image embed from file content
                            // Match ![[image.png]] or ![alt](image.png)
                            // Use negative lookahead to match until ]] for wikilinks
                            const wikiImageMatch = text.match(/!\[\[((?:(?!\]\]).)+\.(png|jpe?g|gif|bmp|svg|webp))\]\]/i);
                            const mdImageMatch = text.match(/!\[([^\]]*)\]\(([^\)]+\.(png|jpe?g|gif|bmp|svg|webp))\)/i);

                            if (wikiImageMatch) {
                                imagePath = wikiImageMatch[1];
                            } else if (mdImageMatch) {
                                imagePath = mdImageMatch[2];
                            }
                        }

                        if (imagePath) {
                            // Resolve the image path relative to the note
                            const imageFile = app.metadataCache.getFirstLinkpathDest(imagePath, p.$path);
                            if (imageFile) {
                                const resourcePath = app.vault.getResourcePath(imageFile);
                                newImages[p.$path] = resourcePath;

                                // If it's a GIF, extract first frame
                                if (resourcePath && resourcePath.toLowerCase().includes('.gif')) {
                                    // Extract first frame asynchronously
                                    const img = new Image();
                                    img.crossOrigin = "anonymous";
                                    img.onload = () => {
                                        const canvas = document.createElement('canvas');
                                        canvas.width = img.width;
                                        canvas.height = img.height;
                                        const ctx = canvas.getContext('2d');
                                        ctx.drawImage(img, 0, 0);
                                        const staticFrame = canvas.toDataURL('image/png');
                                        setStaticGifs(prev => ({...prev, [p.$path]: staticFrame}));
                                    };
                                    img.src = resourcePath;
                                }
                            }
                        }
                    } else {
                        newSnippets[p.$path] = "(File not found)";
                    }
                } catch (e) {
                    console.error("Error reading file:", p.$path, e);
                    newSnippets[p.$path] = "(Error reading file)";
                }
            }

            setSnippets(newSnippets);
            setImages(newImages);
        };

        loadSnippets();
    }, [sorted]);

    return (
        <div className="masonry-view-root">
            <div className="controls-wrapper">
                <div className="bottom-controls">
                    <div className="view-controls-wrapper">
                        <div className="view-dropdown-wrapper">
                            <button
                                className="view-dropdown-btn"
                                onClick={() => setShowViewDropdown(!showViewDropdown)}
                            >
                                <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                    {viewMode === "list" ? (
                                        <>
                                            <line x1="8" y1="6" x2="21" y2="6"/>
                                            <line x1="8" y1="12" x2="21" y2="12"/>
                                            <line x1="8" y1="18" x2="21" y2="18"/>
                                            <line x1="3" y1="6" x2="3.01" y2="6"/>
                                            <line x1="3" y1="12" x2="3.01" y2="12"/>
                                            <line x1="3" y1="18" x2="3.01" y2="18"/>
                                        </>
                                    ) : viewMode === "card" ? (
                                        <>
                                            <rect width="7" height="7" x="3" y="3" rx="1"/>
                                            <rect width="7" height="7" x="14" y="3" rx="1"/>
                                            <rect width="7" height="7" x="14" y="14" rx="1"/>
                                            <rect width="7" height="7" x="3" y="14" rx="1"/>
                                        </>
                                    ) : (
                                        <>
                                            <rect width="7" height="9" x="3" y="3" rx="1"/>
                                            <rect width="7" height="5" x="14" y="3" rx="1"/>
                                            <rect width="7" height="9" x="14" y="12" rx="1"/>
                                            <rect width="7" height="5" x="3" y="16" rx="1"/>
                                        </>
                                    )}
                                </svg>
                                <svg className="chevron" xmlns="http://www.w3.org/2000/svg" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                    <path d="m6 9 6 6 6-6"/>
                                </svg>
                            </button>
                            {showViewDropdown && (
                                <div className="view-dropdown-menu">
                                    <div className="view-option" onClick={() => { setViewMode("list"); setShowViewDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <line x1="8" y1="6" x2="21" y2="6"/>
                                            <line x1="8" y1="12" x2="21" y2="12"/>
                                            <line x1="8" y1="18" x2="21" y2="18"/>
                                            <line x1="3" y1="6" x2="3.01" y2="6"/>
                                            <line x1="3" y1="12" x2="3.01" y2="12"/>
                                            <line x1="3" y1="18" x2="3.01" y2="18"/>
                                        </svg>
                                        <span>List</span>
                                    </div>
                                    <div className="view-option" onClick={() => { setViewMode("card"); setShowViewDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <rect width="7" height="7" x="3" y="3" rx="1"/>
                                            <rect width="7" height="7" x="14" y="3" rx="1"/>
                                            <rect width="7" height="7" x="14" y="14" rx="1"/>
                                            <rect width="7" height="7" x="3" y="14" rx="1"/>
                                        </svg>
                                        <span>Card</span>
                                    </div>
                                    <div className="view-option" onClick={() => { setViewMode("masonry"); setShowViewDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <rect width="7" height="9" x="3" y="3" rx="1"/>
                                            <rect width="7" height="5" x="14" y="3" rx="1"/>
                                            <rect width="7" height="9" x="14" y="12" rx="1"/>
                                            <rect width="7" height="5" x="3" y="16" rx="1"/>
                                        </svg>
                                        <span>Masonry</span>
                                    </div>
                                </div>
                            )}
                        </div>
                        <div className="sort-dropdown-wrapper">
                            <button
                                className="sort-dropdown-btn"
                                onClick={() => setShowSortDropdown(!showSortDropdown)}
                                title={
                                    isShuffled ? "Shuffle" :
                                    sortMethod === "name-asc" ? "File name (A to Z)" :
                                    sortMethod === "name-desc" ? "File name (Z to A)" :
                                    sortMethod === "mtime-desc" ? "Modified time (new to old)" :
                                    sortMethod === "mtime-asc" ? "Modified time (old to new)" :
                                    sortMethod === "ctime-desc" ? "Created time (new to old)" :
                                    "Created time (old to new)"
                                }
                            >
                                {isShuffled ? (
                                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                        <path d="m3 8 4-4 4 4"/>
                                        <path d="M7 4v16"/>
                                        <path d="M11 12h4"/>
                                        <path d="M11 16h7"/>
                                        <path d="M11 20h10"/>
                                    </svg>
                                ) : (
                                    <>
                                        {sortMethod === "name-asc" && (
                                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="m3 16 4 4 4-4"/><path d="M7 20V4"/><path d="M20 8h-5"/><path d="M15 10V6.5a2.5 2.5 0 0 1 5 0V10"/><path d="M15 14h5l-5 6h5"/>
                                            </svg>
                                        )}
                                        {sortMethod === "name-desc" && (
                                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="m3 8 4-4 4 4"/><path d="M7 4v16"/><path d="M15 4h5l-5 6h5"/><path d="M15 20v-3.5a2.5 2.5 0 0 1 5 0V20"/><path d="M20 20h-5"/>
                                            </svg>
                                        )}
                                        {sortMethod === "mtime-desc" && (
                                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="M12.338 21.994A10 10 0 1 1 21.925 13.227"/><path d="M12 6v6l2 1"/><path d="m14 18 4-4 4 4"/><path d="M18 14v8"/>
                                            </svg>
                                        )}
                                        {sortMethod === "mtime-asc" && (
                                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="M13.228 21.925A10 10 0 1 1 21.994 12.338"/><path d="M12 6v6l1.562.781"/><path d="m14 18 4 4 4-4"/><path d="M18 22v-8"/>
                                            </svg>
                                        )}
                                        {sortMethod === "ctime-desc" && (
                                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="M8 2v4"/><path d="M16 2v4"/><rect width="18" height="18" x="3" y="4" rx="2"/><path d="M3 10h18"/><path d="M12 14 8 18"/><path d="M12 14 16 18"/>
                                            </svg>
                                        )}
                                        {sortMethod === "ctime-asc" && (
                                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                                <path d="M8 2v4"/><path d="M16 2v4"/><rect width="18" height="18" x="3" y="4" rx="2"/><path d="M3 10h18"/><path d="M12 18 8 14"/><path d="M12 18 16 14"/>
                                            </svg>
                                        )}
                                    </>
                                )}
                                <svg className="chevron" xmlns="http://www.w3.org/2000/svg" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                    <path d="m6 9 6 6 6-6"/>
                                </svg>
                            </button>
                            {showSortDropdown && (
                                <div className="sort-dropdown-menu">
                                    <div className="sort-option" onClick={() => { handleSetSortMethod("name-asc"); setShowSortDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <path d="m3 16 4 4 4-4"/><path d="M7 20V4"/><path d="M20 8h-5"/><path d="M15 10V6.5a2.5 2.5 0 0 1 5 0V10"/><path d="M15 14h5l-5 6h5"/>
                                        </svg>
                                        <span>File name (A to Z)</span>
                                    </div>
                                    <div className="sort-option" onClick={() => { handleSetSortMethod("name-desc"); setShowSortDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <path d="m3 8 4-4 4 4"/><path d="M7 4v16"/><path d="M15 4h5l-5 6h5"/><path d="M15 20v-3.5a2.5 2.5 0 0 1 5 0V20"/><path d="M20 20h-5"/>
                                        </svg>
                                        <span>File name (Z to A)</span>
                                    </div>
                                    <div className="sort-option" onClick={() => { handleSetSortMethod("mtime-desc"); setShowSortDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <path d="M12.338 21.994A10 10 0 1 1 21.925 13.227"/><path d="M12 6v6l2 1"/><path d="m14 18 4-4 4 4"/><path d="M18 14v8"/>
                                        </svg>
                                        <span>Modified time (new to old)</span>
                                    </div>
                                    <div className="sort-option" onClick={() => { handleSetSortMethod("mtime-asc"); setShowSortDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <path d="M13.228 21.925A10 10 0 1 1 21.994 12.338"/><path d="M12 6v6l1.562.781"/><path d="m14 18 4 4 4-4"/><path d="M18 22v-8"/>
                                        </svg>
                                        <span>Modified time (old to new)</span>
                                    </div>
                                    <div className="sort-option" onClick={() => { handleSetSortMethod("ctime-desc"); setShowSortDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <path d="M8 2v4"/><path d="M16 2v4"/><rect width="18" height="18" x="3" y="4" rx="2"/><path d="M3 10h18"/><path d="M12 14 8 18"/><path d="M12 14 16 18"/>
                                        </svg>
                                        <span>Created time (new to old)</span>
                                    </div>
                                    <div className="sort-option" onClick={() => { handleSetSortMethod("ctime-asc"); setShowSortDropdown(false); }}>
                                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            <path d="M8 2v4"/><path d="M16 2v4"/><rect width="18" height="18" x="3" y="4" rx="2"/><path d="M3 10h18"/><path d="M12 18 8 14"/><path d="M12 18 16 14"/>
                                        </svg>
                                        <span>Created time (old to new)</span>
                                    </div>
                                </div>
                            )}
                        </div>
                        <span className="results-count">{sorted.length} results</span>
                    </div>
                    <div className="search-controls">
                        <div className="search-input-container">
                            <input
                                type="text"
                                placeholder="Search in filenames... (use # for tags)"
                                value={searchQuery}
                                onChange={(e) => setSearchQuery(e.target.value)}
                                onFocus={() => {
                                    setShowSettings(false);
                                    setShowQueryEditor(false);
                                }}
                                className="search-input desktop-search"
                            />
                            {searchQuery && (
                                <svg
                                    className="search-input-clear-button"
                                    aria-label="Clear search"
                                    onClick={() => setSearchQuery('')}
                                    xmlns="http://www.w3.org/2000/svg"
                                    viewBox="0 0 16 16"
                                >
                                    <circle cx="8" cy="8" r="7" fill="currentColor"/>
                                    <line x1="5" y1="5" x2="11" y2="11" stroke="white" strokeWidth="1.5" strokeLinecap="round"/>
                                    <line x1="11" y1="5" x2="5" y2="11" stroke="white" strokeWidth="1.5" strokeLinecap="round"/>
                                </svg>
                            )}
                        </div>
                    </div>
                    <div className="meta-controls">
                        <button
                            className="search-toggle-btn mobile-search"
                            onClick={() => {
                                setShowSearchBar(!showSearchBar);
                                if (!showSearchBar) {
                                    setShowSettings(false);
                                    setShowQueryEditor(false);
                                }
                            }}
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                <circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/>
                            </svg>
                        </button>
                        <button
                            className="shuffle-btn"
                            onClick={handleShuffle}
                            title="Shuffle"
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M2 18h1.4c1.3 0 2.5-.6 3.3-1.7l6.1-8.6c.7-1.1 2-1.7 3.3-1.7H22"/>
                                <path d="m18 2 4 4-4 4"/>
                                <path d="M2 6h1.9c1.5 0 2.9.9 3.6 2.2"/>
                                <path d="M22 18h-5.9c-1.3 0-2.6-.7-3.3-1.8l-.5-.8"/>
                                <path d="m18 14 4 4-4 4"/>
                            </svg>
                        </button>
                        <button
                            className="query-toggle-btn"
                            onClick={() => {
                                setShowQueryEditor(!showQueryEditor);
                                if (!showQueryEditor) {
                                    setShowSettings(false);
                                    setShowSearchBar(false);
                                }
                            }}
                            title={showQueryEditor ? "Hide query" : "Edit query"}
                        >
                            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                <polyline points="16 18 22 12 16 6"/><polyline points="8 6 2 12 8 18"/>
                            </svg>
                        </button>
                        <button
                            className="settings-btn"
                            onClick={() => {
                                setShowSettings(!showSettings);
                                if (!showSettings) {
                                    setShowSearchBar(false);
                                    setShowQueryEditor(false);
                                }
                            }}
                            title="Settings"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                strokeWidth="2"
                                strokeLinecap="round"
                                strokeLinejoin="round"
                            >
                                <path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.1a2 2 0 0 1 1 1.72v.51a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.39a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.09a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"/>
                                <circle cx="12" cy="12" r="3"/>
                            </svg>
                        </button>
                    </div>
                </div>
                {showSearchBar && (
                    <div className="search-bar-expanded">
                        <input
                            type="text"
                            placeholder="Search files... (use #tag for tags)"
                            value={searchQuery}
                            onChange={(e) => setSearchQuery(e.target.value)}
                            className="search-input-mobile"
                        />
                    </div>
                )}
                {showQueryEditor && (
                    <div className="query-editor">
                        <textarea
                            value={query}
                            onChange={(e) => {
                                setQuery(e.target.value);
                                e.target.style.height = 'auto';
                                e.target.style.height = e.target.scrollHeight + 'px';
                            }}
                            className="query-input"
                            placeholder="#work&#10;/Projects/Active&#10;status:in-progress"
                            ref={(el) => {
                                if (el) {
                                    el.style.height = 'auto';
                                    el.style.height = el.scrollHeight + 'px';
                                }
                            }}
                        />
                    </div>
                )}
                {showSettings && (
                    <div className="settings-panel">
                        <div className="setting-item setting-item-text">
                            <div className="setting-item-info">
                                <label>Title property</label>
                                <div className="setting-desc">Set property to show as file title. Will fall back to filename if unavailable.</div>
                            </div>
                            <input
                                type="text"
                                value={settings.titleProperty}
                                onChange={(e) => setSettings({...settings, titleProperty: e.target.value})}
                                placeholder="title"
                                className="setting-text-input"
                            />
                        </div>
                        <div className="setting-item setting-item-text">
                            <div className="setting-item-info">
                                <label>Description property</label>
                                <div className="setting-desc">Set property to show in file description. Will fall back to first lines of file if unavailable.</div>
                            </div>
                            <input
                                type="text"
                                value={settings.descriptionProperty}
                                onChange={(e) => setSettings({...settings, descriptionProperty: e.target.value})}
                                placeholder="description"
                                className="setting-text-input"
                            />
                        </div>
                        <div className="setting-item setting-item-text">
                            <div className="setting-item-info">
                                <label>Image property</label>
                                <div className="setting-desc">Set property to show as image. Will fall back to first embed in note if unavailable.</div>
                            </div>
                            <input
                                type="text"
                                value={settings.imageProperty}
                                onChange={(e) => setSettings({...settings, imageProperty: e.target.value})}
                                placeholder="cover"
                                className="setting-text-input"
                            />
                        </div>
                        <div className="setting-item setting-item-text">
                            <div className="setting-item-info">
                                <label>Card bottom display</label>
                                <div className="setting-desc">Set what to show at card bottom.</div>
                            </div>
                            <select
                                value={settings.cardBottomDisplay}
                                onChange={(e) => setSettings({...settings, cardBottomDisplay: e.target.value})}
                                className="setting-select-input"
                            >
                                <option value="tags">File tags</option>
                                <option value="path">File path</option>
                            </select>
                        </div>
                        <div className="setting-item setting-item-text">
                            <div className="setting-item-info">
                                <label>List marker</label>
                                <div className="setting-desc">Set marker style for list view.</div>
                            </div>
                            <select
                                value={settings.listMarker}
                                onChange={(e) => setSettings({...settings, listMarker: e.target.value})}
                                className="setting-select-input"
                            >
                                <option value="bullet">Bullet</option>
                                <option value="number">Number</option>
                                <option value="none">None</option>
                            </select>
                        </div>
                    </div>
                )}
            </div>

            {viewMode === 'list' ? (
                <ul
                    ref={containerRef}
                    className={`list-view marker-${settings.listMarker}`}
                >
                {sorted.map((p, index) => {
                // Get title from property or fallback to filename
                const titleValue = p.value(settings.titleProperty) || p.$name;

                // List view - simple link list
                if (viewMode === 'list') {
                    return (
                        <li key={p.$path} className="list-item">
                            <a
                                href={p.$path}
                                className="internal-link list-link"
                                onClick={(e) => {
                                    if (!e.metaKey && !e.ctrlKey && !e.shiftKey) {
                                        e.preventDefault();
                                        app.workspace.openLinkText(p.$path, "", false);
                                    }
                                }}
                                onMouseEnter={(e) => {
                                    app.workspace.trigger('hover-link', {
                                        event: e.nativeEvent,
                                        source: 'datacore-explorer',
                                        hoverParent: e.currentTarget,
                                        targetEl: e.currentTarget,
                                        linktext: p.$path,
                                        sourcePath: p.$path
                                    });
                                }}
                            >
                                {titleValue}
                            </a>
                        </li>
                    );
                }

                // Card/Masonry view - full cards
                // Determine which timestamp to show based on sort order
                const useCreatedTime = sortMethod.startsWith('ctime');
                const timestamp = useCreatedTime ? p.$ctime : p.$mtime;
                const timestampMillis = timestamp?.toMillis() || 0;

                // Check if timestamp is in last 24 hours
                const now = Date.now();
                const isRecent = now - timestampMillis < 86400000;
                const date = timestamp ? (isRecent ? timestamp.toFormat("yyyy-MM-dd HH:mm") : timestamp.toFormat("yyyy-MM-dd")) : "";
                const timeIcon = useCreatedTime ? "calendar" : "clock";

                const snippet = p.$path in snippets ? snippets[p.$path] : "Loading...";
                const tags = p.$tags || [];
                const imageSrc = images[p.$path];

                // Get parent folder path (without filename)
                const folderPath = p.$path.split('/').slice(0, -1).join('/');

                // Check if image is GIF
                const isGif = imageSrc && imageSrc.toLowerCase().includes('.gif');
                const staticFrame = isGif ? staticGifs[p.$path] : null;

                return (
                    <div
                        key={p.$path}
                        className="writing-card"
                        onClick={(e) => {
                            // Only open if clicking on the card itself, not on interactive elements or images
                            if (e.target.tagName !== 'A' && !e.target.closest('a') && e.target.tagName !== 'IMG') {
                                app.workspace.openLinkText(p.$path, "", false);
                            }
                        }}
                        onMouseEnter={(e) => {
                            // Trigger Obsidian's hover preview
                            app.workspace.trigger('hover-link', {
                                event: e.nativeEvent,
                                source: 'datacore-explorer',
                                hoverParent: e.currentTarget,
                                targetEl: e.currentTarget,
                                linktext: p.$path,
                                sourcePath: p.$path
                            });
                        }}
                        style={{ cursor: 'pointer' }}
                    >
                        <div className="writing-title">
                            <span>{titleValue}</span>
                        </div>
                        <div className="snippet-container">
                            <div className="writing-snippet">{snippet}</div>
                            {imageSrc && (
                                <div className="card-thumbnail">
                                    <img
                                        src={isGif ? (staticFrame || imageSrc) : imageSrc}
                                        data-gif-src={isGif ? imageSrc : ''}
                                        data-static-src={staticFrame || ''}
                                        alt=""
                                        onLoad={() => {
                                            // Trigger layout recalculation when image loads
                                            if (updateLayoutRef.current) {
                                                updateLayoutRef.current();
                                            }
                                        }}
                                        onMouseEnter={(e) => {
                                            const gifSrc = e.target.getAttribute('data-gif-src');
                                            if (gifSrc) {
                                                e.target.src = gifSrc;
                                            }
                                        }}
                                        onMouseLeave={(e) => {
                                            const staticSrc = e.target.getAttribute('data-static-src');
                                            if (staticSrc) {
                                                e.target.src = staticSrc;
                                            }
                                        }}
                                    />
                                </div>
                            )}
                        </div>
                        <div className="writing-meta">
                            <span className="meta-left">
                                {date && (
                                    <>
                                        <svg className="timestamp-icon" xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            {timeIcon === "calendar" ? (
                                                <>
                                                    <path d="M8 2v4"/>
                                                    <path d="M16 2v4"/>
                                                    <rect width="18" height="18" x="3" y="4" rx="2"/>
                                                    <path d="M3 10h18"/>
                                                </>
                                            ) : (
                                                <>
                                                    <circle cx="12" cy="12" r="10"/>
                                                    <polyline points="12 6 12 12 16 14"/>
                                                </>
                                            )}
                                        </svg>
                                        {date}
                                    </>
                                )}
                            </span>
                            <div className="meta-right">
                                {settings.cardBottomDisplay === "tags" ? (
                                    <div className="tags-wrapper">
                                        {tags.map(tag => (
                                            <a
                                                key={tag}
                                                href="#"
                                                className="tag"
                                                onClick={(e) => {
                                                    e.preventDefault();
                                                    const searchPlugin = app.internalPlugins.plugins["global-search"];
                                                    if (searchPlugin && searchPlugin.instance) {
                                                        const searchView = searchPlugin.instance;
                                                        searchView.openGlobalSearch("tag:" + tag);
                                                    }
                                                }}
                                            >
                                                {tag.replace(/^#/, '')}
                                            </a>
                                        ))}
                                    </div>
                                ) : (
                                    <div className="path-wrapper">
                                        {folderPath.split('/').map((folder, index, array) => {
                                            const cumulativePath = array.slice(0, index + 1).join('/');
                                            return (
                                                <span key={index}>
                                                    <span
                                                        className="file-path-segment"
                                                        onClick={() => {
                                                            const fileExplorer = app.internalPlugins?.plugins?.["file-explorer"];
                                                            if (fileExplorer && fileExplorer.instance) {
                                                                const folder = app.vault.getAbstractFileByPath(cumulativePath);
                                                                if (folder) {
                                                                    fileExplorer.instance.revealInFolder(folder);
                                                                }
                                                            }
                                                        }}
                                                    >
                                                        {folder}
                                                    </span>
                                                    {index < array.length - 1 && <span className="path-separator">/</span>}
                                                </span>
                                            );
                                        })}
                                    </div>
                                )}
                            </div>
                        </div>
                    </div>
                );
                })}
                </ul>
            ) : (
                <div
                    ref={containerRef}
                    className={viewMode === "masonry" ? "cards-masonry" : "cards-feed"}
                >
                {sorted.map((p, index) => {
                // Get title from property or fallback to filename
                const titleValue = p.value(settings.titleProperty) || p.$name;

                // Determine which timestamp to show based on sort order
                const useCreatedTime = sortMethod.startsWith('ctime');
                const timestamp = useCreatedTime ? p.$ctime : p.$mtime;
                const timestampMillis = timestamp?.toMillis() || 0;

                // Check if timestamp is in last 24 hours
                const now = Date.now();
                const isRecent = now - timestampMillis < 86400000;
                const date = timestamp ? (isRecent ? timestamp.toFormat("yyyy-MM-dd HH:mm") : timestamp.toFormat("yyyy-MM-dd")) : "";
                const timeIcon = useCreatedTime ? "calendar" : "clock";

                const snippet = p.$path in snippets ? snippets[p.$path] : "Loading...";
                const tags = p.$tags || [];
                const imageSrc = images[p.$path];

                // Get parent folder path (without filename)
                const folderPath = p.$path.split('/').slice(0, -1).join('/');

                // Check if image is GIF
                const isGif = imageSrc && imageSrc.toLowerCase().includes('.gif');
                const staticFrame = isGif ? staticGifs[p.$path] : null;

                return (
                    <div
                        key={p.$path}
                        className="writing-card"
                        onClick={(e) => {
                            // Only open if clicking on the card itself, not on interactive elements or images
                            if (e.target.tagName !== 'A' && !e.target.closest('a') && e.target.tagName !== 'IMG') {
                                app.workspace.openLinkText(p.$path, "", false);
                            }
                        }}
                        onMouseEnter={(e) => {
                            // Trigger Obsidian's hover preview
                            app.workspace.trigger('hover-link', {
                                event: e.nativeEvent,
                                source: 'datacore-explorer',
                                hoverParent: e.currentTarget,
                                targetEl: e.currentTarget,
                                linktext: p.$path,
                                sourcePath: p.$path
                            });
                        }}
                        style={{ cursor: 'pointer' }}
                    >
                        <div className="writing-title">
                            <span>{titleValue}</span>
                        </div>
                        <div className="snippet-container">
                            <div className="writing-snippet">{snippet}</div>
                            {imageSrc && (
                                <div className="card-thumbnail">
                                    <img
                                        src={isGif ? (staticFrame || imageSrc) : imageSrc}
                                        data-gif-src={isGif ? imageSrc : ''}
                                        data-static-src={staticFrame || ''}
                                        alt=""
                                        onLoad={() => {
                                            // Trigger layout recalculation when image loads
                                            if (updateLayoutRef.current) {
                                                updateLayoutRef.current();
                                            }
                                        }}
                                        onMouseEnter={(e) => {
                                            const gifSrc = e.target.getAttribute('data-gif-src');
                                            if (gifSrc) {
                                                e.target.src = gifSrc;
                                            }
                                        }}
                                        onMouseLeave={(e) => {
                                            const staticSrc = e.target.getAttribute('data-static-src');
                                            if (staticSrc) {
                                                e.target.src = staticSrc;
                                            }
                                        }}
                                    />
                                </div>
                            )}
                        </div>
                        <div className="writing-meta">
                            <span className="meta-left">
                                {date && (
                                    <>
                                        <svg className="timestamp-icon" xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                            {timeIcon === "calendar" ? (
                                                <>
                                                    <path d="M8 2v4"/>
                                                    <path d="M16 2v4"/>
                                                    <rect width="18" height="18" x="3" y="4" rx="2"/>
                                                    <path d="M3 10h18"/>
                                                </>
                                            ) : (
                                                <>
                                                    <circle cx="12" cy="12" r="10"/>
                                                    <polyline points="12 6 12 12 16 14"/>
                                                </>
                                            )}
                                        </svg>
                                        {date}
                                    </>
                                )}
                            </span>
                            <div className="meta-right">
                                {settings.cardBottomDisplay === "tags" ? (
                                    <div className="tags-wrapper">
                                        {tags.map(tag => (
                                            <a
                                                key={tag}
                                                href="#"
                                                className="tag"
                                                onClick={(e) => {
                                                    e.preventDefault();
                                                    const searchPlugin = app.internalPlugins.plugins["global-search"];
                                                    if (searchPlugin && searchPlugin.instance) {
                                                        const searchView = searchPlugin.instance;
                                                        searchView.openGlobalSearch("tag:" + tag);
                                                    }
                                                }}
                                            >
                                                {tag.replace(/^#/, '')}
                                            </a>
                                        ))}
                                    </div>
                                ) : (
                                    <div className="path-wrapper">
                                        {folderPath.split('/').map((folder, index, array) => {
                                            const cumulativePath = array.slice(0, index + 1).join('/');
                                            return folder ? (
                                                <span key={cumulativePath} style={{ display: 'inline-flex', alignItems: 'center' }}>
                                                    <span
                                                        className="path-segment"
                                                        onClick={(e) => {
                                                            e.stopPropagation();
                                                            const searchPlugin = app.internalPlugins.plugins["global-search"];
                                                            if (searchPlugin && searchPlugin.instance) {
                                                                const searchView = searchPlugin.instance;
                                                                searchView.openGlobalSearch(`path:"${cumulativePath}"`);
                                                            }
                                                        }}
                                                    >
                                                        {folder}
                                                    </span>
                                                    {index < array.length - 1 && <span className="path-separator">/</span>}
                                                </span>
                                            ) : null;
                                        })}
                                    </div>
                                )}
                            </div>
                        </div>
                    </div>
                );
                })}
                </div>
            )}

            <style>{`
                .masonry-view-root {
                    width: 100%;
                    max-width: 100%;
                    overflow-x: hidden;
                    box-sizing: border-box;
                }
                .controls-wrapper {
                    display: flex;
                    flex-direction: column;
                    gap: 8px;
                    margin-bottom: 16px;
                    max-width: 100%;
                }
                .bottom-controls {
                    display: flex;
                    justify-content: space-between;
                    align-items: center;
                    gap: 12px;
                }
                .view-controls-wrapper {
                    display: flex;
                    align-items: center;
                    gap: 12px;
                }
                .view-dropdown-wrapper {
                    position: relative;
                }
                .view-dropdown-btn {
                    padding: 6px 12px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    cursor: pointer;
                    display: flex;
                    align-items: center;
                    gap: 8px;
                    transition: all 0.2s;
                    font-size: 0.9em;
                    color: var(--text-normal);
                }
                .view-dropdown-btn .chevron {
                    opacity: 0.6;
                }
                .view-dropdown-btn:hover {
                    background-color: var(--background-modifier-hover);
                    border-color: var(--text-muted);
                }
                .view-dropdown-menu {
                    position: absolute;
                    top: calc(100% + 4px);
                    left: 0;
                    background-color: var(--background-primary);
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    box-shadow: 0 4px 12px var(--background-modifier-box-shadow);
                    z-index: 1000;
                    min-width: 120px;
                    overflow: hidden;
                }
                .view-option {
                    padding: 8px 12px;
                    cursor: pointer;
                    transition: background-color 0.2s;
                    font-size: 0.9em;
                    display: flex;
                    align-items: center;
                    gap: 8px;
                }
                .view-option:hover {
                    background-color: var(--background-modifier-hover);
                }
                .sort-dropdown-wrapper {
                    position: relative;
                }
                .sort-dropdown-btn {
                    padding: 6px 8px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    cursor: pointer;
                    display: flex;
                    align-items: center;
                    gap: 4px;
                    transition: all 0.2s;
                    color: var(--text-normal);
                }
                .sort-dropdown-btn svg {
                    flex-shrink: 0;
                    width: 16px;
                    height: 16px;
                }
                .sort-dropdown-btn .chevron {
                    width: 12px;
                    height: 12px;
                    opacity: 0.6;
                }
                .sort-dropdown-btn:hover {
                    background-color: var(--background-modifier-hover);
                    border-color: var(--text-muted);
                }
                .sort-dropdown-menu {
                    position: absolute;
                    top: calc(100% + 4px);
                    left: 0;
                    background-color: var(--background-primary);
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    box-shadow: 0 4px 12px var(--background-modifier-box-shadow);
                    z-index: 1000;
                    min-width: 240px;
                    overflow: hidden;
                }
                .sort-option {
                    display: flex;
                    align-items: center;
                    gap: 8px;
                    padding: 8px 12px;
                    cursor: pointer;
                    transition: background-color 0.2s;
                    font-size: 0.9em;
                }
                .sort-option:hover {
                    background-color: var(--background-modifier-hover);
                }
                .sort-option svg {
                    flex-shrink: 0;
                }
                .results-count {
                    font-size: 0.85em;
                    color: var(--text-muted);
                    white-space: nowrap;
                }
                .meta-controls {
                    display: flex;
                    gap: 8px;
                    align-items: center;
                }
                .settings-btn {
                    padding: 6px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    cursor: pointer;
                    display: flex;
                    align-items: center;
                    justify-content: center;
                    transition: all 0.2s;
                }
                .settings-btn svg {
                    width: 16px;
                    height: 16px;
                    color: var(--text-normal);
                }
                .settings-btn:hover {
                    background-color: var(--background-modifier-hover);
                    border-color: var(--text-muted);
                }
                .query-toggle-btn {
                    padding: 6px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    cursor: pointer;
                    display: flex;
                    align-items: center;
                    justify-content: center;
                    transition: all 0.2s;
                }
                .query-toggle-btn svg {
                    width: 16px;
                    height: 16px;
                    color: var(--text-normal);
                }
                .query-toggle-btn:hover {
                    border-color: var(--text-muted);
                    background-color: var(--background-modifier-hover);
                }
                .shuffle-btn {
                    padding: 6px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    cursor: pointer;
                    display: flex;
                    align-items: center;
                    justify-content: center;
                    transition: all 0.2s;
                }
                .shuffle-btn svg {
                    width: 16px;
                    height: 16px;
                    color: var(--text-normal);
                }
                .shuffle-btn:hover {
                    border-color: var(--text-muted);
                    background-color: var(--background-modifier-hover);
                }
                .masonry-view-root .settings-panel {
                    display: flex !important;
                    flex-direction: column !important;
                    gap: 12px;
                    padding: 16px;
                    background-color: var(--background-primary-alt);
                    border-radius: 8px;
                    border: 1px solid var(--background-modifier-border);
                    text-align: left !important;
                    align-items: flex-start !important;
                    max-width: 100%;
                    box-sizing: border-box;
                }
                .masonry-view-root .setting-item {
                    display: flex !important;
                    flex-direction: column !important;
                    gap: 4px;
                    align-items: flex-start !important;
                    width: 100%;
                    max-width: 100%;
                    text-align: left !important;
                    box-sizing: border-box;
                }
                .masonry-view-root .setting-item-toggle {
                    flex-direction: row !important;
                    justify-content: space-between !important;
                    align-items: center !important;
                }
                .masonry-view-root .setting-item-info {
                    display: flex;
                    flex-direction: column;
                    gap: 4px;
                    text-align: left !important;
                }
                .masonry-view-root .setting-item label {
                    font-size: 0.9em;
                    color: var(--text-normal);
                    font-weight: 500;
                    text-align: left !important;
                }
                .masonry-view-root .setting-item input[type="text"] {
                    padding: 6px 10px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 4px;
                    background-color: var(--background-primary);
                    color: var(--text-normal);
                    font-size: 0.9em;
                    box-sizing: border-box;
                }
                .masonry-view-root .setting-item input[type="text"]:focus {
                    border-color: var(--interactive-accent);
                }
                .masonry-view-root .setting-item select {
                    padding: 6px 10px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 4px;
                    background-color: var(--background-primary);
                    color: var(--text-normal);
                    cursor: pointer;
                    font-size: 0.9em;
                    box-sizing: border-box;
                }
                .masonry-view-root .setting-item select:focus {
                    border-color: var(--interactive-accent);
                }
                .masonry-view-root .setting-desc {
                    font-size: 0.8em;
                    color: var(--text-faint);
                    text-align: left !important;
                }
                .masonry-view-root .setting-item-text {
                    flex-direction: row !important;
                    gap: 16px;
                    align-items: flex-start !important;
                }
                .masonry-view-root .setting-item-text .setting-item-info {
                    flex: 1;
                    min-width: 150px;
                }
                .masonry-view-root .setting-text-input,
                .masonry-view-root .setting-select-input {
                    width: 180px;
                    max-width: 180px;
                    flex-shrink: 1;
                    padding: 6px 10px;
                    border: 1px solid var(--background-modifier-border);
                    box-sizing: border-box;
                    border-radius: 4px;
                    background-color: var(--background-primary);
                    color: var(--text-normal);
                    font-size: 0.9em;
                }
                .masonry-view-root .setting-text-input:focus,
                .masonry-view-root .setting-select-input:focus {
                    border-color: var(--interactive-accent);
                }
                .masonry-view-root .setting-select-input {
                    cursor: pointer;
                    min-height: 32px;
                }
                @media (max-width: 900px) {
                    .masonry-view-root .setting-item-text {
                        flex-direction: column !important;
                        gap: 4px;
                    }
                    .masonry-view-root .setting-item-text .setting-item-info {
                        min-width: 0;
                    }
                    .masonry-view-root .setting-text-input,
                    .masonry-view-root .setting-select-input {
                        width: 100%;
                    }
                }
                .toggle-switch {
                    position: relative;
                    display: inline-block;
                    width: 44px;
                    height: 24px;
                    flex-shrink: 0;
                }
                .toggle-switch input {
                    opacity: 0;
                    width: 0;
                    height: 0;
                }
                .toggle-slider {
                    position: absolute;
                    cursor: pointer;
                    top: 0;
                    left: 0;
                    right: 0;
                    bottom: 0;
                    background-color: var(--background-modifier-border);
                    transition: 0.3s;
                    border-radius: 24px;
                }
                .toggle-slider:before {
                    position: absolute;
                    content: "";
                    height: 18px;
                    width: 18px;
                    left: 3px;
                    bottom: 3px;
                    background-color: var(--background-primary);
                    transition: 0.3s;
                    border-radius: 50%;
                }
                label.toggle-switch input[type="checkbox"]:checked + span.toggle-slider {
                    background-color: var(--interactive-accent) !important;
                }
                .toggle-switch input:checked + .toggle-slider:before {
                    transform: translateX(20px);
                }
                .toggle-switch:hover .toggle-slider {
                    opacity: 0.9;
                }
                .query-editor {
                    display: flex;
                    flex-direction: column;
                    gap: 8px;
                    padding: 12px;
                    background-color: var(--background-primary-alt);
                    border-radius: 8px;
                    border: 1px solid var(--background-modifier-border);
                }
                .query-input {
                    width: 100%;
                    padding: 6px 10px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 4px;
                    background-color: var(--background-primary);
                    color: var(--text-normal);
                    font-size: 0.9em;
                    font-family: var(--font-monospace);
                    resize: none;
                    overflow: hidden;
                    min-height: 32px;
                }
                .query-input:focus {
                    border-color: var(--interactive-accent);
                }
                .cards-feed {
                    display: flex;
                    flex-direction: column;
                    max-width: 100%;
                    overflow-x: hidden;
                }
                .cards-masonry {
                    position: relative;
                    width: 100%;
                    max-width: 100%;
                    overflow-x: hidden;
                }
                .cards-masonry .writing-card {
                    transition: all 0.3s ease;
                    max-width: 100%;
                }
                .list-view {
                    margin: 0;
                    padding-left: 0;
                }
                .list-view.marker-bullet {
                    list-style-type: disc !important;
                    list-style-position: outside;
                    padding-left: 20px;
                }
                .list-view.marker-bullet .list-item {
                    display: list-item !important;
                }
                .list-view.marker-number {
                    list-style-type: decimal !important;
                    list-style-position: outside;
                    padding-left: 20px;
                }
                .list-view.marker-number .list-item {
                    display: list-item !important;
                }
                .list-view.marker-none {
                    list-style-type: none !important;
                    padding-left: 0 !important;
                }
                .list-view.marker-none .list-item {
                    display: list-item !important;
                }
                .list-item {
                    margin-bottom: 4px;
                    line-height: 1.6;
                }
                .list-link {
                    color: var(--link-color);
                    text-decoration: none;
                }
                .list-link:hover {
                    color: var(--link-color-hover);
                }
                .search-controls {
                    flex: 0 0 auto;
                    width: 300px;
                    margin-right: auto;
                }
                .search-input-container {
                    position: relative;
                    width: 100%;
                }
                .desktop-search {
                    width: 100%;
                    padding: 6px 28px 6px 10px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    color: var(--text-normal);
                    font-size: 0.9em;
                    transition: all 0.2s;
                }
                .search-input-clear-button {
                    position: absolute;
                    right: 6px;
                    top: 6px;
                    bottom: 6px;
                    margin: auto 0;
                    width: 15px;
                    height: 15px;
                    cursor: pointer;
                    color: var(--text-muted);
                    opacity: 0.7;
                    transition: opacity 0.2s, color 0.2s;
                    display: block;
                }
                .search-input-clear-button:hover {
                    opacity: 1;
                    color: var(--text-normal);
                }
                .desktop-search:hover {
                    border-color: var(--text-muted);
                }
                .desktop-search:focus {
                    border-color: var(--interactive-accent);
                }
                .desktop-search::placeholder {
                    color: var(--text-faint);
                }
                .mobile-search {
                    display: none !important;
                    padding: 6px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary-alt);
                    cursor: pointer;
                    align-items: center;
                    justify-content: center;
                    transition: all 0.2s;
                }
                .mobile-search:hover {
                    background-color: var(--background-modifier-hover);
                    border-color: var(--text-muted);
                }
                .mobile-search svg {
                    width: 16px;
                    height: 16px;
                }
                .search-bar-expanded {
                    padding: 8px 12px;
                    background-color: var(--background-primary-alt);
                    border-radius: 8px;
                    border: 1px solid var(--background-modifier-border);
                }
                .search-input-mobile {
                    width: 100%;
                    padding: 6px 10px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 6px;
                    background-color: var(--background-primary);
                    color: var(--text-normal);
                    font-size: 0.9em;
                }
                .search-input-mobile:focus {
                    border-color: var(--interactive-accent);
                }
                @media (max-width: 700px) {
                    .search-controls {
                        display: none !important;
                    }
                    .mobile-search {
                        display: flex !important;
                    }
                }
                .writing-card {
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 12px;
                    padding: 12px 16px;
                    margin: 8px 0;
                    background-color: var(--background-primary-alt);
                    box-shadow: 0 1px 4px var(--background-modifier-box-shadow);
                    break-inside: avoid;
                    page-break-inside: avoid;
                }
                .writing-title {
                    font-weight: 600;
                    margin-bottom: 4px;
                    color: var(--text-normal);
                    max-width: 100%;
                    word-break: break-word;
                }
                .writing-title > span {
                    display: -webkit-box;
                    -webkit-line-clamp: 2;
                    -webkit-box-orient: vertical;
                    overflow: hidden;
                }
                .snippet-container {
                    display: flex;
                    gap: 12px;
                    margin-bottom: 8px;
                }
                .writing-snippet {
                    color: var(--text-muted);
                    font-size: 0.9em;
                    flex: 1;
                    min-width: 0;
                    display: -webkit-box;
                    -webkit-line-clamp: 5;
                    -webkit-box-orient: vertical;
                    overflow: hidden;
                    line-height: 1.4;
                }
                .card-thumbnail {
                    flex-shrink: 0;
                    width: 80px;
                    height: 80px;
                    overflow: hidden;
                    border-radius: 6px;
                    pointer-events: none;
                }
                .card-thumbnail img {
                    width: 100%;
                    height: 100%;
                    object-fit: cover;
                    pointer-events: auto;
                }
                .writing-meta {
                    display: flex;
                    align-items: center;
                    font-size: 0.8em;
                    color: var(--text-faint);
                }
                .meta-left {
                    flex-shrink: 0;
                    white-space: nowrap;
                    display: flex;
                    align-items: center;
                    gap: 4px;
                    line-height: 1;
                }
                .timestamp-icon {
                    flex-shrink: 0;
                    opacity: 0.7;
                    display: block;
                    margin-bottom: 1px;
                }
                .meta-right {
                    overflow-x: auto;
                    flex: 1;
                    min-width: 0;
                    margin-left: 6px;
                    scrollbar-width: none;
                    text-align: right;
                }
                .meta-right::-webkit-scrollbar {
                    display: none;
                }
                .tags-wrapper {
                    display: flex;
                    gap: 4px;
                    white-space: nowrap;
                    flex-shrink: 0;
                    justify-content: flex-end;
                    min-width: min-content;
                }
                .path-wrapper {
                    display: flex;
                    white-space: nowrap;
                    flex-shrink: 0;
                    justify-content: flex-end;
                    min-width: min-content;
                }
                .writing-meta .tag {
                    background-color: var(--tag-background);
                    color: var(--tag-text);
                    border-radius: 6px;
                    padding: 2px 6px;
                    text-decoration: none;
                    white-space: nowrap;
                    flex-shrink: 0;
                }
                .writing-meta .tag:hover {
                    opacity: 0.8;
                }
                .file-path-segment {
                    color: var(--text-faint);
                    white-space: nowrap;
                    cursor: pointer;
                }
                .file-path-segment:hover {
                    color: var(--text-muted);
                    text-decoration: underline;
                }
                .path-separator {
                    color: var(--text-faint);
                    margin: 0 2px;
                    user-select: none;
                }
            `}</style>
        </div>
    );
}
```
