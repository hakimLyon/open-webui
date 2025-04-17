<script lang="ts">
	import Fuse from 'fuse.js';

	import { createEventDispatcher, onMount } from 'svelte';
	import { tick, getContext } from 'svelte';

	import { customTags } from '$lib/stores';

	const i18n = getContext('i18n');

	const dispatch = createEventDispatcher();

	export let command = '';

	interface TagItem {
		name: string;
		color?: string;
		category?: string;
		categoryColor?: string;
		categoryIcon?: string;
	}

	interface CategoryItem {
		name: string;
		icon: string;
		color: string;
		subOptions?: TagItem[];
		isCategory?: boolean;
	}

	let selectedIdx = 0;
	let selectedCategory = '';
	let showSubOptions = false;
	let filteredItems: (CategoryItem | TagItem)[] = [];
	let filteredSubItems: TagItem[] = [];
	
	// Données des tags
	const tagData: (CategoryItem | TagItem)[] = [
		{
			name: "Priorité",
			icon: "🚩",
			color: "#ff6b6b",
			isCategory: true,
			subOptions: [
				{ name: "critique", color: "#ff0000" },
				{ name: "élevée", color: "#ff6b6b" },
				{ name: "moyenne", color: "#ffa500" },
				{ name: "faible", color: "#4ecdc4" }
			]
		},
		{
			name: "Statut",
			icon: "📊",
			color: "#4ecdc4",
			isCategory: true,
			subOptions: [
				{ name: "à-faire", color: "#ff9f43" },
				{ name: "en-cours", color: "#a29bfe" },
				{ name: "révision", color: "#74b9ff" },
				{ name: "terminé", color: "#55efc4" },
				{ name: "bloqué", color: "#e17055" }
			]
		},
		{
			name: "Type",
			icon: "📋",
			color: "#a29bfe",
			isCategory: true,
			subOptions: [
				{ name: "fonctionnalité", color: "#81ecec" },
				{ name: "bogue", color: "#ff7675" },
				{ name: "amélioration", color: "#55efc4" },
				{ name: "tâche", color: "#ffeaa7" }
			]
		},
		{
			name: "Équipe",
			icon: "👥",
			color: "#74b9ff",
			isCategory: true,
			subOptions: [
				{ name: "frontend", color: "#6c5ce7" },
				{ name: "backend", color: "#0984e3" },
				{ name: "design", color: "#e84393" },
				{ name: "qa", color: "#00b894" }
			]
		},
		
		{ name: "important", color: "#ff0000" },
		{ name: "suivi", color: "#ff9500" },
		{ name: "question", color: "#a855f7" },
		{ name: "idée", color: "#3b82f6" }
	];
	
	// Récupérer toutes les catégories
	const categories = tagData.filter(item => 'isCategory' in item && item.isCategory) as CategoryItem[];
	
	// Récupérer tous les tags individuels
	const individualTags = tagData.filter(item => !('isCategory' in item && item.isCategory)) as TagItem[];
	
	// Flatten tags pour la recherche
	const flattenedTags: TagItem[] = tagData.reduce((acc: TagItem[], item) => {
		if ('isCategory' in item && item.isCategory && item.subOptions) {
			item.subOptions.forEach(tag => {
				acc.push({
					...tag, 
					category: item.name,
					categoryColor: item.color,
					categoryIcon: item.icon
				});
			});
		} else if (!('isCategory' in item) || !item.isCategory) {
			acc.push(item as TagItem);
		}
		return acc;
	}, []);

	// Créer un index de recherche pour tous les tags (individuels et imbriqués)
	let fuse = new Fuse(
		$customTags.length ? $customTags : flattenedTags,
		{
			keys: ['name', 'category'],
			threshold: 0.3
		}
	);
	
	// Créer un index de recherche pour les catégories
	let categoryFuse = new Fuse(
		categories,
		{
			keys: ['name'],
			threshold: 0.3
		}
	);

	$: if (showSubOptions) {
		const category = categories.find(cat => cat.name === selectedCategory);
		filteredSubItems = category?.subOptions || [];
	} else {
		if (command.slice(1)) {
			// Recherche globale combinant tags et catégories
			const searchTerm = command.slice(1);
			
			// Recherche des tags (individuels ou dans des catégories)
			const tagResults = fuse.search(searchTerm).map(result => result.item as TagItem);
			
			// Recherche des catégories
			const categoryResults = categoryFuse.search(searchTerm).map(result => result.item as CategoryItem);
			
			// Combinaison des résultats (catégories d'abord, puis tags)
			filteredItems = [...categoryResults, ...tagResults];
		} else {
			// Vue par défaut - catégories en premier, puis tags individuels
			filteredItems = $customTags.length ? $customTags : [...categories, ...individualTags];
		}
	}

	$: if (command && !showSubOptions) {
		selectedIdx = 0;
	}

	export const selectUp = () => {
		if (showSubOptions) {
			selectedIdx = Math.max(0, selectedIdx - 1);
		} else {
			selectedIdx = Math.max(0, selectedIdx - 1);
		}
	};

	export const selectDown = () => {
		if (showSubOptions) {
			selectedIdx = Math.min(selectedIdx + 1, filteredSubItems.length - 1);
		} else {
			selectedIdx = Math.min(selectedIdx + 1, filteredItems.length - 1);
		}
	};

	const confirmSelect = async (tag: TagItem) => {
		command = '';
		dispatch('select', tag);
	};

	const selectCategory = (category: string) => {
		selectedCategory = category;
		selectedIdx = 0;
		showSubOptions = true;
	};

	const goBack = () => {
		showSubOptions = false;
		selectedIdx = 0;
	};

	const isCategory = (item: CategoryItem | TagItem): item is CategoryItem => {
		return 'isCategory' in item && item.isCategory === true;
	};

	onMount(async () => {
		await tick();
		const chatInputElement = document.getElementById('chat-input');
		await tick();
		chatInputElement?.focus();
		await tick();
	});
</script>

{#if showSubOptions ? filteredSubItems.length > 0 : filteredItems.length > 0}
	<div
		id="commands-container"
		class="absolute bottom-0 left-0 right-0 z-10 w-full px-2 mb-2 text-left"
	>
		<div class="w-full overflow-hidden border border-gray-100 shadow-md rounded-xl dark:border-gray-800 backdrop-blur-sm">
			<div
				class="w-full transition-all duration-200 bg-white/95 dark:bg-gray-900/95 max-h-60 sm:max-h-80 rounded-xl"
			>
				{#if showSubOptions}
					<div class="sticky top-0 z-10 border-b border-gray-100 dark:border-gray-800 bg-white/95 dark:bg-gray-900/95">
						<button
							class="flex items-center justify-between w-full px-3 py-2.5 text-gray-600 dark:text-gray-300 transition-colors hover:bg-gray-50 dark:hover:bg-gray-800"
							type="button"
							on:click={goBack}
						>
							<div class="flex items-center">
								<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="w-4 h-4 mr-2 text-gray-500 dark:text-gray-400">
									<path stroke-linecap="round" stroke-linejoin="round" d="M10.5 19.5L3 12m0 0l7.5-7.5M3 12h18" />
								</svg>
								<span class="font-medium">{selectedCategory}</span>
							</div>
						</button>
					</div>
				{/if}
				
				<div class="p-1 overflow-y-auto space-y-1 scrollbar-hidden {showSubOptions ? 'max-h-[calc(60vh-2.75rem)]' : 'max-h-60'}">
					{#if showSubOptions}
						{#if filteredSubItems.length === 0}
							<div class="px-3 py-6 text-center text-gray-500 dark:text-gray-400">
								Aucun tag trouvé dans cette catégorie
							</div>
						{:else}
							{#if filteredSubItems.length > 4}
								<div class="px-3 py-1 text-xs font-medium text-gray-500 uppercase dark:text-gray-400">
									Tags {selectedCategory}
								</div>
							{/if}
							<div class="flex flex-col space-y-1">
								{#each filteredSubItems as tag, tagIdx}
									<button
										class="group px-3 py-2 rounded-lg w-full text-left transition-colors duration-100 hover:bg-gray-50 dark:hover:bg-gray-800 {tagIdx === selectedIdx
											? 'bg-gray-50/80 dark:bg-gray-800/80 ring-1 ring-gray-200 dark:ring-gray-700 selected-command-option-button'
											: ''}"
										type="button"
										on:click={() => confirmSelect(tag)}
										on:mousemove={() => { selectedIdx = tagIdx; }}
									>
										<div class="flex items-center justify-between">
											<div class="flex items-center space-x-2">
												<div 
													class="w-3 h-3 rounded-full" 
													style="background-color: {tag.color};"
												></div>
												<span class="font-medium text-gray-900 dark:text-gray-100">{tag.name}</span>
											</div>
											<span class="text-xs text-gray-500 transition-opacity duration-200 opacity-0 dark:text-gray-400 group-hover:opacity-100">
												Ajouter
											</span>
										</div>
									</button>
								{/each}
							</div>
						{/if}
					{:else}
						{#each filteredItems as item, idx}
							{#if isCategory(item)}
								<button
									class="group px-3 py-2 rounded-lg w-full text-left transition-colors duration-100 hover:bg-gray-50 dark:hover:bg-gray-800 {idx === selectedIdx
										? 'bg-gray-50/80 dark:bg-gray-800/80 ring-1 ring-gray-200 dark:ring-gray-700 selected-command-option-button'
										: ''}"
									type="button"
									on:click={() => selectCategory(item.name)}
									on:mousemove={() => { selectedIdx = idx; }}
								>
									<div class="flex items-center justify-between">
										<div class="flex items-center space-x-2">
											<div class="flex items-center justify-center w-5 h-5 text-sm">{item.icon}</div>
											<span class="font-medium text-gray-900 dark:text-gray-100">{item.name}</span>
										</div>
										<div class="flex items-center text-gray-500 dark:text-gray-400">
											<span class="mr-1 text-xs transition-opacity duration-200 opacity-0 group-hover:opacity-100">
												Voir les tags
											</span>
											<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="w-3.5 h-3.5">
												<path stroke-linecap="round" stroke-linejoin="round" d="M8.25 4.5l7.5 7.5-7.5 7.5" />
											</svg>
										</div>
									</div>
								</button>
							{:else}
								<button
									class="group px-3 py-2 rounded-lg w-full text-left transition-colors duration-100 hover:bg-gray-50 dark:hover:bg-gray-800 {idx === selectedIdx
										? 'bg-gray-50/80 dark:bg-gray-800/80 ring-1 ring-gray-200 dark:ring-gray-700 selected-command-option-button'
										: ''}"
									type="button"
									on:click={() => confirmSelect(item)}
									on:mousemove={() => { selectedIdx = idx; }}
								>
									<div class="flex items-center justify-between">
										<div class="flex items-center space-x-2">
											<div 
												class="w-3 h-3 rounded-full" 
												style="background-color: {item.color};"
											></div>
											<span class="font-medium text-gray-900 dark:text-gray-100">{item.name}</span>
										</div>
										<span class="text-xs text-gray-500 transition-opacity duration-200 opacity-0 dark:text-gray-400 group-hover:opacity-100">
											Ajouter
										</span>
									</div>
								</button>
							{/if}
						{/each}
					{/if}
				</div>
			</div>
		</div>
	</div>
{/if} 