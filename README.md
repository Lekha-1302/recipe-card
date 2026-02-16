(function() {
    'use strict';

    // ===== Configuration =====
    const FAVORITES_STORAGE_KEY = 'recipeAppFavorites';
    const DEBOUNCE_DELAY = 300;

    // ===== Recipe Data =====
    const recipes = [
        {
            id: 1,
            title: 'Spaghetti Carbonara',
            emoji: '🍝',
            category: 'lunch',
            time: 25,
            ingredients: ['Spaghetti', 'Eggs', 'Bacon', 'Parmesan', 'Black Pepper'],
            instructions: ['Cook spaghetti', 'Fry bacon until crispy', 'Mix eggs with parmesan', 'Combine pasta with bacon', 'Add egg mixture and toss', 'Serve with black pepper']
        },
        {
            id: 2,
            title: 'Chocolate Brownies',
            emoji: '🍫',
            category: 'dessert',
            time: 45,
            ingredients: ['Flour', 'Cocoa Powder', 'Sugar', 'Butter', 'Eggs', 'Vanilla'],
            instructions: ['Preheat oven to 350°F', 'Mix dry ingredients', 'Melt butter and chocolate', 'Combine wet ingredients', 'Mix wet and dry', 'Bake for 30 minutes']
        },
        {
            id: 3,
            title: 'Vegetable Stir-Fry',
            emoji: '🥦',
            category: 'dinner',
            time: 20,
            ingredients: ['Broccoli', 'Bell Peppers', 'Carrots', 'Soy Sauce', 'Garlic', 'Ginger', 'Sesame Oil'],
            instructions: ['Heat wok or pan', 'Add sesame oil', 'Stir-fry aromatics', 'Add vegetables', 'Season with soy sauce', 'Serve hot']
        },
        {
            id: 4,
            title: 'Fluffy Pancakes',
            emoji: '🥞',
            category: 'breakfast',
            time: 15,
            ingredients: ['Flour', 'Eggs', 'Milk', 'Baking Powder', 'Sugar', 'Butter', 'Vanilla'],
            instructions: ['Mix dry ingredients', 'Beat eggs and milk', 'Combine mixtures', 'Heat griddle', 'Pour batter', 'Flip when bubbles form', 'Serve with toppings']
        },
        {
            id: 5,
            title: 'Caesar Salad',
            emoji: '🥗',
            category: 'lunch',
            time: 15,
            ingredients: ['Romaine Lettuce', 'Croutons', 'Parmesan', 'Caesar Dressing', 'Lemon Juice'],
            instructions: ['Wash and tear lettuce', 'Make dressing', 'Toss salad', 'Add croutons', 'Sprinkle parmesan', 'Serve immediately']
        },
        {
            id: 6,
            title: 'Salmon Bake',
            emoji: '🐟',
            category: 'dinner',
            time: 35,
            ingredients: ['Salmon Fillet', 'Lemon', 'Olive Oil', 'Herbs', 'Garlic', 'Salt', 'Pepper'],
            instructions: ['Preheat oven to 400°F', 'Season salmon', 'Add lemon and herbs', 'Bake for 25 minutes', 'Check for doneness', 'Serve hot']
        },
        {
            id: 7,
            title: 'Avocado Toast',
            emoji: '🥑',
            category: 'breakfast',
            time: 10,
            ingredients: ['Whole Grain Bread', 'Avocado', 'Lemon', 'Olive Oil', 'Salt', 'Pepper', 'Red Pepper Flakes'],
            instructions: ['Toast bread', 'Mash avocado', 'Add lemon juice', 'Spread on toast', 'Drizzle olive oil', 'Season to taste']
        },
        {
            id: 8,
            title: 'Tiramisu',
            emoji: '🍰',
            category: 'dessert',
            time: 30,
            ingredients: ['Mascarpone', 'Eggs', 'Sugar', 'Coffee', 'Cocoa Powder', 'Ladyfingers'],
            instructions: ['Brew strong coffee', 'Beat eggs and sugar', 'Fold in mascarpone', 'Dip ladyfingers in coffee', 'Layer with cream', 'Dust with cocoa', 'Chill 4 hours']
        }
    ];

    // ===== State Management =====
    const state = {
        allRecipes: JSON.parse(JSON.stringify(recipes)),
        favorites: loadFavoritesFromStorage(),
        searchQuery: '',
        categoryFilter: '',
        sortOption: 'default',
        showFavoritesOnly: false,
        filteredRecipes: []
    };

    // ===== DOM Elements =====
    const elements = {
        searchInput: document.getElementById('searchInput'),
        clearSearchBtn: document.getElementById('clearSearchBtn'),
        categoryFilter: document.getElementById('categoryFilter'),
        sortSelect: document.getElementById('sortSelect'),
        favoritesFilter: document.getElementById('favoritesFilter'),
        recipesContainer: document.getElementById('recipesContainer'),
        emptyState: document.getElementById('emptyState'),
        recipeCount: document.getElementById('recipeCount'),
        totalRecipes: document.getElementById('totalRecipes')
    };

    // ===== LocalStorage Management =====
    function loadFavoritesFromStorage() {
        try {
            const stored = localStorage.getItem(FAVORITES_STORAGE_KEY);
            return stored ? JSON.parse(stored) : [];
        } catch (error) {
            console.error('Error loading favorites from storage:', error);
            return [];
        }
    }

    function saveFavoritesToStorage(favorites) {
        try {
            localStorage.setItem(FAVORITES_STORAGE_KEY, JSON.stringify(favorites));
        } catch (error) {
            console.error('Error saving favorites to storage:', error);
        }
    }

    function toggleFavorite(recipeId) {
        const index = state.favorites.indexOf(recipeId);
        if (index > -1) {
            state.favorites.splice(index, 1);
        } else {
            state.favorites.push(recipeId);
        }
        saveFavoritesToStorage(state.favorites);
    }

    // ===== Search Functionality with Debouncing =====
    let searchDebounceTimer;

    function debounce(func, delay) {
        return function(...args) {
            clearTimeout(searchDebounceTimer);
            searchDebounceTimer = setTimeout(() => func.apply(this, args), delay);
        };
    }

    function handleSearch(event) {
        state.searchQuery = event.target.value.toLowerCase().trim();
        updateClearButton();
        applyFiltersAndSort();
    }

    function updateClearButton() {
        if (state.searchQuery) {
            elements.clearSearchBtn.classList.add('active');
        } else {
            elements.clearSearchBtn.classList.remove('active');
        }
    }

    function clearSearch() {
        state.searchQuery = '';
        elements.searchInput.value = '';
        updateClearButton();
        applyFiltersAndSort();
    }

    // ===== Filtering Logic =====
    function filterRecipes(recipes) {
        return recipes.filter(recipe => {
            // Search filter
            if (state.searchQuery) {
                const matchesTitle = recipe.title.toLowerCase().includes(state.searchQuery);
                const matchesIngredient = recipe.ingredients.some(ing =>
                    ing.toLowerCase().includes(state.searchQuery)
                );
                if (!matchesTitle && !matchesIngredient) return false;
            }

            // Category filter
            if (state.categoryFilter && recipe.category !== state.categoryFilter) {
                return false;
            }

            // Favorites filter
            if (state.showFavoritesOnly && !state.favorites.includes(recipe.id)) {
                return false;
            }

            return true;
        });
    }

    // ===== Sorting Logic =====
    function sortRecipes(recipes) {
        const sorted = [...recipes];

        switch (state.sortOption) {
            case 'a-z':
                sorted.sort((a, b) => a.title.localeCompare(b.title));
                break;
            case 'z-a':
                sorted.sort((a, b) => b.title.localeCompare(a.title));
                break;
            case 'time-asc':
                sorted.sort((a, b) => a.time - b.time);
                break;
            case 'time-desc':
                sorted.sort((a, b) => b.time - a.time);
                break;
            case 'default':
            default:
                sorted.sort((a, b) => a.id - b.id);
        }

        return sorted;
    }

    // ===== Main Filtering and Sorting =====
    function applyFiltersAndSort() {
        const filtered = filterRecipes(state.allRecipes);
        state.filteredRecipes = sortRecipes(filtered);
        renderRecipes();
        updateRecipeCounter();
    }

    // ===== Rendering =====
    function createRecipeCard(recipe) {
        const isFavorited = state.favorites.includes(recipe.id);
        const card = document.createElement('div');
        card.className = 'recipe-card';
        card.innerHTML = `
            <div class="recipe-header" data-emoji="${recipe.emoji}">
                <button class="favorite-btn ${isFavorited ? 'favorited' : ''}" 
                        data-recipe-id="${recipe.id}" 
                        title="${isFavorited ? 'Remove from favorites' : 'Add to favorites'}"
                        aria-pressed="${isFavorited}">
                    ${isFavorited ? '❤️' : '🤍'}
                </button>
            </div>
            <div class="recipe-info">
                <h3 class="recipe-title">${escapeHtml(recipe.title)}</h3>
                <span class="recipe-category">${escapeHtml(recipe.category)}</span>
                <div class="recipe-time">${recipe.time} minutes</div>
                
                <button class="toggle-btn" data-recipe-id="${recipe.id}">
                    View Details
                </button>

                <div class="expandable-section" data-recipe-id="${recipe.id}">
                    <div>
                        <h4>Ingredients:</h4>
                        <ul class="ingredients-list">
                            ${recipe.ingredients.map(ing => `<li>${escapeHtml(ing)}</li>`).join('')}
                        </ul>
                        <h4>Instructions:</h4>
                        <ol class="instructions-list">
                            ${recipe.instructions.map(inst => `<li>${escapeHtml(inst)}</li>`).join('')}
                        </ol>
                    </div>
                </div>
            </div>
        `;

        // Attach event listeners
        const toggleBtn = card.querySelector('.toggle-btn');
        const favoriteBtn = card.querySelector('.favorite-btn');

        toggleBtn.addEventListener('click', () => toggleExpandableSection(recipe.id));
        favoriteBtn.addEventListener('click', () => handleFavoriteClick(recipe.id, card));

        return card;
    }

    function renderRecipes() {
        elements.recipesContainer.innerHTML = '';

        if (state.filteredRecipes.length === 0) {
            elements.emptyState.style.display = 'block';
            elements.recipesContainer.style.display = 'none';
            return;
        }

        elements.emptyState.style.display = 'none';
        elements.recipesContainer.style.display = 'grid';

        state.filteredRecipes.forEach(recipe => {
            const card = createRecipeCard(recipe);
            elements.recipesContainer.appendChild(card);
        });
    }

    function toggleExpandableSection(recipeId) {
        const section = document.querySelector(`[data-recipe-id="${recipeId}"].expandable-section`);
        const btn = document.querySelector(`[data-recipe-id="${recipeId}"].toggle-btn`);

        if (section && btn) {
            section.classList.toggle('expanded');
            btn.classList.toggle('expanded');
        }
    }

    function handleFavoriteClick(recipeId, cardElement) {
        toggleFavorite(recipeId);

        const favoriteBtn = cardElement.querySelector('.favorite-btn');
        const isFavorited = state.favorites.includes(recipeId);

        favoriteBtn.textContent = isFavorited ? '❤️' : '🤍';
        favoriteBtn.classList.toggle('favorited');
        favoriteBtn.setAttribute('aria-pressed', isFavorited);
        favoriteBtn.title = isFavorited ? 'Remove from favorites' : 'Add to favorites';
    }

    function updateRecipeCounter() {
        const totalCount = state.allRecipes.length;
        const displayCount = state.filteredRecipes.length;

        elements.recipeCount.textContent = displayCount;
        elements.totalRecipes.textContent = totalCount;
    }

    // ===== Utility Functions =====
    function escapeHtml(text) {
        const div = document.createElement('div');
        div.textContent = text;
        return div.innerHTML;
    }

    // ===== Event Listeners Setup =====
    function setupEventListeners() {
        // Search with debouncing
        elements.searchInput.addEventListener('input', debounce(handleSearch, DEBOUNCE_DELAY));

        // Clear search button
        elements.clearSearchBtn.addEventListener('click', clearSearch);

        // Category filter
        elements.categoryFilter.addEventListener('change', (e) => {
            state.categoryFilter = e.target.value;
            applyFiltersAndSort();
        });

        // Sort select
        elements.sortSelect.addEventListener('change', (e) => {
            state.sortOption = e.target.value;
            applyFiltersAndSort();
        });

        // Favorites filter
        elements.favoritesFilter.addEventListener('change', (e) => {
            state.showFavoritesOnly = e.target.checked;
            applyFiltersAndSort();
        });
    }

    // ===== Initialization =====
    function init() {
        setupEventListeners();
        applyFiltersAndSort();
    }

    // Start the app when DOM is ready
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', init);
    } else {
        init();
    }
})();
