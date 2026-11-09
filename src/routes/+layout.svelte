<script>
	// Packagae
	import { registerSW } from 'virtual:pwa-register';
	import { isLoading, locale } from 'svelte-i18n';
	import { page } from '$app/stores';
	import { dev } from '$app/environment';
	import { onMount, setContext } from 'svelte';
	import 'zoomist/css';

	import {
		viewportHeight,
		viewportWidth,
		isMobile,
		mobileMode,
		isPWA
	} from '$lib/store/app-stores';
	import { IDBUpdater } from '$lib/helpers/migrator/IDBUpdater';
	import { storageLocal } from '$lib/helpers/dataAPI/api-localstore';
	import { HOST, KEYWORDS, APP_TITLE, DESCRIPTION } from '$lib/env';
	import { sync } from '$lib/helpers/dataAPI/sync';
	import { autoExport } from '$lib/store/filesystem-store';
	import { mountLocale } from '$lib/helpers/i18n';
	import { mobileDetect } from '$lib/helpers/mobileDetect';
	import { wakeLock } from '$lib/helpers/wakeLock';
	import { syncCustomBanner } from '$lib/helpers/banner-custom';
	// import { initializeDriveAPI } from '$lib/helpers/dataAPI/google-api';
	import '../app.css';

	import Iklan from '$lib/components/Iklan.svelte';
	import Toasts from '$lib/components/Toasts.svelte';
	import Loader from './_index/InitialLoader.svelte';

	const homepageDescription = "原神祈愿（抽卡）模拟器，在网页里体验每一次十连";
	const homepageMetadata = {
		"@context": "https://schema.org",
		"@type": "WebApplication",
		"name": "Genshin Impact Wish Simulator",
		"url": "https://yswish.march7th.cn/",
		"description": "原神祈愿（抽卡）模拟器，在网页里体验每一次十连",
		"applicationCategory": "GameApplication",
		"operatingSystem": "Web",
		"inLanguage": "zh-CN",
		"isPartOf": {
			"@type": "WebSite",
			"@id": "https://march7th.cn/#website",
			"name": "March7th · 三月七工具站",
			"url": "https://march7th.cn/"
		},
		"publisher": {
			"@type": "Organization",
			"@id": "https://march7th.cn/#org",
			"name": "March7th Team",
			"url": "https://march7th.cn/"
		}
	};
	$: isHomepage = $page.url.pathname === '/';
	$: canonicalUrl = "https://yswish.march7th.cn" + $page.url.pathname;
	$: isSharePage = $page.url.pathname === '/screen' || $page.url.pathname.startsWith('/screen/');
	$: pageDescription = isHomepage ? homepageDescription : DESCRIPTION;

	let innerHeight;
	let innerWidth;
	let isBannerLoaded = false;
	let isloaded = false;

	setContext('bannerLoaded', () => (isBannerLoaded = true));
	setContext('loaded', () => (isloaded = true));

	let font = '';
	$: {
		const lc = $locale?.toLowerCase() || '';
		const isSpecial = lc.match(/(cn|ja|kr)/);
		font = isSpecial || lc.includes('th') ? lc.split('-')[0] : 'global';
	}

	$: viewportWidth.set(innerWidth);
	$: viewportHeight.set(innerHeight);

	let directLoad = false;
	let preview = false;

	const checkPath = () => {
		const path = $page.url.pathname.split('/');
		directLoad = !!path[1];
		preview = path[1] === 'screen';

		const validPaths = ['adkey', 'bnlist', 'install', 'privacy-policy', 'screen'];
		const isPathValid = validPaths.includes(path[1].toLowerCase());
		const redirect = path[1] && !isPathValid;
		return redirect;
	};

	const setMobileMode = () => {
		if ($isPWA) return mobileMode.set(true);
		const angle = screen.orientation?.angle || 0;
		const rotate = angle === 90 || angle === 270;
		mobileMode.set(rotate);
	};

	mountLocale();
	onMount(async () => {
		if (checkPath()) return window.location.replace('/');

		const url = new URL(window.location.href);
		const searchParams = new URLSearchParams(url.search);
		isPWA.set(searchParams.get('pwa') === 'true' || !!searchParams.get('pwasc'));

		isMobile.set(mobileDetect() || innerWidth < 601);
		if ($isMobile) setMobileMode();

		window.addEventListener('orientationchange', () => {
			if ($isMobile) setMobileMode();
		});

		storageLocal.initEvent(); //for autoexport
		registerSW(); // Service Worker for faster load
		wakeLock(); // Prevent screen off while open the app
		await IDBUpdater(); // Update site data to the newer Version
		syncCustomBanner(); // Sync Custom Banner
		// initializeDriveAPI(); // Drive API for cloud Sync

		document.addEventListener('storageUpdate', () => sync($autoExport));
		// prevent Righ click (hold on android) on production mode
		if (!dev) document.addEventListener('contextmenu', (e) => e.preventDefault());
	});
</script>

<svelte:window bind:innerHeight bind:innerWidth />

<svelte:head>
	<link rel="canonical" href={canonicalUrl} />
	<meta property="og:url" content={canonicalUrl} />
	<meta property="og:title" content={APP_TITLE} />
	<meta name="description" content={pageDescription} />
	<meta property="og:description" content={pageDescription} />
	<meta name="robots" content={isSharePage ? "noindex,follow" : "index,follow,max-image-preview:large"} />
	{#if isHomepage}
		{@html `<script type="application/ld+json">${JSON.stringify(homepageMetadata).replace(/</g, '\\u003c')}</script>`}
	{/if}

	<meta name="keywords" content={KEYWORDS} />
	<meta property="al:web:url" content={canonicalUrl} />
	<link rel="fluid-icon" href="{HOST}/meta-picture.jpg" title={APP_TITLE} />

	<meta property="og:image" content="{HOST}/meta-picture.jpg" />

	<meta property="twitter:domain" content="yswish.march7th.cn" />
	<meta property="twitter:url" content={canonicalUrl} />
	<meta name="twitter:title" content={APP_TITLE} />
	<meta name="twitter:description" content={pageDescription} />
	<meta name="twitter:image" content="{HOST}/meta-picture.jpg" />

	<link
		rel="preload"
		href="/fonts/optimized_global_web.woff2"
		as="font"
		type="font/woff2"
		crossorigin
	/>
	<link
		rel="preload"
		href="/fonts/optimized_th_web.woff2"
		as="font"
		type="font/woff2"
		crossorigin
	/>
	<link
		rel="preload"
		href="/fonts/optimized_ja_web.woff2"
		as="font"
		type="font/woff2"
		crossorigin
	/>
	<link
		rel="preload"
		href="/fonts/optimized_zh_web.woff2"
		as="font"
		type="font/woff2"
		crossorigin
	/>
	<link
		rel="preload"
		href="/fonts/optimized_ko_web.woff2"
		as="font"
		type="font/woff2"
		crossorigin
	/>

	{#if !dev}
		<link rel="manifest" href="/appmanifest.json" />
	{/if}

	<Iklan head />
</svelte:head>

<Loader {isBannerLoaded} {directLoad} />

<main
	class:mobile={$mobileMode}
	class:preview
	class={$locale}
	style="--screen-height: {$viewportHeight
		? `${$viewportHeight}px`
		: '100vh'};--screen-width: {$viewportWidth}px;
		--genshin-font: var(--gi-{font}-font);"
>
	<Toasts />

	{#if !$isLoading && isloaded}
		<slot />
	{/if}
	<a
		href="/"
		on:click|preventDefault={() => window.location.replace('/')}
		class="uid"
		title="Try Your Luck by this Simulator"
	>
		WishSimulator.App
	</a>
</main>

<style global>
	@import '../../node_modules/overlayscrollbars/css/OverlayScrollbars.css';

	@font-face {
		font-family: 'GI_Global_Web';
		src: url('/fonts/optimized_global_web.woff2') format('woff2');
		font-weight: normal;
		font-style: normal;
	}

	@font-face {
		font-family: 'GI_JA_Web';
		src: url('/fonts/optimized_ja_web.woff2') format('woff2');
		font-weight: normal;
		font-style: normal;
	}

	@font-face {
		font-family: 'GI_KO_Web';
		src: url('/fonts/optimized_ko_web.woff2') format('woff2');
		font-weight: normal;
		font-style: normal;
	}

	@font-face {
		font-family: 'GI_TH_Web';
		src: url('/fonts/optimized_th_web.woff2') format('woff2');
		font-weight: normal;
		font-style: normal;
	}

	@font-face {
		font-family: 'GI_ZH_Web';
		src: url('/fonts/optimized_zh_web.woff2') format('woff2');
		font-weight: normal;
		font-style: normal;
	}

	:global(.os-theme-light > .os-scrollbar > .os-scrollbar-track > .os-scrollbar-handle) {
		background-color: #d2c69c;
		opacity: 0.5;
	}
	:global(.os-theme-light > .os-scrollbar > .os-scrollbar-track > .os-scrollbar-handle:hover),
	:global(.os-theme-light > .os-scrollbar > .os-scrollbar-track > .os-scrollbar-handle:active) {
		background-color: #d2c69c;
		opacity: 1;
	}

	:global(.os-theme-light > .os-scrollbar-vertical) {
		width: 8px;
	}
	:global(.os-theme-light > .os-scrollbar-horizontal) {
		height: 8px;
	}

	main {
		display: block;
		width: var(--screen-width);
		height: var(--screen-height);
		font-family: var(--genshin-font);
		overflow: hidden;
	}

	.uid {
		display: block;
		position: fixed;
		bottom: 0px;
		right: 2em;
		z-index: 9999;
		color: #fff;
		text-shadow: 0 0 1.5px rgba(0, 0, 0, 0.7);
		font-family: Roboto, sans-serif;
		pointer-events: none;
	}

	.mobile .uid {
		right: 5%;
	}

	.preview .uid {
		pointer-events: unset;
		right: unset;
		left: 1rem;
		bottom: 1rem;
	}
</style>
