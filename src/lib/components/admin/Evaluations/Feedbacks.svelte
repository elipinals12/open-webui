<script lang="ts">
	import { toast } from 'svelte-sonner';
	import fileSaver from 'file-saver';
	const { saveAs } = fileSaver;

	import dayjs from 'dayjs';
	import relativeTime from 'dayjs/plugin/relativeTime';
	dayjs.extend(relativeTime);

	import { onMount, getContext } from 'svelte';
	const i18n = getContext('i18n');

	import { deleteFeedbackById, exportAllFeedbacks, getAllFeedbacks } from '$lib/apis/evaluations';

	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import ArrowDownTray from '$lib/components/icons/ArrowDownTray.svelte';
	import Badge from '$lib/components/common/Badge.svelte';
	import CloudArrowUp from '$lib/components/icons/CloudArrowUp.svelte';
	import Pagination from '$lib/components/common/Pagination.svelte';
	import FeedbackMenu from './FeedbackMenu.svelte';
	import EllipsisHorizontal from '$lib/components/icons/EllipsisHorizontal.svelte';
	import ChartBar from '$lib/components/icons/ChartBar.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';

	export let feedbacks = [];
	let originalFeedbacksData = []; // Store the original data
	let parsedFeedbacksData = []; // Store the filtered data for display
	let showFeedbackAnalysis = false;
	let activeFilter = 'all';
	let searchQuery = '';

	let page = 1;
	$: paginatedFeedbacks = feedbacks.slice((page - 1) * 10, page * 10);

	type Feedback = {
		id: string;
		data: {
			rating: number;
			model_id: string;
			sibling_model_ids: string[] | null;
			reason: string;
			comment: string;
			tags: string[];
			details?: {
				rating?: number;
			};
		};
		user: {
			name: string;
			profile_image_url: string;
		};
		updated_at: number;
		created_at?: number;
		browser_id?: string;
		snapshot?: {
			chat: {
				title: string;
				chat: {
					messages: Array<{
						role: string;
						content: string;
						annotation?: string;
						feedbackId?: string;
						error?: {
							content: string;
						};
					}>;
				};
			};
		};
		meta?: {
			model_id?: string;
			tags?: string[];
		};
	};

	type ModelStats = {
		rating: number;
		won: number;
		lost: number;
	};

	// Summary stats for parser view
	let totalItems = 0;
	let positiveCount = 0;
	let negativeCount = 0;

	//////////////////////
	//
	// CRUD operations
	//
	//////////////////////

	const deleteFeedbackHandler = async (feedbackId: string) => {
		const response = await deleteFeedbackById(localStorage.token, feedbackId).catch((err) => {
			toast.error(err);
			return null;
		});
		if (response) {
			feedbacks = feedbacks.filter((f) => f.id !== feedbackId);
		}
	};

	const shareHandler = async () => {
		toast.success($i18n.t('Redirecting you to Open WebUI Community'));

		// remove snapshot from feedbacks
		const feedbacksToShare = feedbacks.map((f) => {
			const { snapshot, user, ...rest } = f;
			return rest;
		});
		console.log(feedbacksToShare);

		const url = 'https://openwebui.com';
		const tab = await window.open(`${url}/leaderboard`, '_blank');

		// Define the event handler function
		const messageHandler = (event) => {
			if (event.origin !== url) return;
			if (event.data === 'loaded') {
				tab.postMessage(JSON.stringify(feedbacksToShare), '*');

				// Remove the event listener after handling the message
				window.removeEventListener('message', messageHandler);
			}
		};

		window.addEventListener('message', messageHandler, false);
	};

	const exportHandler = async () => {
		const _feedbacks = await exportAllFeedbacks(localStorage.token).catch((err) => {
			toast.error(err);
			return null;
		});

		if (_feedbacks) {
			let blob = new Blob([JSON.stringify(_feedbacks)], {
				type: 'application/json'
			});
			saveAs(blob, `feedback-history-export-${Date.now()}.json`);
		}
	};

	//////////////////////
	//
	// Feedback Parser Functions
	//
	//////////////////////

	// Add some debugging to help trace issues
	const debugStructure = (item) => {
		console.log('Feedback item structure:', {
			id: item.id,
			hasSnapshot: !!item.snapshot,
			title: item.snapshot?.chat?.title,
			messages: !!item.snapshot?.chat?.chat?.messages,
			messageCount: item.snapshot?.chat?.chat?.messages?.length || 0
		});
		
		if (item.snapshot?.chat?.chat?.messages?.length > 0) {
			console.log('First message structure:', item.snapshot.chat.chat.messages[0]);
		}
	};

	const toggleFeedbackAnalysis = () => {
		showFeedbackAnalysis = !showFeedbackAnalysis;
		if (showFeedbackAnalysis && parsedFeedbacksData.length === 0) {
			processFeedbackData([...feedbacks]);
		}
	};

	const processFeedbackData = (feedbackData) => {
		originalFeedbacksData = [...feedbackData]; // Store a copy of the original data
		
		// Calculate summary statistics
		totalItems = feedbackData.length;
		positiveCount = 0;
		negativeCount = 0;
		
		feedbackData.forEach(item => {
			// Debug the structure of each item to help diagnose issues
			debugStructure(item);
			
			const rating = item.data?.rating;
			if (rating > 0) positiveCount++;
			else if (rating < 0) negativeCount++;
		});
		
		// Set initial filter to show all items
		filterFeedbackItems('all', '');
	};
	
	const filterFeedbackItems = (filterType, query = '') => {
		activeFilter = filterType;
		searchQuery = query;
		
		// Always start with the original data
		let filteredItems = [...originalFeedbacksData];
		
		// Apply rating filter
		if (filterType === 'positive') {
			filteredItems = filteredItems.filter(item => item.data?.rating > 0);
		} else if (filterType === 'negative') {
			filteredItems = filteredItems.filter(item => item.data?.rating < 0);
		}
		
		// Apply search query if present
		if (query.trim()) {
			const lowerQuery = query.toLowerCase();
			filteredItems = filteredItems.filter(item => {
				// Search in chat title
				if (item.snapshot?.chat?.title && 
					item.snapshot.chat.title.toLowerCase().includes(lowerQuery)) {
					return true;
				}
				
				// Search in messages
				const messages = item.snapshot?.chat?.chat?.messages;
				if (Array.isArray(messages)) {
					return messages.some(message => 
						message.content && message.content.toLowerCase().includes(lowerQuery)
					);
				}
				
				return false;
			});
		}
		
		// Set the display data to the filtered results
		parsedFeedbacksData = filteredItems;
		
		// For debugging
		console.log(`Filter: ${filterType}, Query: "${query}", Results: ${filteredItems.length}/${originalFeedbacksData.length}`);
	};
	
	const formatConversation = (messages) => {
		if (!Array.isArray(messages)) {
			return '<p class="text-gray-500 dark:text-gray-400 text-sm">No conversation data available</p>';
		}
		
		return messages.map(message => {
			const isUser = message.role === 'user';
			const className = isUser 
				? 'bg-gray-50 dark:bg-gray-800 p-3 rounded-lg mb-2'
				: 'bg-gray-100 dark:bg-gray-700 p-3 rounded-lg mb-2';
			
			// Check for annotation/feedback
			const hasFeedback = message.annotation || message.feedbackId;
			const feedbackHighlight = hasFeedback ? 'border-2 border-red-500 dark:border-red-400' : '';
			
			// Check for content availability and handle error messages
			let displayContent = message.content || '';
			if (message.error && message.error.content) {
				displayContent += `<div class="text-red-500 dark:text-red-400 text-xs mt-1">${message.error.content}</div>`;
			}
			
			return `
				<div class="${className} ${feedbackHighlight}">
					<div class="font-medium text-gray-700 dark:text-gray-300">${isUser ? 'User' : 'Assistant'}</div>
					<div class="text-sm mt-1 text-gray-600 dark:text-gray-300">${displayContent}</div>
					${hasFeedback ? '<div class="text-red-500 dark:text-red-400 text-xs font-medium mt-1">[This message received feedback]</div>' : ''}
				</div>
			`;
		}).join('');
	};

	// Handle refreshing feedbacks for analysis
	const refreshFeedbacksAnalysis = async () => {
		toast.info($i18n.t('Refreshing feedback analysis...'));
		
		try {
			const refreshedFeedbacks = await exportAllFeedbacks(localStorage.token);
			if (refreshedFeedbacks) {
				// Process the fresh data
				processFeedbackData(refreshedFeedbacks);
				toast.success($i18n.t('Feedback analysis refreshed successfully!'));
			}
		} catch (err) {
			toast.error(err);
			console.error("Error refreshing feedback analysis:", err);
		}
	};
</script>

<div class="mt-0.5 mb-2 gap-1 flex flex-row justify-between">
	<div class="flex md:self-center text-lg font-medium px-0.5">
		{$i18n.t('Feedback History')}

		<div class="flex self-center w-[1px] h-6 mx-2.5 bg-gray-50 dark:bg-gray-850" />

		<span class="text-lg font-medium text-gray-500 dark:text-gray-300">{feedbacks.length}</span>
	</div>

	<div class="flex gap-1">
		{#if feedbacks.length > 0}
			<!-- Export Button -->
			<Tooltip content={$i18n.t('Export')}>
				<button
					class="p-2 rounded-xl hover:bg-gray-100 dark:bg-gray-900 dark:hover:bg-gray-850 transition font-medium text-sm flex items-center space-x-1"
					on:click={() => {
						exportHandler();
					}}
				>
					<ArrowDownTray className="size-3" />
				</button>
			</Tooltip>

			<!-- Analysis Button -->
			<Tooltip content={$i18n.t('Analyze Feedback')}>
				<button
					class="p-2 rounded-xl hover:bg-gray-100 dark:bg-gray-900 dark:hover:bg-gray-850 transition font-medium text-sm flex items-center space-x-1"
					on:click={toggleFeedbackAnalysis}
				>
					<ChartBar className="size-3" />
				</button>
			</Tooltip>
		{/if}
	</div>
</div>

<div
	class="scrollbar-hidden relative whitespace-nowrap overflow-x-auto max-w-full rounded-sm pt-0.5"
>
	{#if (feedbacks ?? []).length === 0}
		<div class="text-center text-xs text-gray-500 dark:text-gray-400 py-1">
			{$i18n.t('No feedbacks found')}
		</div>
	{:else}
		<table
			class="w-full text-sm text-left text-gray-500 dark:text-gray-400 table-auto max-w-full rounded-sm"
		>
			<thead
				class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-850 dark:text-gray-400 -translate-y-0.5"
			>
				<tr class="">
					<th scope="col" class="px-3 text-right cursor-pointer select-none w-0">
						{$i18n.t('User')}
					</th>

					<th scope="col" class="px-3 pr-1.5 cursor-pointer select-none">
						{$i18n.t('Models')}
					</th>

					<th scope="col" class="px-3 py-1.5 text-right cursor-pointer select-none w-fit">
						{$i18n.t('Result')}
					</th>

					<th scope="col" class="px-3 py-1.5 text-right cursor-pointer select-none w-0">
						{$i18n.t('Updated At')}
					</th>

					<th scope="col" class="px-3 py-1.5 text-right cursor-pointer select-none w-0"> </th>
				</tr>
			</thead>
			<tbody class="">
				{#each paginatedFeedbacks as feedback (feedback.id)}
					<tr class="bg-white dark:bg-gray-900 dark:border-gray-850 text-xs">
						<td class=" py-0.5 text-right font-semibold">
							<div class="flex justify-center">
								<Tooltip content={feedback?.user?.name}>
									<div class="shrink-0">
										<img
											src={feedback?.user?.profile_image_url ?? '/user.png'}
											alt={feedback?.user?.name}
											class="size-5 rounded-full object-cover shrink-0"
										/>
									</div>
								</Tooltip>
							</div>
						</td>

						<td class=" py-1 pl-3 flex flex-col">
							<div class="flex flex-col items-start gap-0.5 h-full">
								<div class="flex flex-col h-full">
									{#if feedback.data?.sibling_model_ids}
										<div class="font-semibold text-gray-600 dark:text-gray-400 flex-1">
											{feedback.data?.model_id}
										</div>

										<Tooltip content={feedback.data.sibling_model_ids.join(', ')}>
											<div class=" text-[0.65rem] text-gray-600 dark:text-gray-400 line-clamp-1">
												{#if feedback.data.sibling_model_ids.length > 2}
													{feedback.data.sibling_model_ids.slice(0, 2).join(', ')}, {$i18n.t(
														'and {{COUNT}} more',
														{ COUNT: feedback.data.sibling_model_ids.length - 2 }
													)}
												{:else}
													{feedback.data.sibling_model_ids.join(', ')}
												{/if}
											</div>
										</Tooltip>
									{:else}
										<div
											class=" text-sm font-medium text-gray-600 dark:text-gray-400 flex-1 py-1.5"
										>
											{feedback.data?.model_id}
										</div>
									{/if}
								</div>
							</div>
						</td>
						<td class="px-3 py-1 text-right font-medium text-gray-900 dark:text-white w-max">
							<div class=" flex justify-end">
								{#if feedback.data.rating.toString() === '1'}
									<Badge type="info" content={$i18n.t('Won')} />
								{:else if feedback.data.rating.toString() === '0'}
									<Badge type="muted" content={$i18n.t('Draw')} />
								{:else if feedback.data.rating.toString() === '-1'}
									<Badge type="error" content={$i18n.t('Lost')} />
								{/if}
							</div>
						</td>

						<td class=" px-3 py-1 text-right font-medium">
							{dayjs(feedback.updated_at * 1000).fromNow()}
						</td>

						<td class=" px-3 py-1 text-right font-semibold">
							<FeedbackMenu
								on:delete={(e) => {
									deleteFeedbackHandler(feedback.id);
								}}
							>
								<button
									class="self-center w-fit text-sm p-1.5 dark:text-gray-300 dark:hover:text-white hover:bg-black/5 dark:hover:bg-white/5 rounded-xl"
								>
									<EllipsisHorizontal />
								</button>
							</FeedbackMenu>
						</td>
					</tr>
				{/each}
			</tbody>
		</table>
	{/if}
</div>

{#if feedbacks.length > 0}
	<div class=" flex flex-col justify-end w-full text-right gap-1">
		<div class="line-clamp-1 text-gray-500 text-xs">
			{$i18n.t('Help us create the best community leaderboard by sharing your feedback history!')}
		</div>

		<div class="flex space-x-1 ml-auto">
			<Tooltip
				content={$i18n.t(
					'To protect your privacy, only ratings, model IDs, tags, and metadata are shared from your feedback—your chat logs remain private and are not included.'
				)}
			>
				<button
					class="flex text-xs items-center px-3 py-1.5 rounded-xl bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-gray-200 transition"
					on:click={async () => {
						shareHandler();
					}}
				>
					<div class=" self-center mr-2 font-medium line-clamp-1">
						{$i18n.t('Share to Open WebUI Community')}
					</div>

					<div class=" self-center">
						<CloudArrowUp className="size-3" strokeWidth="3" />
					</div>
				</button>
			</Tooltip>
		</div>
	</div>
{/if}

{#if feedbacks.length > 10}
	<Pagination bind:page count={feedbacks.length} perPage={10} />
{/if}

<!-- Feedback Analysis Section -->
{#if showFeedbackAnalysis}
<div class="mt-8 border-t border-gray-100 dark:border-gray-800 pt-4">
	<div class="flex justify-between items-center mb-4">
		<h2 class="text-lg font-medium text-gray-900 dark:text-white">{$i18n.t('Feedback Analysis')}</h2>
		<div class="flex gap-2">
			<button
				class="p-2 rounded-xl bg-gray-50 hover:bg-gray-100 dark:bg-gray-850 dark:hover:bg-gray-800 text-gray-600 dark:text-gray-300 transition text-sm"
				on:click={refreshFeedbacksAnalysis}
			>
				{$i18n.t('Refresh Analysis')}
			</button>
			<button
				class="p-2 rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 text-gray-600 dark:text-gray-300 transition"
				on:click={toggleFeedbackAnalysis}
			>
				<XMark className="size-4" />
			</button>
		</div>
	</div>

	<div class="bg-gray-50 dark:bg-gray-850 p-4 rounded-lg mb-4">
		<h3 class="text-base font-medium text-gray-800 dark:text-gray-200 mb-2">{$i18n.t('Summary')}</h3>
		<div class="grid grid-cols-1 md:grid-cols-3 gap-2">
			<div class="bg-white dark:bg-gray-900 p-3 rounded-lg">
				<div class="text-sm text-gray-500 dark:text-gray-400">{$i18n.t('Total Feedback')}</div>
				<div class="text-xl font-bold text-gray-900 dark:text-white">{totalItems}</div>
			</div>
			<div class="bg-white dark:bg-gray-900 p-3 rounded-lg">
				<div class="text-sm text-gray-500 dark:text-gray-400">{$i18n.t('Positive Ratings')}</div>
				<div class="text-xl font-bold text-emerald-600 dark:text-emerald-400">
					{positiveCount} ({totalItems ? Math.round((positiveCount / totalItems) * 100) : 0}%)
				</div>
			</div>
			<div class="bg-white dark:bg-gray-900 p-3 rounded-lg">
				<div class="text-sm text-gray-500 dark:text-gray-400">{$i18n.t('Negative Ratings')}</div>
				<div class="text-xl font-bold text-red-600 dark:text-red-400">
					{negativeCount} ({totalItems ? Math.round((negativeCount / totalItems) * 100) : 0}%)
				</div>
			</div>
		</div>
	</div>

	<div class="flex flex-col sm:flex-row gap-2 mb-4">
		<input 
			type="text" 
			placeholder={$i18n.t('Search in conversations...')}
			class="flex-1 px-3 py-2 border border-gray-200 dark:border-gray-700 rounded-lg text-sm bg-white dark:bg-gray-900 text-gray-800 dark:text-gray-200 focus:outline-none focus:ring-2 focus:ring-blue-500"
			bind:value={searchQuery}
			on:input={(e) => filterFeedbackItems(activeFilter, e.target.value)}
		/>
		
		<div class="flex gap-1">
			<button 
				class={`px-3 py-2 text-sm rounded-lg transition ${activeFilter === 'all' ? 'bg-blue-500 text-white' : 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300'}`}
				on:click={() => filterFeedbackItems('all', searchQuery)}
			>
				{$i18n.t('All')}
			</button>
			<button 
				class={`px-3 py-2 text-sm rounded-lg transition ${activeFilter === 'positive' ? 'bg-blue-500 text-white' : 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300'}`}
				on:click={() => filterFeedbackItems('positive', searchQuery)}
			>
				{$i18n.t('Positive')}
			</button>
			<button 
				class={`px-3 py-2 text-sm rounded-lg transition ${activeFilter === 'negative' ? 'bg-blue-500 text-white' : 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300'}`}
				on:click={() => filterFeedbackItems('negative', searchQuery)}
			>
				{$i18n.t('Negative')}
			</button>
		</div>
	</div>

	{#if parsedFeedbacksData.length === 0}
		<div class="text-center py-8 text-gray-500 dark:text-gray-400">
			{$i18n.t('No matching feedback items found')}
		</div>
	{:else}
		<div class="space-y-4 mb-4">
			{#each parsedFeedbacksData as item (item.id)}
				<div class={`bg-white dark:bg-gray-900 p-4 rounded-lg border-l-4 ${item.data.rating > 0 ? 'border-emerald-500' : item.data.rating < 0 ? 'border-red-500' : 'border-gray-300 dark:border-gray-600'}`}>
					<div>
					<h3 class="text-base font-medium text-gray-800 dark:text-gray-200 mb-2">{item.snapshot?.chat?.title || item.meta?.model_id || $i18n.t('Untitled Chat')}</h3>
					
					<div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-3">
						<div>
							<p class="text-xs text-gray-500 dark:text-gray-400 mb-1">{$i18n.t('Browser ID')}: <span class="font-medium text-gray-700 dark:text-gray-300">{item.browser_id || 'N/A'}</span></p>
							<p class="text-xs text-gray-500 dark:text-gray-400 mb-1">{$i18n.t('Timestamp')}: <span class="font-medium text-gray-700 dark:text-gray-300">{dayjs((item.created_at || item.updated_at) * 1000).format('YYYY-MM-DD HH:mm:ss')}</span></p>
							<p class="text-xs text-gray-500 dark:text-gray-400 mb-1">
								{$i18n.t('Rating')}: 
								<span class={`font-medium ${item.data.rating > 0 ? 'text-emerald-600 dark:text-emerald-400' : item.data.rating < 0 ? 'text-red-600 dark:text-red-400' : 'text-gray-700 dark:text-gray-300'}`}>
									{item.data.rating > 0 ? $i18n.t('Positive') : item.data.rating < 0 ? $i18n.t('Negative') : $i18n.t('Neutral')}
									{item.data.details?.rating !== undefined ? ` (${item.data.details.rating}/10)` : ''}
								</span>
							</p>
						</div>
						<div>
							<p class="text-xs text-gray-500 dark:text-gray-400 mb-1">{$i18n.t('Reason')}: <span class="font-medium text-gray-700 dark:text-gray-300">{item.data.reason || $i18n.t('Not specified')}</span></p>
							<p class="text-xs text-gray-500 dark:text-gray-400 mb-1">{$i18n.t('Comment')}: <span class="font-medium text-gray-700 dark:text-gray-300">{item.data.comment || $i18n.t('No comment provided')}</span></p>
							{#if item.data.tags && item.data.tags.length > 0}
								<div class="flex flex-wrap gap-1 mt-1">
									{#each item.data.tags as tag}
										<span class="px-2 py-1 text-xs rounded-full bg-blue-100 dark:bg-blue-900 text-blue-800 dark:text-blue-200">{tag}</span>
									{/each}
								</div>
							{/if}
						</div>
					</div>
					
					<div class="mt-3">
						<h4 class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">{$i18n.t('Conversation')}:</h4>
						<div class="max-h-96 overflow-y-auto p-2 bg-gray-50 dark:bg-gray-800 rounded-lg">
							{#if item.snapshot?.chat?.chat?.messages}
								{@html formatConversation(item.snapshot.chat.chat.messages)}
							{:else}
								<p class="text-sm text-gray-500 dark:text-gray-400">{$i18n.t('No conversation data available')}</p>
							{/if}
						</div>
					</div>
				</div>
				</div>
			{/each}
		</div>
	{/if}
</div>
{/if}