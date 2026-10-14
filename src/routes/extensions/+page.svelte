<script lang="ts">
	import { onMount } from 'svelte';
	import {
		Network,
		ArrowLeft,
		Search,
		Plus,
		Download,
		RotateCcw,
		Trash2,
		ExternalLink,
		Check,
		Copy,
		X,
		UploadCloud,
		FileCode,
		Globe,
		SlidersHorizontal,
		Info,
		Layers,
		Sparkles,
		Eye,
		RefreshCw,
		Blocks,
		Code2,
		ShieldCheck
	} from '@lucide/svelte';
	import DynamicIcon from '../../lib/components/DynamicIcon.svelte';
	import type { ExtensionPackage, ExtensionAccentColor } from '../../lib/types';
	import type { PageData } from './$types';

	let { data }: { data: PageData } = $props();

	// svelte-ignore state_referenced_locally
	let extensionsList = $state<ExtensionPackage[]>(data?.extensions || []);
	let isLoading = $state(false);
	let searchQuery = $state('');
	let selectedCategory = $state('ALL');
	let selectedStatus = $state<'ALL' | 'ENABLED' | 'DISABLED' | 'BUILTIN' | 'COMMUNITY'>('ALL');

	// Notification feedback
	let toastMessage = $state<string | null>(null);
	let toastType = $state<'success' | 'error'>('success');
	let copiedManifest = $state(false);

	function showToast(msg: string, type: 'success' | 'error' = 'success') {
		toastMessage = msg;
		toastType = type;
		setTimeout(() => {
			toastMessage = null;
		}, 3000);
	}

	// Import Modal state
	let isImportOpen = $state(false);
	let activeImportTab = $state<'upload' | 'url' | 'editor' | 'registry'>('registry');
	let importUrl = $state('');
	let importJsonText = $state('');
	let isImporting = $state(false);
	let importError = $state<string | null>(null);

	// Inspect Modal state
	let isInspectOpen = $state(false);
	let inspectedExt = $state<ExtensionPackage | null>(null);
	let inspectActiveTab = $state<'overview' | 'properties' | 'manifest'>('overview');

	// Delete confirmation modal state
	let isDeleteOpen = $state(false);
	let extToDelete = $state<ExtensionPackage | null>(null);
	let isDeleting = $state(false);

	// Load extensions from API
	async function loadExtensions() {
		isLoading = true;
		try {
			const res = await fetch('/api/extensions');
			if (res.ok) {
				const json = await res.json();
				extensionsList = json.extensions || [];
			}
		} catch (err) {
			console.error('Failed to load extensions:', err);
		} finally {
			isLoading = false;
		}
	}

	onMount(() => {
		loadExtensions();
	});

	// Toggle enabled state
	async function toggleExtension(ext: ExtensionPackage) {
		const nextState = !ext.enabled;
		// Optimistic update
		extensionsList = extensionsList.map((item) =>
			item.id === ext.id ? { ...item, enabled: nextState } : item
		);

		try {
			const res = await fetch('/api/extensions', {
				method: 'PATCH',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ id: ext.id, enabled: nextState })
			});
			if (!res.ok) {
				// Revert on failure
				extensionsList = extensionsList.map((item) =>
					item.id === ext.id ? { ...item, enabled: !nextState } : item
				);
				showToast('Failed to update extension status', 'error');
			} else {
				showToast(`Extension "${ext.name}" ${nextState ? 'enabled' : 'disabled'}`);
			}
		} catch (err) {
			extensionsList = extensionsList.map((item) =>
				item.id === ext.id ? { ...item, enabled: !nextState } : item
			);
			showToast('Network error while toggling extension', 'error');
		}
	}

	// Delete extension
	async function confirmDelete() {
		if (!extToDelete) return;
		isDeleting = true;
		try {
			const res = await fetch(`/api/extensions?id=${encodeURIComponent(extToDelete.id)}`, {
				method: 'DELETE'
			});
			if (res.ok) {
				extensionsList = extensionsList.filter((item) => item.id !== extToDelete?.id);
				showToast(`Extension "${extToDelete.name}" uninstalled successfully`);
				isDeleteOpen = false;
				extToDelete = null;
			} else {
				const err = await res.json().catch(() => ({}));
				showToast(err.error || 'Failed to delete extension', 'error');
			}
		} catch (err) {
			showToast('Network error while deleting extension', 'error');
		} finally {
			isDeleting = false;
		}
	}

	// Reset to defaults
	async function handleResetDefaults() {
		if (
			!confirm('Reset all extensions to Nodeflow defaults? Custom imported extensions may be lost.')
		)
			return;
		isLoading = true;
		try {
			const res = await fetch('/api/extensions', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ reset: true })
			});
			if (res.ok) {
				const json = await res.json();
				extensionsList = json.extensions || [];
				showToast('Extensions reset to factory defaults');
			}
		} catch (err) {
			showToast('Failed to reset extensions', 'error');
		} finally {
			isLoading = false;
		}
	}

	// Import via Manifest JSON
	async function submitImportJson() {
		if (!importJsonText.trim()) {
			importError = 'Please paste a JSON manifest';
			return;
		}
		isImporting = true;
		importError = null;
		try {
			const parsed = JSON.parse(importJsonText);
			const res = await fetch('/api/extensions', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ manifest: parsed })
			});
			const data = await res.json();
			if (!res.ok || !data.success) {
				importError = data.error || 'Failed to import manifest';
				return;
			}
			await loadExtensions();
			isImportOpen = false;
			importJsonText = '';
			showToast(data.message || 'Extension imported successfully!');
		} catch (err: any) {
			importError = err.message || 'Invalid JSON syntax';
		} finally {
			isImporting = false;
		}
	}

	// Import via URL
	async function submitImportUrl() {
		if (!importUrl.trim()) {
			importError = 'Please enter a valid URL';
			return;
		}
		isImporting = true;
		importError = null;
		try {
			const res = await fetch('/api/extensions', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ url: importUrl.trim() })
			});
			const data = await res.json();
			if (!res.ok || !data.success) {
				importError = data.error || 'Failed to fetch extension from URL';
				return;
			}
			await loadExtensions();
			isImportOpen = false;
			importUrl = '';
			showToast(data.message || 'Extension imported successfully!');
		} catch (err: any) {
			importError = err.message || 'Network error fetching URL';
		} finally {
			isImporting = false;
		}
	}

	// File Upload Handler
	function handleFileUpload(e: Event) {
		const target = e.target as HTMLInputElement;
		const file = target.files?.[0];
		if (!file) return;

		const reader = new FileReader();
		reader.onload = async (evt) => {
			const content = evt.target?.result as string;
			importJsonText = content;
			activeImportTab = 'editor';
		};
		reader.readAsText(file);
	}

	// Install a curated registry item
	async function installCurated(curatedTemplate: any) {
		isImporting = true;
		try {
			const res = await fetch('/api/extensions', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ manifest: curatedTemplate })
			});
			const data = await res.json();
			if (res.ok && data.success) {
				await loadExtensions();
				isImportOpen = false;
				showToast(`Extension "${curatedTemplate.name}" installed!`);
			} else {
				showToast(data.error || 'Failed to install extension', 'error');
			}
		} catch (err) {
			showToast('Network error installing extension', 'error');
		} finally {
			isImporting = false;
		}
	}

	// Sample template for code editor
	const sampleManifestTemplate = JSON.stringify(
		{
			id: 'ext_sample_webhook',
			name: 'Custom Service Webhook',
			version: '1.0.0',
			description: 'Dispatch custom webhooks to any internal microservice.',
			category: 'Integrations',
			author: 'Your Name or Team',
			icon: 'Send',
			accentColor: 'indigo',
			nodeType: 'customWebhook',
			enabled: true,
			tags: ['webhook', 'custom', 'api'],
			nodeDefinition: {
				title: 'Custom Webhook',
				badgeText: 'Webhook',
				width: 'w-80',
				hasInputHandle: true,
				hasOutputHandle: false,
				outputs: [
					{ id: 'success', label: 'SUCCESS', color: 'emerald' },
					{ id: 'error', label: 'ERROR', color: 'rose' }
				],
				properties: [
					{
						name: 'endpointUrl',
						label: 'Endpoint URL',
						type: 'text',
						placeholder: 'https://api.mycompany.com/webhook',
						defaultValue: 'https://api.mycompany.com/webhook'
					},
					{
						name: 'authHeader',
						label: 'Auth Token / Key (Optional)',
						type: 'password',
						placeholder: 'Bearer token...'
					},
					{
						name: 'payloadTemplate',
						label: 'JSON Payload Template',
						type: 'textarea',
						defaultValue: '{"event": "nodeflow_trigger", "data": {{payload}}}'
					}
				],
				defaultData: {
					endpointUrl: 'https://api.mycompany.com/webhook',
					payloadTemplate: '{"event": "nodeflow_trigger", "data": {{payload}}}'
				}
			}
		},
		null,
		2
	);

	function openInspect(ext: ExtensionPackage) {
		inspectedExt = ext;
		inspectActiveTab = 'overview';
		isInspectOpen = true;
	}

	function copyManifestToClipboard() {
		if (!inspectedExt) return;
		navigator.clipboard.writeText(JSON.stringify(inspectedExt, null, 2));
		copiedManifest = true;
		setTimeout(() => (copiedManifest = false), 2000);
	}

	// Curated registry catalog
	const curatedCatalog = [
		{
			name: 'Discord Webhook Dispatcher',
			icon: 'MessageSquare',
			category: 'Integrations',
			accentColor: 'indigo',
			version: '1.0.0',
			desc: 'Deliver rich messages, notifications, and embeds directly to Discord server channels.',
			tags: ['discord', 'chat', 'webhook'],
			manifest: {
				id: 'ext_discord_webhook',
				name: 'Discord Webhook Dispatcher',
				version: '1.0.0',
				description:
					'Deliver rich messages, notifications, and embeds directly to Discord server channels.',
				category: 'Integrations',
				author: 'Community / Discord',
				icon: 'MessageSquare',
				accentColor: 'indigo',
				nodeType: 'discordWebhook',
				enabled: true,
				tags: ['discord', 'chat', 'webhook'],
				nodeDefinition: {
					title: 'Discord Webhook',
					badgeText: 'Discord',
					width: 'w-80',
					hasInputHandle: true,
					hasOutputHandle: false,
					outputs: [
						{ id: 'success', label: 'SENT', color: 'emerald' },
						{ id: 'error', label: 'FAILED', color: 'rose' }
					],
					properties: [
						{
							name: 'webhookUrl',
							label: 'Discord Webhook URL',
							type: 'password',
							placeholder: 'https://discord.com/api/webhooks/...'
						},
						{
							name: 'content',
							label: 'Message Content',
							type: 'textarea',
							defaultValue: 'Event triggered: {{payload.summary || payload.text}}'
						},
						{
							name: 'username',
							label: 'Bot Username (Optional)',
							type: 'text',
							placeholder: 'Nodeflow Bot'
						}
					],
					defaultData: {
						content: 'Event triggered: {{payload.summary || payload.text}}',
						username: 'Nodeflow Bot'
					}
				}
			}
		},
		{
			name: 'Stripe Payment Gateway',
			icon: 'CreditCard',
			category: 'Payments',
			accentColor: 'emerald',
			version: '1.0.0',
			desc: 'Create PaymentIntents, charges, and process customer checkouts securely.',
			tags: ['stripe', 'payments', 'checkout'],
			manifest: {
				id: 'ext_stripe_payments',
				name: 'Stripe Payment Gateway',
				version: '1.0.0',
				description: 'Create PaymentIntents, charges, and process customer checkouts securely.',
				category: 'Payments',
				author: 'Community / Stripe',
				icon: 'CreditCard',
				accentColor: 'emerald',
				nodeType: 'stripeCharge',
				enabled: true,
				tags: ['stripe', 'payments', 'checkout'],
				nodeDefinition: {
					title: 'Stripe Payments',
					badgeText: 'Stripe API',
					width: 'w-80',
					hasInputHandle: true,
					hasOutputHandle: false,
					outputs: [
						{ id: 'success', label: 'APPROVED', color: 'emerald' },
						{ id: 'error', label: 'DECLINED', color: 'rose' }
					],
					properties: [
						{
							name: 'apiKey',
							label: 'Stripe Secret Key',
							type: 'password',
							placeholder: 'sk_test_...'
						},
						{
							name: 'action',
							label: 'Action',
							type: 'select',
							defaultValue: 'createPaymentIntent',
							options: [
								{ label: 'Create Payment Intent', value: 'createPaymentIntent' },
								{ label: 'Create Customer', value: 'createCustomer' }
							]
						},
						{
							name: 'amountExpr',
							label: 'Amount (in cents) Expression',
							type: 'text',
							defaultValue: 'payload.amount || 2000'
						},
						{ name: 'currency', label: 'Currency', type: 'text', defaultValue: 'usd' }
					],
					defaultData: {
						action: 'createPaymentIntent',
						amountExpr: 'payload.amount || 2000',
						currency: 'usd'
					}
				}
			}
		},
		{
			name: 'Slack Block Kit Notifier',
			icon: 'Slack',
			category: 'Integrations',
			accentColor: 'purple',
			version: '1.0.0',
			desc: 'Post alerts and interactive block cards to Slack channels via incoming webhooks.',
			tags: ['slack', 'chat', 'alerts'],
			manifest: {
				id: 'ext_slack_notifier',
				name: 'Slack Block Kit Notifier',
				version: '1.0.0',
				description:
					'Post alerts and interactive block cards to Slack channels via incoming webhooks.',
				category: 'Integrations',
				author: 'Community / Slack',
				icon: 'Slack',
				accentColor: 'purple',
				nodeType: 'slackNotifier',
				enabled: true,
				tags: ['slack', 'chat', 'alerts'],
				nodeDefinition: {
					title: 'Slack Notifier',
					badgeText: 'Slack Bot',
					width: 'w-80',
					hasInputHandle: true,
					hasOutputHandle: false,
					outputs: [
						{ id: 'success', label: 'SUCCESS', color: 'emerald' },
						{ id: 'error', label: 'ERROR', color: 'rose' }
					],
					properties: [
						{
							name: 'webhookUrl',
							label: 'Incoming Webhook URL',
							type: 'password',
							placeholder: 'https://hooks.slack.com/services/...'
						},
						{ name: 'channel', label: 'Channel Override', type: 'text', placeholder: '#general' },
						{
							name: 'messageText',
							label: 'Message Text',
							type: 'textarea',
							defaultValue: 'Alert from Nodeflow: {{payload.summary}}'
						}
					],
					defaultData: { messageText: 'Alert from Nodeflow: {{payload.summary}}' }
				}
			}
		},
		{
			name: 'Resend Transactional Email',
			icon: 'Mail',
			category: 'Messaging',
			accentColor: 'teal',
			version: '1.0.0',
			desc: 'Deliver transactional emails with HTML templates and tracking through Resend API.',
			tags: ['email', 'resend', 'mailer'],
			manifest: {
				id: 'ext_resend_email',
				name: 'Resend Transactional Email',
				version: '1.0.0',
				description:
					'Deliver transactional emails with HTML templates and tracking through Resend API.',
				category: 'Messaging',
				author: 'Community / Resend',
				icon: 'Mail',
				accentColor: 'teal',
				nodeType: 'resendEmail',
				enabled: true,
				tags: ['email', 'resend', 'mailer'],
				nodeDefinition: {
					title: 'Resend Email',
					badgeText: 'Resend',
					width: 'w-80',
					hasInputHandle: true,
					hasOutputHandle: false,
					outputs: [
						{ id: 'success', label: 'SENT', color: 'emerald' },
						{ id: 'error', label: 'FAILED', color: 'rose' }
					],
					properties: [
						{ name: 'apiKey', label: 'Resend API Key', type: 'password', placeholder: 're_123...' },
						{
							name: 'from',
							label: 'From Email Address',
							type: 'text',
							defaultValue: 'notifications@example.com'
						},
						{
							name: 'toExpr',
							label: 'Recipient Email Expr',
							type: 'text',
							defaultValue: 'payload.email'
						},
						{
							name: 'subject',
							label: 'Subject Line',
							type: 'text',
							defaultValue: 'Welcome to our platform!'
						}
					],
					defaultData: {
						from: 'notifications@example.com',
						toExpr: 'payload.email',
						subject: 'Welcome to our platform!'
					}
				}
			}
		},
		{
			name: 'Redis Cache & Rate Limiter',
			icon: 'Database',
			category: 'Storage',
			accentColor: 'rose',
			version: '1.0.0',
			desc: 'Fast key-value cache and rate limit counter with configurable TTL.',
			tags: ['redis', 'cache', 'kv'],
			manifest: {
				id: 'ext_redis_cache',
				name: 'Redis Cache & Rate Limiter',
				version: '1.0.0',
				description: 'Fast key-value cache and rate limit counter with configurable TTL.',
				category: 'Storage',
				author: 'Community / Redis',
				icon: 'Database',
				accentColor: 'rose',
				nodeType: 'redisCache',
				enabled: true,
				tags: ['redis', 'cache', 'kv'],
				nodeDefinition: {
					title: 'Redis Cache',
					badgeText: 'Redis KV',
					width: 'w-80',
					hasInputHandle: true,
					hasOutputHandle: false,
					outputs: [
						{ id: 'hit', label: 'HIT / OK', color: 'emerald' },
						{ id: 'miss', label: 'MISS / EMPTY', color: 'amber' },
						{ id: 'error', label: 'ERROR', color: 'rose' }
					],
					properties: [
						{
							name: 'redisUrl',
							label: 'Redis URL',
							type: 'password',
							placeholder: 'redis://default:token@host:port'
						},
						{
							name: 'operation',
							label: 'Operation',
							type: 'select',
							defaultValue: 'get',
							options: [
								{ label: 'GET (Retrieve)', value: 'get' },
								{ label: 'SET (Store with TTL)', value: 'set' },
								{ label: 'INCR (Increment Counter)', value: 'incr' },
								{ label: 'DEL (Invalidate)', value: 'del' }
							]
						},
						{
							name: 'keyExpr',
							label: 'Key Expression',
							type: 'text',
							defaultValue: 'payload.cacheKey || "item:" + payload.id'
						},
						{ name: 'ttlSeconds', label: 'TTL in Seconds', type: 'number', defaultValue: 300 }
					],
					defaultData: { operation: 'get', keyExpr: 'payload.cacheKey', ttlSeconds: 300 }
				}
			}
		},
		{
			name: 'GitHub Event Dispatcher',
			icon: 'Github',
			category: 'DevOps',
			accentColor: 'slate',
			version: '1.0.0',
			desc: 'Trigger GitHub repository dispatches and webhook actions to automate CI/CD.',
			tags: ['github', 'devops', 'cicd'],
			manifest: {
				id: 'ext_github_dispatcher',
				name: 'GitHub Event Dispatcher',
				version: '1.0.0',
				description: 'Trigger GitHub repository dispatches and webhook actions to automate CI/CD.',
				category: 'DevOps',
				author: 'Community / GitHub',
				icon: 'Github',
				accentColor: 'slate',
				nodeType: 'githubDispatch',
				enabled: true,
				tags: ['github', 'devops', 'cicd'],
				nodeDefinition: {
					title: 'GitHub Dispatcher',
					badgeText: 'GitHub API',
					width: 'w-80',
					hasInputHandle: true,
					hasOutputHandle: false,
					outputs: [
						{ id: 'success', label: 'DISPATCHED', color: 'emerald' },
						{ id: 'error', label: 'FAILED', color: 'rose' }
					],
					properties: [
						{
							name: 'token',
							label: 'Personal Access Token',
							type: 'password',
							placeholder: 'ghp_...'
						},
						{ name: 'owner', label: 'Repo Owner', type: 'text', placeholder: 'octocat' },
						{ name: 'repo', label: 'Repository Name', type: 'text', placeholder: 'my-project' },
						{ name: 'eventType', label: 'Event Type', type: 'text', defaultValue: 'nodeflow_event' }
					],
					defaultData: { eventType: 'nodeflow_event' }
				}
			}
		}
	];

	// Filtered list
	const filteredExtensions = $derived.by(() => {
		return extensionsList.filter((ext) => {
			// Category filter
			if (
				selectedCategory !== 'ALL' &&
				ext.category.toLowerCase() !== selectedCategory.toLowerCase()
			) {
				return false;
			}
			// Status filter
			if (selectedStatus === 'ENABLED' && !ext.enabled) return false;
			if (selectedStatus === 'DISABLED' && ext.enabled) return false;
			if (selectedStatus === 'BUILTIN' && !ext.isBuiltIn) return false;
			if (selectedStatus === 'COMMUNITY' && ext.isBuiltIn) return false;

			// Search query
			if (searchQuery.trim()) {
				const q = searchQuery.toLowerCase().trim();
				const matchName = ext.name.toLowerCase().includes(q);
				const matchDesc = ext.description.toLowerCase().includes(q);
				const matchAuthor = ext.author.toLowerCase().includes(q);
				const matchCategory = ext.category.toLowerCase().includes(q);
				const matchTag = ext.tags?.some((t: string) => t.toLowerCase().includes(q));
				return matchName || matchDesc || matchAuthor || matchCategory || matchTag;
			}
			return true;
		});
	});

	// Metrics
	const totalCount = $derived(extensionsList.length);
	const enabledCount = $derived(extensionsList.filter((e: ExtensionPackage) => e.enabled).length);
	const builtInCount = $derived(extensionsList.filter((e: ExtensionPackage) => e.isBuiltIn).length);
	const communityCount = $derived(
		extensionsList.filter((e: ExtensionPackage) => !e.isBuiltIn).length
	);

	const allCategories = $derived.by(() => {
		const set = new Set<string>();
		for (const ext of extensionsList) {
			if (ext.category) set.add(ext.category);
		}
		return Array.from(set).sort();
	});

	function getAccentBadgeClass(color: ExtensionAccentColor): string {
		switch (color) {
			case 'emerald':
				return 'bg-emerald-500/10 text-emerald-400 border-emerald-500/30';
			case 'indigo':
				return 'bg-indigo-500/10 text-indigo-400 border-indigo-500/30';
			case 'purple':
				return 'bg-purple-500/10 text-purple-400 border-purple-500/30';
			case 'amber':
				return 'bg-amber-500/10 text-amber-400 border-amber-500/30';
			case 'rose':
				return 'bg-rose-500/10 text-rose-400 border-rose-500/30';
			case 'teal':
				return 'bg-teal-500/10 text-teal-400 border-teal-500/30';
			case 'cyan':
				return 'bg-cyan-500/10 text-cyan-400 border-cyan-500/30';
			case 'yellow':
				return 'bg-yellow-500/10 text-yellow-400 border-yellow-500/30';
			case 'slate':
				return 'bg-slate-500/10 text-slate-400 border-slate-500/30';
			default:
				return 'bg-blue-500/10 text-blue-400 border-blue-500/30';
		}
	}

	function getAccentIconWrapperClass(color: ExtensionAccentColor): string {
		switch (color) {
			case 'emerald':
				return 'bg-emerald-500/20 text-emerald-400 border-emerald-500/30';
			case 'indigo':
				return 'bg-indigo-500/20 text-indigo-400 border-indigo-500/30';
			case 'purple':
				return 'bg-purple-500/20 text-purple-400 border-purple-500/30';
			case 'amber':
				return 'bg-amber-500/20 text-amber-400 border-amber-500/30';
			case 'rose':
				return 'bg-rose-500/20 text-rose-400 border-rose-500/30';
			case 'teal':
				return 'bg-teal-500/20 text-teal-400 border-teal-500/30';
			case 'cyan':
				return 'bg-cyan-500/20 text-cyan-400 border-cyan-500/30';
			case 'yellow':
				return 'bg-yellow-500/20 text-yellow-400 border-yellow-500/30';
			case 'slate':
				return 'bg-slate-800 text-slate-300 border-slate-700';
			default:
				return 'bg-blue-500/20 text-blue-400 border-blue-500/30';
		}
	}
</script>

<svelte:head>
	<title>Extensions & Plugins — Nodeflow</title>
</svelte:head>

<div class="flex min-h-screen flex-col bg-slate-950 text-slate-100 select-none">
	<!-- Top Navigation -->
	<header
		class="sticky top-0 z-30 flex h-16 w-full items-center justify-between border-b border-slate-800 bg-slate-950/90 px-6 backdrop-blur-xl"
	>
		<!-- Left: Brand & Links -->
		<div class="flex items-center gap-4">
			<a
				href="/"
				class="group flex items-center gap-1.5 rounded-lg border border-slate-800 bg-slate-900/90 px-2.5 py-1 text-xs font-semibold text-slate-300 transition hover:border-slate-700 hover:bg-slate-800 hover:text-white"
				title="Back to All Nodeflows"
			>
				<ArrowLeft
					class="h-3.5 w-3.5 text-slate-400 transition-transform group-hover:-translate-x-0.5"
				/>
				<span>Nodeflows</span>
			</a>

			<div class="flex items-center gap-3">
				<div
					class="flex h-10 w-10 items-center justify-center rounded-xl bg-gradient-to-tr from-blue-600 via-indigo-500 to-purple-500 text-white shadow-lg shadow-indigo-500/25"
				>
					<Blocks class="h-5 w-5" />
				</div>
				<div>
					<div class="flex items-center gap-2">
						<h1 class="text-base font-bold tracking-tight text-white">Nodeflow Extensions</h1>
						<span
							class="rounded-full border border-blue-500/20 bg-blue-500/10 px-2 py-0.5 text-[10px] font-semibold text-blue-400"
						>
							v0.5.2
						</span>
					</div>
					<p class="text-xs text-slate-400">
						Import, create & manage custom nodes for your nodeflows
					</p>
				</div>
			</div>
		</div>

		<!-- Center: Navigation Tabs -->
		<div
			class="hidden items-center gap-1 rounded-xl border border-slate-800 bg-slate-900/60 p-1 md:flex"
		>
			<a
				href="/"
				class="rounded-lg px-3 py-1 text-xs font-medium text-slate-400 transition hover:text-white"
			>
				Nodeflows
			</a>
			<a
				href="/users"
				class="rounded-lg px-3 py-1 text-xs font-medium text-slate-400 transition hover:text-white"
			>
				Project Users
			</a>
			<a
				href="/extensions"
				class="rounded-lg bg-slate-800 px-3 py-1 text-xs font-semibold text-white shadow-sm"
			>
				Extensions
			</a>
			<a
				href="/settings"
				class="rounded-lg px-3 py-1 text-xs font-medium text-slate-400 transition hover:text-white"
			>
				Settings
			</a>
		</div>

		<!-- Right: Actions -->
		<div class="flex items-center gap-2.5">
			<button
				type="button"
				onclick={handleResetDefaults}
				class="rounded-lg border border-slate-800 bg-slate-900/80 p-2 text-slate-400 transition hover:border-slate-700 hover:bg-slate-800 hover:text-slate-200"
				title="Reset Extensions to Defaults"
			>
				<RotateCcw class="h-3.5 w-3.5" />
			</button>

			<a
				href="/api/extensions/export"
				download
				class="flex items-center gap-1.5 rounded-lg border border-slate-800 bg-slate-900/80 px-3 py-2 text-xs font-medium text-slate-300 transition hover:border-slate-700 hover:bg-slate-800 hover:text-white"
				title="Export all extensions as a JSON bundle"
			>
				<Download class="h-3.5 w-3.5" />
				<span class="hidden sm:inline">Export All</span>
			</a>

			<button
				type="button"
				onclick={() => {
					isImportOpen = true;
					importError = null;
					activeImportTab = 'registry';
				}}
				class="flex items-center gap-2 rounded-xl bg-gradient-to-r from-blue-600 to-indigo-600 px-4 py-2 text-xs font-semibold text-white shadow-lg shadow-blue-950/60 transition hover:from-blue-500 hover:to-indigo-500 active:scale-95"
			>
				<Plus class="h-4 w-4" />
				<span>Import Extension</span>
			</button>
		</div>
	</header>

	<!-- Main Content Area -->
	<main class="mx-auto w-full max-w-7xl flex-1 space-y-6 px-6 py-6">
		<!-- Toast Notification -->
		{#if toastMessage}
			<div
				class="animate-in fade-in slide-in-from-bottom-3 fixed right-6 bottom-6 z-50 flex items-center gap-2 rounded-xl border px-4 py-3 shadow-2xl backdrop-blur-xl duration-200 {toastType ===
				'success'
					? 'border-emerald-500/40 bg-emerald-950/90 text-emerald-200'
					: 'border-rose-500/40 bg-rose-950/90 text-rose-200'}"
			>
				{#if toastType === 'success'}
					<Check class="h-4 w-4 text-emerald-400" />
				{:else}
					<Info class="h-4 w-4 text-rose-400" />
				{/if}
				<span class="text-xs font-medium">{toastMessage}</span>
			</div>
		{/if}

		<!-- Overview Metrics Bar -->
		<div class="grid grid-cols-2 gap-4 md:grid-cols-4">
			<div class="rounded-xl border border-slate-800/80 bg-slate-900/50 p-4 backdrop-blur-sm">
				<div class="flex items-center justify-between text-xs text-slate-400">
					<span>Total Extensions</span>
					<Blocks class="h-4 w-4 text-blue-400" />
				</div>
				<div class="mt-2 text-2xl font-bold tracking-tight text-white">{totalCount}</div>
				<div class="mt-1 text-[11px] text-slate-500">Registered across system</div>
			</div>

			<div class="rounded-xl border border-slate-800/80 bg-slate-900/50 p-4 backdrop-blur-sm">
				<div class="flex items-center justify-between text-xs text-slate-400">
					<span>Active in Canvas</span>
					<Sparkles class="h-4 w-4 text-emerald-400" />
				</div>
				<div class="mt-2 text-2xl font-bold tracking-tight text-emerald-400">{enabledCount}</div>
				<div class="mt-1 text-[11px] text-slate-500">Available in editor palette</div>
			</div>

			<div class="rounded-xl border border-slate-800/80 bg-slate-900/50 p-4 backdrop-blur-sm">
				<div class="flex items-center justify-between text-xs text-slate-400">
					<span>Built-in Nodes</span>
					<ShieldCheck class="h-4 w-4 text-indigo-400" />
				</div>
				<div class="mt-2 text-2xl font-bold tracking-tight text-white">{builtInCount}</div>
				<div class="mt-1 text-[11px] text-slate-500">Core Nodeflow engine nodes</div>
			</div>

			<div class="rounded-xl border border-slate-800/80 bg-slate-900/50 p-4 backdrop-blur-sm">
				<div class="flex items-center justify-between text-xs text-slate-400">
					<span>Community & Custom</span>
					<Layers class="h-4 w-4 text-purple-400" />
				</div>
				<div class="mt-2 text-2xl font-bold tracking-tight text-purple-400">{communityCount}</div>
				<div class="mt-1 text-[11px] text-slate-500">Imported / third-party extensions</div>
			</div>
		</div>

		<!-- Search & Category Filters -->
		<div class="space-y-3">
			<div class="flex flex-col items-stretch justify-between gap-3 sm:flex-row sm:items-center">
				<!-- Search Input -->
				<div class="relative max-w-md flex-1">
					<Search class="absolute top-2.5 left-3 h-4 w-4 text-slate-500" />
					<input
						type="text"
						placeholder="Search extensions by name, author, tag, or description..."
						class="w-full rounded-xl border border-slate-800 bg-slate-900/80 py-2 pr-4 pl-9 text-xs text-slate-200 placeholder-slate-500 transition focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50 focus:outline-none"
						bind:value={searchQuery}
					/>
					{#if searchQuery}
						<button
							type="button"
							class="absolute top-2.5 right-3 text-slate-500 hover:text-slate-300"
							onclick={() => (searchQuery = '')}
						>
							<X class="h-4 w-4" />
						</button>
					{/if}
				</div>

				<!-- Source / Status Pill Filter -->
				<div
					class="flex items-center gap-1 self-start overflow-x-auto rounded-xl border border-slate-800 bg-slate-900/80 p-1 sm:self-auto"
				>
					<button
						type="button"
						onclick={() => (selectedStatus = 'ALL')}
						class="rounded-lg px-2.5 py-1 text-[11px] font-medium transition {selectedStatus ===
						'ALL'
							? 'bg-slate-800 font-semibold text-white shadow-sm'
							: 'text-slate-400 hover:text-white'}"
					>
						All
					</button>
					<button
						type="button"
						onclick={() => (selectedStatus = 'ENABLED')}
						class="rounded-lg px-2.5 py-1 text-[11px] font-medium transition {selectedStatus ===
						'ENABLED'
							? 'bg-slate-800 font-semibold text-emerald-400 shadow-sm'
							: 'text-slate-400 hover:text-white'}"
					>
						Enabled ({enabledCount})
					</button>
					<button
						type="button"
						onclick={() => (selectedStatus = 'COMMUNITY')}
						class="rounded-lg px-2.5 py-1 text-[11px] font-medium transition {selectedStatus ===
						'COMMUNITY'
							? 'bg-slate-800 font-semibold text-purple-400 shadow-sm'
							: 'text-slate-400 hover:text-white'}"
					>
						Community ({communityCount})
					</button>
					<button
						type="button"
						onclick={() => (selectedStatus = 'BUILTIN')}
						class="rounded-lg px-2.5 py-1 text-[11px] font-medium transition {selectedStatus ===
						'BUILTIN'
							? 'bg-slate-800 font-semibold text-blue-400 shadow-sm'
							: 'text-slate-400 hover:text-white'}"
					>
						Built-in ({builtInCount})
					</button>
				</div>
			</div>

			<!-- Category Pills -->
			<div class="flex items-center gap-1.5 overflow-x-auto pb-1 text-xs">
				<button
					type="button"
					onclick={() => (selectedCategory = 'ALL')}
					class="shrink-0 rounded-lg border px-3 py-1 font-medium transition {selectedCategory ===
					'ALL'
						? 'border-indigo-500/60 bg-indigo-500/20 text-indigo-300'
						: 'border-slate-800/80 bg-slate-900/60 text-slate-400 hover:border-slate-700 hover:text-slate-200'}"
				>
					All Categories
				</button>
				{#each allCategories as cat}
					<button
						type="button"
						onclick={() => (selectedCategory = cat)}
						class="shrink-0 rounded-lg border px-3 py-1 font-medium transition {selectedCategory ===
						cat
							? 'border-indigo-500/60 bg-indigo-500/20 text-indigo-300'
							: 'border-slate-800/80 bg-slate-900/60 text-slate-400 hover:border-slate-700 hover:text-slate-200'}"
					>
						{cat}
					</button>
				{/each}
			</div>
		</div>

		<!-- Extensions Grid -->
		{#if filteredExtensions.length === 0}
			<div
				class="space-y-3 rounded-2xl border border-dashed border-slate-800 bg-slate-900/30 p-12 text-center"
			>
				<div
					class="mx-auto flex h-12 w-12 items-center justify-center rounded-2xl bg-slate-800 text-slate-500"
				>
					<Blocks class="h-6 w-6" />
				</div>
				<h3 class="text-sm font-semibold text-slate-200">No extensions found</h3>
				<p class="mx-auto max-w-sm text-xs text-slate-500">
					No extensions matched your filter or search query. Try clearing filters or import a new
					extension.
				</p>
				<button
					type="button"
					onclick={() => {
						searchQuery = '';
						selectedCategory = 'ALL';
						selectedStatus = 'ALL';
					}}
					class="rounded-lg border border-slate-800 bg-slate-900 px-3 py-1.5 text-xs text-slate-300 transition hover:bg-slate-800"
				>
					Clear Filters
				</button>
			</div>
		{:else}
			<div class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
				{#each filteredExtensions as ext (ext.id)}
					<div
						class="flex flex-col justify-between rounded-xl border bg-slate-900/70 p-4 backdrop-blur-sm transition-all duration-200 {ext.enabled
							? 'border-slate-800 shadow-md hover:border-slate-700'
							: 'border-slate-900 bg-slate-950/40 opacity-70'}"
					>
						<!-- Card Top -->
						<div class="space-y-3">
							<!-- Header: Icon, Name, Category & Toggle -->
							<div class="flex items-start justify-between gap-3">
								<div class="flex items-start gap-2.5">
									<div
										class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl border {getAccentIconWrapperClass(
											ext.accentColor
										)}"
									>
										<DynamicIcon name={ext.icon} class="h-5 w-5" />
									</div>

									<div>
										<div class="flex flex-wrap items-center gap-1.5">
											<h3 class="text-sm leading-tight font-bold text-white">
												{ext.name}
											</h3>
											<span class="font-mono text-[10px] text-slate-500">
												v{ext.version}
											</span>
										</div>

										<div class="mt-1 flex items-center gap-1.5">
											<span
												class="py-0.2 rounded border px-1.5 text-[9px] font-semibold {getAccentBadgeClass(
													ext.accentColor
												)}"
											>
												{ext.category}
											</span>

											{#if ext.isBuiltIn}
												<span
													class="py-0.2 rounded border border-slate-700/60 bg-slate-800/90 px-1.5 text-[9px] font-medium text-slate-400"
												>
													Built-in
												</span>
											{:else}
												<span
													class="py-0.2 rounded border border-purple-500/20 bg-purple-500/10 px-1.5 text-[9px] font-medium text-purple-400"
												>
													Community
												</span>
											{/if}
										</div>
									</div>
								</div>

								<!-- Enabled Toggle Switch -->
								<button
									type="button"
									role="switch"
									aria-checked={ext.enabled}
									onclick={() => toggleExtension(ext)}
									class="relative inline-flex h-5 w-9 shrink-0 cursor-pointer rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none {ext.enabled
										? 'bg-emerald-500'
										: 'bg-slate-800'}"
									title={ext.enabled ? 'Click to disable from canvas' : 'Click to enable in canvas'}
								>
									<span
										class="pointer-events-none inline-block h-4 w-4 transform rounded-full bg-white shadow ring-0 transition duration-200 ease-in-out {ext.enabled
											? 'translate-x-4'
											: 'translate-x-0'}"
									></span>
								</button>
							</div>

							<!-- Description -->
							<p class="line-clamp-2 text-xs leading-relaxed text-slate-400">
								{ext.description}
							</p>

							<!-- Node Spec Summary (Handles & Branches) -->
							<div
								class="border-slate-850 space-y-1.5 rounded-lg border bg-slate-950/60 p-2.5 text-[11px]"
							>
								<div class="flex items-center justify-between text-[10px] text-slate-400">
									<span>Outputs & Branches:</span>
									<span class="font-mono text-slate-500">
										{ext.nodeDefinition.outputs?.length || 1} branch{(ext.nodeDefinition.outputs
											?.length || 1) > 1
											? 'es'
											: ''}
									</span>
								</div>

								{#if ext.nodeDefinition.outputs && ext.nodeDefinition.outputs.length > 0}
									<div class="flex flex-wrap items-center gap-1.5">
										{#each ext.nodeDefinition.outputs as out}
											<div
												class="flex items-center gap-1 rounded border border-slate-800 bg-slate-900 px-1.5 py-0.5 font-mono text-[9px] font-bold"
											>
												<span
													class="h-1.5 w-1.5 rounded-full {out.color === 'emerald'
														? 'bg-emerald-400'
														: out.color === 'rose'
															? 'bg-rose-400'
															: out.color === 'amber'
																? 'bg-amber-400'
																: 'bg-blue-400'}"
												></span>
												<span class="text-slate-300">{out.label || out.id}</span>
											</div>
										{/each}
									</div>
								{:else}
									<div class="font-mono text-[10px] text-slate-500">
										Default single downstream output
									</div>
								{/if}
							</div>

							<!-- Tags -->
							{#if ext.tags && ext.tags.length > 0}
								<div class="flex flex-wrap items-center gap-1">
									{#each ext.tags as tag}
										<span class="font-mono text-[9px] text-slate-500">
											#{tag}
										</span>
									{/each}
								</div>
							{/if}
						</div>

						<!-- Card Bottom Toolbar -->
						<div
							class="mt-4 flex items-center justify-between border-t border-slate-800/80 pt-3 text-xs"
						>
							<span class="max-w-[120px] truncate text-[10px] text-slate-500">
								by {ext.author}
							</span>

							<div class="flex items-center gap-1.5">
								<button
									type="button"
									onclick={() => openInspect(ext)}
									class="flex items-center gap-1 rounded-lg border border-slate-800 bg-slate-900 px-2 py-1 text-[11px] font-medium text-slate-300 transition hover:bg-slate-800 hover:text-white"
									title="Inspect Manifest & Schema"
								>
									<Eye class="h-3 w-3" />
									<span>Details</span>
								</button>

								<a
									href="/api/extensions/export?id={encodeURIComponent(ext.id)}"
									download
									class="rounded-lg border border-slate-800 bg-slate-900 p-1 text-slate-400 transition hover:bg-slate-800 hover:text-slate-200"
									title="Download JSON Manifest"
								>
									<Download class="h-3.5 w-3.5" />
								</a>

								{#if !ext.isBuiltIn}
									<button
										type="button"
										onclick={() => {
											extToDelete = ext;
											isDeleteOpen = true;
										}}
										class="rounded-lg border border-slate-800 bg-slate-900 p-1 text-slate-400 transition hover:border-rose-500/40 hover:bg-rose-500/10 hover:text-rose-400"
										title="Uninstall Extension"
									>
										<Trash2 class="h-3.5 w-3.5" />
									</button>
								{/if}
							</div>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</main>
</div>

<!-- ============================================================== -->
<!-- IMPORT EXTENSION MODAL                                         -->
<!-- ============================================================== -->
{#if isImportOpen}
	<div
		class="animate-in fade-in fixed inset-0 z-50 flex items-center justify-center bg-black/75 p-4 backdrop-blur-sm duration-150"
	>
		<div
			class="flex max-h-[90vh] w-full max-w-2xl flex-col overflow-hidden rounded-2xl border border-slate-800 bg-slate-900 shadow-2xl"
		>
			<!-- Header -->
			<div class="flex items-center justify-between border-b border-slate-800 px-6 py-4">
				<div class="flex items-center gap-2.5">
					<div
						class="flex h-8 w-8 items-center justify-center rounded-lg bg-indigo-500/20 text-indigo-400"
					>
						<Plus class="h-4 w-4" />
					</div>
					<div>
						<h2 class="text-sm font-bold text-white">Import Extension</h2>
						<p class="text-[11px] text-slate-400">
							Add custom nodes from manifest files, URLs, or community registry
						</p>
					</div>
				</div>
				<button
					type="button"
					class="rounded-lg p-1 text-slate-400 transition hover:bg-slate-800 hover:text-white"
					onclick={() => (isImportOpen = false)}
				>
					<X class="h-4 w-4" />
				</button>
			</div>

			<!-- Navigation Tabs -->
			<div
				class="flex gap-2 border-b border-slate-800 bg-slate-950/60 px-6 pt-2 text-xs font-medium"
			>
				<button
					type="button"
					onclick={() => (activeImportTab = 'registry')}
					class="border-b-2 px-3 pb-2.5 transition {activeImportTab === 'registry'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					Curated Registry
				</button>
				<button
					type="button"
					onclick={() => (activeImportTab = 'upload')}
					class="border-b-2 px-3 pb-2.5 transition {activeImportTab === 'upload'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					Upload JSON File
				</button>
				<button
					type="button"
					onclick={() => (activeImportTab = 'url')}
					class="border-b-2 px-3 pb-2.5 transition {activeImportTab === 'url'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					From URL / CDN
				</button>
				<button
					type="button"
					onclick={() => {
						activeImportTab = 'editor';
						if (!importJsonText) importJsonText = sampleManifestTemplate;
					}}
					class="border-b-2 px-3 pb-2.5 transition {activeImportTab === 'editor'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					Manifest JSON Editor
				</button>
			</div>

			<!-- Tab Content Body -->
			<div class="flex-1 space-y-4 overflow-y-auto p-6">
				{#if importError}
					<div
						class="flex items-center justify-between rounded-xl border border-rose-500/40 bg-rose-500/10 p-3 text-xs text-rose-300"
					>
						<span>{importError}</span>
						<button type="button" onclick={() => (importError = null)}>
							<X class="h-3.5 w-3.5 text-rose-400" />
						</button>
					</div>
				{/if}

				<!-- TAB: REGISTRY -->
				{#if activeImportTab === 'registry'}
					<div class="space-y-3">
						<div class="text-xs text-slate-400">
							Select a popular community integration below to install its node manifest into your
							Nodeflow workspace:
						</div>

						<div class="grid grid-cols-1 gap-3 sm:grid-cols-2">
							{#each curatedCatalog as item}
								<div
									class="flex flex-col justify-between gap-3 rounded-xl border border-slate-800 bg-slate-950/70 p-3.5"
								>
									<div class="space-y-1.5">
										<div class="flex items-center justify-between">
											<div class="flex items-center gap-2">
												<div
													class="flex h-7 w-7 items-center justify-center rounded-lg {getAccentIconWrapperClass(
														item.accentColor as any
													)}"
												>
													<DynamicIcon name={item.icon} class="h-4 w-4" />
												</div>
												<span class="text-xs font-bold text-white">{item.name}</span>
											</div>
											<span class="font-mono text-[9px] text-slate-500">v{item.version}</span>
										</div>
										<p class="text-[11px] leading-snug text-slate-400">
											{item.desc}
										</p>
									</div>

									<div class="border-slate-850 flex items-center justify-between border-t pt-2">
										<span
											class="text-[9px] font-semibold {getAccentBadgeClass(
												item.accentColor as any
											)} rounded border px-1.5 py-0.5"
										>
											{item.category}
										</span>
										<button
											type="button"
											disabled={isImporting}
											onclick={() => installCurated(item.manifest)}
											class="rounded-lg bg-indigo-600 px-3 py-1 text-[11px] font-semibold text-white transition hover:bg-indigo-500 active:scale-95 disabled:opacity-50"
										>
											{isImporting ? 'Installing...' : 'Install Node'}
										</button>
									</div>
								</div>
							{/each}
						</div>
					</div>

					<!-- TAB: UPLOAD JSON -->
				{:else if activeImportTab === 'upload'}
					<div class="space-y-4">
						<label
							class="group flex cursor-pointer flex-col items-center justify-center rounded-2xl border-2 border-dashed border-slate-800 bg-slate-950/50 p-8 transition hover:border-indigo-500/60"
						>
							<UploadCloud
								class="h-10 w-10 text-slate-500 transition-all group-hover:scale-110 group-hover:text-indigo-400"
							/>
							<span class="mt-3 text-xs font-bold text-slate-200"
								>Click to upload or drag & drop</span
							>
							<span class="mt-1 text-[11px] text-slate-500"
								>Select an extension manifest file (.json)</span
							>
							<input
								type="file"
								accept=".json,application/json"
								class="hidden"
								onchange={handleFileUpload}
							/>
						</label>

						<div
							class="flex items-start gap-2 rounded-lg border border-slate-800/80 bg-slate-950 p-3 text-[11px] text-slate-400"
						>
							<Info class="mt-0.5 h-4 w-4 shrink-0 text-blue-400" />
							<span
								>The uploaded file will be verified against the Nodeflow Extension schema. You can
								review and edit before finalizing.</span
							>
						</div>
					</div>

					<!-- TAB: URL / CDN -->
				{:else if activeImportTab === 'url'}
					<div class="space-y-3">
						<div class="space-y-1">
							<label for="import-url-input" class="text-xs font-medium text-slate-400"
								>Remote Manifest URL</label
							>
							<input
								id="import-url-input"
								type="url"
								placeholder="https://raw.githubusercontent.com/org/repo/main/nodeflow-extension.json"
								class="w-full rounded-xl border border-slate-800 bg-slate-950 px-3 py-2 font-mono text-xs text-slate-200 placeholder-slate-600 focus:border-indigo-500 focus:outline-none"
								bind:value={importUrl}
							/>
						</div>

						<p class="text-[11px] leading-relaxed text-slate-500">
							Provide a direct HTTP/HTTPS link to an extension manifest JSON file on GitHub, unpkg,
							jsDelivr, or any web endpoint.
						</p>

						<div class="flex justify-end pt-2">
							<button
								type="button"
								disabled={isImporting || !importUrl.trim()}
								onclick={submitImportUrl}
								class="flex items-center gap-2 rounded-xl bg-indigo-600 px-4 py-2 text-xs font-semibold text-white transition hover:bg-indigo-500 active:scale-95 disabled:opacity-50"
							>
								{#if isImporting}
									<span
										class="inline-block h-3.5 w-3.5 animate-spin rounded-full border-2 border-white border-t-transparent"
									></span>
									<span>Fetching & Installing...</span>
								{:else}
									<Download class="h-4 w-4" />
									<span>Fetch & Install</span>
								{/if}
							</button>
						</div>
					</div>

					<!-- TAB: MANIFEST EDITOR -->
				{:else if activeImportTab === 'editor'}
					<div class="space-y-3">
						<div class="flex items-center justify-between text-xs text-slate-400">
							<span>Manifest JSON Editor</span>
							<button
								type="button"
								onclick={() => (importJsonText = sampleManifestTemplate)}
								class="text-[11px] text-indigo-400 hover:text-indigo-300"
							>
								Reset to Sample Template
							</button>
						</div>

						<div
							class="overflow-hidden rounded-xl border border-slate-800 bg-slate-950 font-mono text-xs"
						>
							<textarea
								rows="12"
								class="w-full resize-y bg-transparent p-3 font-mono leading-relaxed text-slate-200 selection:bg-indigo-900/60 focus:outline-none"
								placeholder="Paste extension JSON manifest..."
								bind:value={importJsonText}
								spellcheck="false"></textarea>
						</div>

						<div class="flex justify-end pt-2">
							<button
								type="button"
								disabled={isImporting || !importJsonText.trim()}
								onclick={submitImportJson}
								class="flex items-center gap-2 rounded-xl bg-gradient-to-r from-blue-600 to-indigo-600 px-4 py-2 text-xs font-semibold text-white transition hover:from-blue-500 hover:to-indigo-500 active:scale-95 disabled:opacity-50"
							>
								{#if isImporting}
									<span
										class="inline-block h-3.5 w-3.5 animate-spin rounded-full border-2 border-white border-t-transparent"
									></span>
									<span>Validating & Installing...</span>
								{:else}
									<Check class="h-4 w-4" />
									<span>Validate & Install Extension</span>
								{/if}
							</button>
						</div>
					</div>
				{/if}
			</div>
		</div>
	</div>
{/if}

<!-- ============================================================== -->
<!-- DETAILS / INSPECT MODAL                                        -->
<!-- ============================================================== -->
{#if isInspectOpen && inspectedExt}
	<div
		class="animate-in fade-in fixed inset-0 z-50 flex items-center justify-center bg-black/75 p-4 backdrop-blur-sm duration-150"
	>
		<div
			class="flex max-h-[90vh] w-full max-w-2xl flex-col overflow-hidden rounded-2xl border border-slate-800 bg-slate-900 shadow-2xl"
		>
			<!-- Header -->
			<div class="flex items-center justify-between border-b border-slate-800 px-6 py-4">
				<div class="flex items-center gap-3">
					<div
						class="flex h-9 w-9 items-center justify-center rounded-xl {getAccentIconWrapperClass(
							inspectedExt.accentColor
						)}"
					>
						<DynamicIcon name={inspectedExt.icon} class="h-5 w-5" />
					</div>
					<div>
						<div class="flex items-center gap-2">
							<h2 class="text-sm font-bold text-white">{inspectedExt.name}</h2>
							<span class="font-mono text-[10px] text-slate-400">v{inspectedExt.version}</span>
						</div>
						<p class="text-[11px] text-slate-400">
							{inspectedExt.category} • by {inspectedExt.author}
						</p>
					</div>
				</div>

				<button
					type="button"
					class="rounded-lg p-1 text-slate-400 transition hover:bg-slate-800 hover:text-white"
					onclick={() => (isInspectOpen = false)}
				>
					<X class="h-4 w-4" />
				</button>
			</div>

			<!-- Tabs -->
			<div
				class="flex gap-2 border-b border-slate-800 bg-slate-950/60 px-6 pt-2 text-xs font-medium"
			>
				<button
					type="button"
					onclick={() => (inspectActiveTab = 'overview')}
					class="border-b-2 px-3 pb-2.5 transition {inspectActiveTab === 'overview'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					Overview
				</button>
				<button
					type="button"
					onclick={() => (inspectActiveTab = 'properties')}
					class="border-b-2 px-3 pb-2.5 transition {inspectActiveTab === 'properties'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					Node Properties ({inspectedExt.nodeDefinition.properties?.length || 0})
				</button>
				<button
					type="button"
					onclick={() => (inspectActiveTab = 'manifest')}
					class="border-b-2 px-3 pb-2.5 transition {inspectActiveTab === 'manifest'
						? 'border-indigo-500 font-semibold text-white'
						: 'border-transparent text-slate-400 hover:text-slate-200'}"
				>
					Raw Manifest JSON
				</button>
			</div>

			<!-- Body -->
			<div class="flex-1 space-y-4 overflow-y-auto p-6 text-xs">
				{#if inspectActiveTab === 'overview'}
					<div class="space-y-4">
						<div class="space-y-1">
							<span class="text-[11px] font-semibold text-slate-400">Description</span>
							<p
								class="rounded-xl border border-slate-800 bg-slate-950/60 p-3 text-xs leading-relaxed text-slate-300"
							>
								{inspectedExt.description}
							</p>
						</div>

						<div class="grid grid-cols-2 gap-3">
							<div class="space-y-1 rounded-xl border border-slate-800 bg-slate-950/50 p-3">
								<span class="text-[10px] text-slate-500">Nodeflow Type Identifier</span>
								<div class="font-mono text-xs text-indigo-400">{inspectedExt.nodeType}</div>
							</div>
							<div class="space-y-1 rounded-xl border border-slate-800 bg-slate-950/50 p-3">
								<span class="text-[10px] text-slate-500">Default Width</span>
								<div class="font-mono text-xs text-slate-300">
									{inspectedExt.nodeDefinition.width || 'w-80'}
								</div>
							</div>
						</div>

						<!-- Output Handles Specification -->
						<div class="space-y-2">
							<span class="text-[11px] font-semibold text-slate-400"
								>Output Handles & Branch Logic</span
							>
							<div class="space-y-2 rounded-xl border border-slate-800 bg-slate-950/70 p-3">
								{#if inspectedExt.nodeDefinition.outputs && inspectedExt.nodeDefinition.outputs.length > 0}
									<div class="space-y-1.5">
										{#each inspectedExt.nodeDefinition.outputs as out}
											<div
												class="border-slate-850 flex items-center justify-between border-b py-1 text-xs last:border-b-0"
											>
												<div class="flex items-center gap-2">
													<span
														class="h-2 w-2 rounded-full {out.color === 'emerald'
															? 'bg-emerald-400'
															: out.color === 'rose'
																? 'bg-rose-400'
																: 'bg-blue-400'}"
													></span>
													<span class="font-mono font-bold text-slate-200">{out.id}</span>
												</div>
												<span class="text-[11px] text-slate-400">{out.label}</span>
											</div>
										{/each}
									</div>
								{:else}
									<p class="text-xs text-slate-400">
										Single bottom output handle (sequential flow).
									</p>
								{/if}
							</div>
						</div>

						{#if inspectedExt.website}
							<div class="pt-2">
								<a
									href={inspectedExt.website}
									target="_blank"
									rel="noreferrer"
									class="inline-flex items-center gap-1.5 text-xs text-indigo-400 hover:text-indigo-300 hover:underline"
								>
									<span>Official Documentation & API Reference</span>
									<ExternalLink class="h-3.5 w-3.5" />
								</a>
							</div>
						{/if}
					</div>
				{:else if inspectActiveTab === 'properties'}
					<div class="space-y-3">
						{#if inspectedExt.nodeDefinition.properties && inspectedExt.nodeDefinition.properties.length > 0}
							<div
								class="divide-y divide-slate-800 rounded-xl border border-slate-800 bg-slate-950/70"
							>
								{#each inspectedExt.nodeDefinition.properties as prop}
									<div class="space-y-1 p-3.5">
										<div class="flex items-center justify-between">
											<span class="font-mono text-xs font-bold text-indigo-300">{prop.name}</span>
											<span
												class="rounded bg-slate-800 px-1.5 py-0.5 font-mono text-[10px] text-slate-400"
											>
												{prop.type}
											</span>
										</div>
										<p class="text-xs text-slate-300">{prop.label}</p>
										{#if prop.description}
											<p class="text-[11px] text-slate-500">{prop.description}</p>
										{/if}
										{#if prop.defaultValue !== undefined}
											<div class="font-mono text-[10px] text-slate-400">
												Default: <span class="text-slate-300"
													>{JSON.stringify(prop.defaultValue)}</span
												>
											</div>
										{/if}
									</div>
								{/each}
							</div>
						{:else}
							<div
								class="rounded-xl border border-slate-800 bg-slate-950 p-6 text-center text-xs text-slate-500"
							>
								This extension does not declare custom configurable properties.
							</div>
						{/if}
					</div>
				{:else if inspectActiveTab === 'manifest'}
					<div class="space-y-2">
						<div class="flex items-center justify-between text-xs text-slate-400">
							<span>Full Extension JSON Manifest</span>
							<button
								type="button"
								onclick={copyManifestToClipboard}
								class="flex items-center gap-1 text-[11px] text-indigo-400 hover:text-indigo-300"
							>
								{#if copiedManifest}
									<Check class="h-3 w-3 text-emerald-400" />
									<span class="text-emerald-400">Copied!</span>
								{:else}
									<Copy class="h-3 w-3" />
									<span>Copy JSON</span>
								{/if}
							</button>
						</div>

						<pre
							class="max-h-96 overflow-x-auto rounded-xl border border-slate-800 bg-slate-950 p-4 font-mono text-[11px] leading-relaxed text-slate-300 selection:bg-indigo-900/60">{JSON.stringify(
								inspectedExt,
								null,
								2
							)}</pre>
					</div>
				{/if}
			</div>
		</div>
	</div>
{/if}

<!-- ============================================================== -->
<!-- DELETE CONFIRMATION MODAL                                      -->
<!-- ============================================================== -->
{#if isDeleteOpen && extToDelete}
	<div
		class="animate-in fade-in fixed inset-0 z-50 flex items-center justify-center bg-black/75 p-4 backdrop-blur-sm duration-150"
	>
		<div
			class="w-full max-w-md space-y-4 rounded-2xl border border-slate-800 bg-slate-900 p-6 shadow-2xl"
		>
			<div class="flex items-center gap-3">
				<div
					class="flex h-10 w-10 items-center justify-center rounded-xl bg-rose-500/20 text-rose-400"
				>
					<Trash2 class="h-5 w-5" />
				</div>
				<div>
					<h3 class="text-sm font-bold text-white">Uninstall Extension</h3>
					<p class="text-xs text-slate-400">Are you sure you want to remove this extension?</p>
				</div>
			</div>

			<p
				class="rounded-xl border border-slate-800 bg-slate-950/60 p-3 text-xs leading-relaxed text-slate-300"
			>
				Uninstalling <strong class="text-white">{extToDelete.name}</strong> will remove it from the palette.
				Existing flows that utilize this node type will require the extension to be re-installed.
			</p>

			<div class="flex items-center justify-end gap-2.5 pt-2">
				<button
					type="button"
					onclick={() => {
						isDeleteOpen = false;
						extToDelete = null;
					}}
					class="rounded-xl border border-slate-800 px-4 py-2 text-xs font-semibold text-slate-400 transition hover:bg-slate-800 hover:text-white"
				>
					Cancel
				</button>
				<button
					type="button"
					disabled={isDeleting}
					onclick={confirmDelete}
					class="rounded-xl bg-rose-600 px-4 py-2 text-xs font-semibold text-white transition hover:bg-rose-500 active:scale-95 disabled:opacity-50"
				>
					{isDeleting ? 'Uninstalling...' : 'Uninstall'}
				</button>
			</div>
		</div>
	</div>
{/if}
