<script lang="ts">
	import { onMount } from "svelte";

	interface Card {
		id: string;
		name: string;
		src: string;
		type: "blender" | "video" | "media" | "image";
	}

	// 【蓝菲修改·一对一】media 卡片里「一张图片 ↔ 一段文字」成对出现
	interface PairItem {
		id: string;
		img: string;
		text: string;
	}

	interface Content {
		text: string; // 兼容旧数据（media 卡片已改用 items）
		images: string[]; // image 卡片使用
		files: { name: string; size: number; type: string }[];
		items: PairItem[]; // media 卡片：图片 ↔ 文字 一对一
	}

	// 【蓝菲修改·点击查看】浮层里显示的东西：图片 / 视频 / 文件信息
	interface ViewItem {
		kind: "image" | "video" | "file";
		title: string;
		src?: string;
		meta?: string;
	}

	const CARDS: Card[] = [
		{ id: "modeling", name: "建模", src: "/tarot/modeling.webp", type: "blender" },
		{ id: "editing", name: "剪辑", src: "/tarot/editing.webp", type: "video" },
		{ id: "anime", name: "动漫", src: "/tarot/anime.webp", type: "media" },
		{ id: "game", name: "游戏", src: "/tarot/game.webp", type: "media" },
		{ id: "drawing", name: "绘画", src: "/tarot/drawing.webp", type: "image" },
		{ id: "music", name: "音乐", src: "/tarot/music.webp", type: "media" },
	];

	// ══════════════════════════════════════════════════════════════
	// 【蓝菲修改·线上同步】优先读取仓库里的 src/data/tarot-content.json，
	// 这样「导出 → 提交推送」之后，线上访客也能看到同样的内容
	// ══════════════════════════════════════════════════════════════
	const repoModules = import.meta.glob("/src/data/tarot-content.json", { eager: true }) as Record<
		string,
		{ default?: Record<string, unknown> }
	>;
	const repoContent: Record<string, unknown> | null = Object.values(repoModules)[0]?.default ?? null;

	// ══════════════════════════════════════════════════════════════
	// 【蓝菲修改·扇形布局】让 6 张牌围绕同一个下方支点排成圆弧
	// ══════════════════════════════════════════════════════════════
	const FAN_SPREAD = 50;
	const FAN_RADIUS = 520;
	const FAN_TOP = 56;
	const CARD_TOP = 64;
	const SEL_LEFT = 40;
	const SEL_SCALE = 1.18;
	const SEL_ROT = -25;

	const fanAngle = (i: number) => (i - (CARDS.length - 1) / 2) * (FAN_SPREAD / (CARDS.length - 1));
	const fanRad = (i: number) => (fanAngle(i) * Math.PI) / 180;

	const SCALE_START = 0.4;
	const SCALE_FADE = 0.85;
	const SCALE_MOVE = 1.0;
	const SCALE_POP = 0.3;

	// 【蓝菲修改·大图可用】导入图片时先压缩：最长边 1600px → WebP → 逐轮降质/缩尺寸
	const MAX_EDGE = 1600;
	const TARGET_CHARS = 420_000;
	const HARD_LIMIT_CHARS = 1_500_000;
	const MIN_QUALITY = 0.45;
	const MAX_SOURCE_BYTES = 40 * 1024 * 1024;

	interface Pos {
		left: number;
		top: number;
		rot: number;
		scale: number;
		opacity: number;
		z: number;
	}

	let stage: HTMLDivElement;
	let stageW = $state(0);
	let cardW = $state(140);
	let phase = $state<"hidden" | "intro" | "idle">("hidden");
	let selectedId = $state<string | null>(null);
	let editMode = $state(true);
	let poses = $state<Record<string, Pos>>({});
	let contents = $state<Record<string, Content>>({});
	let videoUrls = $state<Record<string, string>>({});
	let dragOverId = $state<string | null>(null);
	let compressing = $state<string | null>(null);
	let storageWarn = $state(false);
	let viewer = $state<ViewItem | null>(null);

	const cardH = () => Math.round(cardW * 2.0);
	const stackedLeft = () => Math.round((stageW - cardW) / 2);

	const fanR = () => {
		const halfW = (stageW - cardW) / 2 - 4;
		const maxR = halfW / Math.sin((FAN_SPREAD / 2) * (Math.PI / 180));
		return Math.max(150, Math.min(FAN_RADIUS, Math.round(maxR)));
	};
	const fanDrop = () => Math.round(fanR() * (1 - Math.cos((FAN_SPREAD / 2) * (Math.PI / 180))));
	const spreadLeft = (i: number) => Math.round(stageW / 2 - cardW / 2 + fanR() * Math.sin(fanRad(i)));
	const spreadTop = (i: number) => Math.round(FAN_TOP + fanR() * (1 - Math.cos(fanRad(i))));
	const spreadRot = (i: number) => Math.round(fanAngle(i) * 10) / 10;
	const stageH = () => FAN_TOP + fanDrop() + cardH() + 90;

	const easeOut = (t: number) => 1 - Math.pow(1 - t, 3);
	const easeInOut = (t: number) => (t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2);

	function animate(
		duration: number,
		ease: (t: number) => number,
		stepFn: (t: number) => void,
		done?: () => void,
	) {
		const start = performance.now();
		const frame = (now: number) => {
			const t = Math.min(1, (now - start) / duration);
			stepFn(ease(t));
			if (t < 1) requestAnimationFrame(frame);
			else done?.();
		};
		requestAnimationFrame(frame);
	}

	function runIntro() {
		for (let i = 0; i < CARDS.length; i++) {
			poses[CARDS[i].id] = {
				left: stackedLeft() + i * 3,
				top: CARD_TOP + i * 4,
				rot: 0,
				scale: SCALE_START,
				opacity: 0,
				z: 10 + i,
			};
		}
		phase = "intro";
		animate(
			300,
			easeOut,
			(t) => {
				for (const c of CARDS) {
					poses[c.id].opacity = t;
					poses[c.id].scale = SCALE_START + (SCALE_FADE - SCALE_START) * t;
				}
			},
			() => {
				setTimeout(() => {
					const fromLeft = stackedLeft();
					const toLeft = Math.round(cardW / 2);
					animate(
						550,
						easeInOut,
						(t) => {
							const l = fromLeft + (toLeft - fromLeft) * t;
							const arc = Math.sin(Math.PI * t) * 42;
							for (let i = 0; i < CARDS.length; i++) {
								poses[CARDS[i].id].left = l + i * 3;
								poses[CARDS[i].id].top = CARD_TOP - arc + i * 4;
								poses[CARDS[i].id].rot = Math.sin(Math.PI * t) * -9;
								poses[CARDS[i].id].scale = SCALE_FADE + (SCALE_MOVE - SCALE_FADE) * t;
							}
						},
						() => {
							const order = [...CARDS].reverse();
							order.forEach((card, k) => {
								const idx = CARDS.indexOf(card);
								setTimeout(() => {
									const from = { left: Math.round(cardW / 2), top: CARD_TOP, rot: 0 };
									const to = { left: spreadLeft(idx), top: spreadTop(idx), rot: spreadRot(idx) };
									animate(520, easeOut, (t) => {
										const arc = Math.sin(Math.PI * t) * 52;
										poses[card.id].left = from.left + (to.left - from.left) * t;
										poses[card.id].top = from.top + (to.top - from.top) * t - arc;
										poses[card.id].rot = to.rot * t;
										poses[card.id].scale = SCALE_MOVE + Math.sin(Math.PI * t) * SCALE_POP;
									});
								}, k * 95);
							});
							setTimeout(() => {
								phase = "idle";
							}, order.length * 95 + 520);
						},
					);
				}, 200);
			},
		);
	}

	function measure() {
		if (!stage) return;
		stageW = stage.clientWidth;
		cardW = Math.max(96, Math.min(150, Math.round(stageW / 6.2)));
		if (phase === "idle" && !selectedId) {
			for (let i = 0; i < CARDS.length; i++) {
				poses[CARDS[i].id] = {
					left: spreadLeft(i),
					top: spreadTop(i),
					rot: spreadRot(i),
					scale: 1,
					opacity: 1,
					z: 10 + i,
				};
			}
		}
	}

	function selectedCard(): Card | undefined {
		return CARDS.find((c) => c.id === selectedId);
	}

	function openCard(id: string) {
		selectedId = id;
		poses[id] = { left: SEL_LEFT, top: CARD_TOP, rot: SEL_ROT, scale: SEL_SCALE, opacity: 1, z: 100 };
		for (const c of CARDS) if (c.id !== id) poses[c.id].opacity = 0.3;
	}

	function closeCard() {
		if (!selectedId) return;
		const id = selectedId;
		selectedId = null;
		setTimeout(() => {
			const idx = CARDS.findIndex((c) => c.id === id);
			if (idx >= 0) {
				poses[id] = {
					left: spreadLeft(idx),
					top: spreadTop(idx),
					rot: spreadRot(idx),
					scale: 1,
					opacity: 1,
					z: 10 + idx,
				};
			}
			for (const c of CARDS) poses[c.id].opacity = 1;
		}, 300);
	}

	function onCardClick(id: string, e: MouseEvent) {
		e.stopPropagation();
		if (phase !== "idle") return;
		if (selectedId === id) return;
		if (selectedId) {
			const prevIdx = CARDS.findIndex((c) => c.id === selectedId);
			if (prevIdx >= 0) {
				poses[selectedId] = {
					left: spreadLeft(prevIdx),
					top: spreadTop(prevIdx),
					rot: spreadRot(prevIdx),
					scale: 1,
					opacity: 1,
					z: 10 + prevIdx,
				};
			}
		}
		openCard(id);
	}

	function onSceneClick() {
		if (selectedId) closeCard();
	}

	const uid = () => Math.random().toString(36).slice(2, 10);

	function emptyContent(): Content {
		return { text: "", images: [], files: [], items: [] };
	}

	function ensureContent(id: string): Content {
		if (!contents[id]) contents[id] = emptyContent();
		return contents[id];
	}

	function normalizeContent(card: Card, raw: unknown): Content {
		const c = emptyContent();
		if (!raw || typeof raw !== "object") return c;
		const r = raw as Record<string, unknown>;
		if (typeof r.text === "string") c.text = r.text;
		if (Array.isArray(r.images)) {
			c.images = r.images.filter((x): x is string => typeof x === "string" && x.length > 0);
		}
		if (Array.isArray(r.files)) {
			c.files = (r.files as Array<Record<string, unknown>>).filter(
				(f) => f && typeof f.name === "string",
			) as Content["files"];
		}
		if (Array.isArray(r.items)) {
			c.items = (r.items as Array<Record<string, unknown>>)
				.filter((it) => it && typeof it.img === "string" && (it.img as string).length > 0)
				.map((it) => ({
					id: typeof it.id === "string" ? it.id : uid(),
					img: it.img as string,
					text: typeof it.text === "string" ? it.text : "",
				}));
		}
		if (card.type === "media" && c.items.length === 0 && c.images.length > 0) {
			c.items = c.images.map((img, i) => ({ id: uid(), img, text: i === 0 ? c.text : "" }));
			c.images = [];
			c.text = "";
		}
		return c;
	}

	function hasAny(c: Content) {
		return c.items.length > 0 || c.images.length > 0 || c.files.length > 0 || c.text.length > 0;
	}

	function persist(id: string) {
		try {
			localStorage.setItem("tarot-content-" + id, JSON.stringify(contents[id]));
			storageWarn = false;
		} catch {
			storageWarn = true;
		}
	}

	// 【蓝菲修改·线上同步】导出整份数据 → 放进 src/data/tarot-content.json 提交即可上线
	function exportJson() {
		const payload = JSON.stringify(contents, null, 2);
		const blob = new Blob([payload], { type: "application/json" });
		const url = URL.createObjectURL(blob);
		const a = document.createElement("a");
		a.href = url;
		a.download = "tarot-content.json";
		a.click();
		URL.revokeObjectURL(url);
		alert(
			"已导出 tarot-content.json 喵～\n\n把它放到 src/data/tarot-content.json 后提交推送，\n线上访客就能看到同样的图文啦。",
		);
	}

	function setText(id: string, v: string) {
		ensureContent(id).text = v;
		persist(id);
	}

	function setItemText(id: string, idx: number, v: string) {
		const c = ensureContent(id);
		const item = c.items[idx];
		if (!item) return;
		item.text = v;
		persist(id);
	}

	function removeItem(id: string, idx: number) {
		const c = ensureContent(id);
		c.items = c.items.filter((_, i) => i !== idx);
		persist(id);
	}

	function moveItem(id: string, idx: number, dir: -1 | 1) {
		const c = ensureContent(id);
		const j = idx + dir;
		if (j < 0 || j >= c.items.length) return;
		const arr = [...c.items];
		const tmp = arr[idx];
		arr[idx] = arr[j];
		arr[j] = tmp;
		c.items = arr;
		persist(id);
	}

	function onFileChange(id: string, e: Event) {
		const input = e.currentTarget as HTMLInputElement;
		void importFiles(id, input.files);
		input.value = "";
	}

	function loadImageElement(file: File): Promise<HTMLImageElement | null> {
		return new Promise((resolve) => {
			const url = URL.createObjectURL(file);
			const img = new Image();
			img.onload = () => {
				URL.revokeObjectURL(url);
				resolve(img);
			};
			img.onerror = () => {
				URL.revokeObjectURL(url);
				resolve(null);
			};
			img.src = url;
		});
	}

	async function compressImage(file: File): Promise<string | null> {
		let source: ImageBitmap | HTMLImageElement | null = null;
		try {
			source = await createImageBitmap(file);
		} catch {
			source = await loadImageElement(file);
		}
		if (!source) return null;
		const sw = source.width;
		const sh = source.height;
		if (!sw || !sh) return null;

		let scale = Math.min(1, MAX_EDGE / Math.max(sw, sh));
		let quality = 0.86;
		let dataUrl = "";

		for (let attempt = 0; attempt < 10; attempt++) {
			const w = Math.max(1, Math.round(sw * scale));
			const h = Math.max(1, Math.round(sh * scale));
			const canvas = document.createElement("canvas");
			canvas.width = w;
			canvas.height = h;
			const ctx = canvas.getContext("2d");
			if (!ctx) return null;
			ctx.imageSmoothingEnabled = true;
			ctx.imageSmoothingQuality = "high";
			ctx.drawImage(source as CanvasImageSource, 0, 0, w, h);

			dataUrl = canvas.toDataURL("image/webp", quality);
			if (!dataUrl.startsWith("data:image/webp")) {
				dataUrl = canvas.toDataURL("image/jpeg", quality);
			}
			if (dataUrl.length <= TARGET_CHARS) break;

			if (quality > MIN_QUALITY) {
				quality = Math.max(MIN_QUALITY, quality - 0.12);
			} else if (scale > 0.25) {
				scale *= 0.8;
			} else {
				break;
			}
		}

		if (typeof ImageBitmap !== "undefined" && source instanceof ImageBitmap) source.close();
		return dataUrl || null;
	}

	function kbOf(dataUrl: string) {
		return Math.max(1, Math.round((dataUrl.length * 0.75) / 1024));
	}

	async function importFiles(id: string, list: FileList | null) {
		if (!list || !list.length) return;
		const card = CARDS.find((c) => c.id === id);
		if (!card) return;
		const c = ensureContent(id);
		compressing = id;

		try {
			for (const f of Array.from(list)) {
				const lower = f.name.toLowerCase();
				const isPsd = lower.endsWith(".psd");
				const isImage = f.type.startsWith("image/") && !isPsd;
				const isVideo = f.type.startsWith("video/");

				if (card.type === "media") {
					if (isImage) {
						if (f.size > MAX_SOURCE_BYTES) {
							alert(`「${f.name}」超过 40MB 啦，先裁一下再拖进来喵～`);
							continue;
						}
						const dataUrl = await compressImage(f);
						if (!dataUrl) {
							alert(`「${f.name}」这张读不出来喵，换一张试试？`);
							continue;
						}
						if (dataUrl.length > HARD_LIMIT_CHARS) {
							alert(`「${f.name}」压完还是太大（${Math.round(dataUrl.length / 1024)}KB），换一张喵`);
							continue;
						}
						c.items = [...c.items, { id: uid(), img: dataUrl, text: "" }];
					} else {
						c.files = [...c.files, { name: f.name, size: f.size, type: f.type || "file" }];
						if (isVideo) {
							try {
								videoUrls[`${id}:${f.name}`] = URL.createObjectURL(f);
							} catch {
								/* 忽略 */
							}
						}
					}
					continue;
				}

				if (card.type === "image" && isImage) {
					if (f.size > MAX_SOURCE_BYTES) {
						alert(`「${f.name}」超过 40MB 啦，先裁一下再拖进来喵～`);
						continue;
					}
					const dataUrl = await compressImage(f);
					if (!dataUrl) {
						alert(`「${f.name}」这张读不出来喵，换一张试试？`);
						continue;
					}
					if (dataUrl.length > HARD_LIMIT_CHARS) {
						alert(`「${f.name}」压完还是太大（${Math.round(dataUrl.length / 1024)}KB），换一张喵`);
						continue;
					}
					c.images = [...c.images, dataUrl];
					continue;
				}

				c.files = [...c.files, { name: f.name, size: f.size, type: f.type || "file" }];
				if (isVideo) {
					try {
						videoUrls[`${id}:${f.name}`] = URL.createObjectURL(f);
					} catch {
						/* 忽略 */
					}
				}
			}
			persist(id);
		} finally {
			compressing = null;
		}
	}

	function removeImage(id: string, i: number) {
		ensureContent(id).images = ensureContent(id).images.filter((_, idx) => idx !== i);
		persist(id);
	}

	function removeFile(id: string, i: number) {
		ensureContent(id).files = ensureContent(id).files.filter((_, idx) => idx !== i);
		persist(id);
	}

	function fmtSize(n: number) {
		if (n < 1024) return n + " B";
		if (n < 1024 * 1024) return (n / 1024).toFixed(1) + " KB";
		return (n / 1024 / 1024).toFixed(1) + " MB";
	}

	function toggleEdit() {
		editMode = !editMode;
		try {
			localStorage.setItem("tarot-edit-mode", editMode ? "1" : "0");
		} catch {
			/* 忽略 */
		}
	}

	// ── 点击查看浮层 ──
	function showImage(src: string, title: string, meta = "") {
		viewer = { kind: "image", title, src, meta };
	}
	function showVideo(src: string, title: string, meta = "") {
		viewer = { kind: "video", title, src, meta };
	}
	function showFile(name: string, size: number, type: string) {
		viewer = {
			kind: "file",
			title: name,
			meta: `大小：${fmtSize(size)}　类型：${type || "未知"}\n\n（浏览器只保存了文件记录，没有保存文件本体，所以无法在这里预览喵～）`,
		};
	}
	function closeViewer() {
		viewer = null;
	}

	// ── 拖拽 ──
	function stopDragDefault(e: DragEvent) {
		e.preventDefault();
	}
	function handleDragOver(e: DragEvent, id: string) {
		e.preventDefault();
		e.stopPropagation();
		if (e.dataTransfer) e.dataTransfer.dropEffect = "copy";
		dragOverId = id;
	}
	function handleDragLeave() {
		dragOverId = null;
	}
	async function handleDrop(e: DragEvent, id: string) {
		e.preventDefault();
		e.stopPropagation();
		dragOverId = null;
		if (phase !== "idle") return;
		if (selectedId !== id) {
			if (selectedId) {
				const prevIdx = CARDS.findIndex((c) => c.id === selectedId);
				if (prevIdx >= 0) {
					poses[selectedId] = {
						left: spreadLeft(prevIdx),
						top: spreadTop(prevIdx),
						rot: spreadRot(prevIdx),
						scale: 1,
						opacity: 1,
						z: 10 + prevIdx,
					};
				}
			}
			openCard(id);
		}
		await importFiles(id, e.dataTransfer?.files ?? null);
	}

	onMount(() => {
		// 【蓝菲修改·线上同步】仓库数据打底，本地 localStorage 有内容时以后者为准
		for (const c of CARDS) {
			let merged = normalizeContent(c, repoContent?.[c.id] ?? null);
			try {
				const raw = localStorage.getItem("tarot-content-" + c.id);
				if (raw) {
					const local = normalizeContent(c, JSON.parse(raw));
					if (hasAny(local)) merged = local;
				}
			} catch {
				/* 忽略坏数据 */
			}
			contents[c.id] = merged;
		}
		try {
			const em = localStorage.getItem("tarot-edit-mode");
			if (em !== null) editMode = em === "1";
		} catch {
			/* 忽略 */
		}
		measure();
		runIntro();
		window.addEventListener("resize", measure);
		return () => window.removeEventListener("resize", measure);
	});
</script>

<svelte:window onkeydown={(e) => e.key === "Escape" && closeViewer()} />

<div
	class="tarot-stage"
	bind:this={stage}
	style="height:{stageH()}px"
	ondragover={stopDragDefault}
	ondrop={stopDragDefault}
>
	<div class="tarot-scene" onclick={onSceneClick}>
		{#each CARDS as card (card.id)}
			{@const p = poses[card.id]}
			{#if p}
				<button
					class="tarot-card"
					class:drag-target={dragOverId === card.id}
					type="button"
					data-card-id={card.id}
					style="left:{p.left}px; top:{p.top}px; width:{cardW}px; height:{cardH()}px; transform:rotate({p.rot}deg) scale({p.scale}); opacity:{p.opacity}; z-index:{p.z}; transition:{phase === 'idle' ? 'left .55s cubic-bezier(.22,1,.36,1), top .55s cubic-bezier(.22,1,.36,1), transform .55s cubic-bezier(.22,1,.36,1), opacity .3s' : 'none'};"
					onclick={(e) => onCardClick(card.id, e)}
					ondragover={(e) => handleDragOver(e, card.id)}
					ondragleave={handleDragLeave}
					ondrop={(e) => handleDrop(e, card.id)}
				>
					<img src={card.src} alt={card.name} />
					<span class="tarot-label">{card.name}</span>
				</button>
			{/if}
		{/each}
	</div>

	<!-- 白色竖线 -->
	<div
		class="tarot-divider"
		style="opacity:{selectedId ? 1 : 0}; height:{Math.round(cardH() * SEL_SCALE)}px; left:{SEL_LEFT + Math.round(cardW * SEL_SCALE) + 26}px; top:{CARD_TOP}px;"
	></div>

	<!-- 内容面板 -->
	<div
		class="tarot-panel"
		class:pulling={!!dragOverId}
		style="opacity:{selectedId ? 1 : 0}; pointer-events:{selectedId ? 'auto' : 'none'}; left:{SEL_LEFT + Math.round(cardW * SEL_SCALE) + 52}px; top:{CARD_TOP}px; height:{Math.round(cardH() * SEL_SCALE)}px; right:16px;"
		onclick={(e) => e.stopPropagation()}
		ondragover={(e) => selectedId && handleDragOver(e, selectedId)}
		ondragleave={handleDragLeave}
		ondrop={(e) => selectedId && handleDrop(e, selectedId)}
	>
		{#if selectedId}
			{@const card = selectedCard()}
			{#if card}
				{@const c = contents[card.id]}
				<div class="panel-head">
					<span class="panel-title">{card.name}</span>
					<button class="panel-btn" type="button" onclick={toggleEdit} title={editMode ? "点击切换为只读" : "点击切换为编辑"}>
						{editMode ? "✏️ 编辑中" : "👁 只读"}
					</button>
					<button class="panel-btn" type="button" onclick={exportJson} title="导出 tarot-content.json（放进 src/data 提交后线上也能看到）">
						⬇ 导出
					</button>
					<button class="panel-btn" type="button" onclick={closeCard} title="关闭">✕</button>
				</div>
				<div class="panel-body">
					{#if c}
						{#if storageWarn}
							<p class="storage-warn">
								⚠️ 浏览器本地存储满了喵：新加的图片只存在内存里，刷新会丢。删掉几组再试～
							</p>
						{/if}
						{#if card.type === "blender"}
							<p class="panel-hint">把 Blender / 3D 文件拖进来（.blend .obj .fbx .glb .gltf …）</p>
							{#if editMode}
								<label class="pick-btn">
									＋ 添加文件
									<input type="file" multiple accept=".blend,.obj,.fbx,.glb,.gltf,.stl,.abc" onchange={(e) => onFileChange(card.id, e)} />
								</label>
							{/if}
							<ul class="file-list">
								{#each c.files as f, i}
									<li>
										<button class="file-open" type="button" onclick={() => showFile(f.name, f.size, f.type)}>📦 {f.name}</button>
										<span class="dim">{fmtSize(f.size)}</span>
										{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
									</li>
								{/each}
							</ul>

						{:else if card.type === "video"}
							<p class="panel-hint">把视频拖进来（mp4 / webm …）· 播放窗口 16:9</p>
							{#if editMode}
								<label class="pick-btn">
									＋ 添加视频
									<input type="file" multiple accept="video/*" onchange={(e) => onFileChange(card.id, e)} />
								</label>
							{/if}
							<div class="video-list">
								{#each c.files as f, i}
									<div class="video-item">
										{#if videoUrls[`${card.id}:${f.name}`]}
											<button
												class="video-frame"
												type="button"
												title="点击放大播放"
												onclick={() => showVideo(videoUrls[`${card.id}:${f.name}`], f.name, fmtSize(f.size))}
											>
												<video src={videoUrls[`${card.id}:${f.name}`]} muted></video>
												<span class="video-play">▶</span>
											</button>
										{/if}
										<span class="video-name">🎬 {f.name} · {fmtSize(f.size)}</span>
										{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
									</div>
								{/each}
							</div>

						{:else if card.type === "media"}
							<!-- 【蓝菲修改·一对一·16:9】左边 16:9 图片，右边同高文字 -->
							<div class="panel-tools">
								<p class="panel-hint">一张图配一段文字 · 共 {c.items.length} 组</p>
								{#if compressing === card.id}
									<span class="busy">🌀 正在压图…</span>
								{/if}
								{#if editMode}
									<label class="pick-btn">
										＋ 添加图片
										<input type="file" multiple accept="image/*" onchange={(e) => onFileChange(card.id, e)} />
									</label>
								{/if}
							</div>
							<div class="pair-list">
								{#each c.items as item, i (item.id)}
									<div class="pair-row">
										<button
											class="pair-img"
											type="button"
											title="点击查看大图"
											onclick={() => showImage(item.img, `${card.name} #${i + 1}`, item.text)}
										>
											<img src={item.img} alt="" />
											<span class="pair-no">#{i + 1} · {kbOf(item.img)}KB</span>
										</button>
										<div class="pair-text">
											{#if editMode}
												<textarea
													value={item.text}
													oninput={(e) => setItemText(card.id, i, e.currentTarget.value)}
													placeholder="给左边这张图配一段文字…"
												></textarea>
												<div class="pair-actions">
													<button class="mini-btn" type="button" title="上移" onclick={() => moveItem(card.id, i, -1)}>↑</button>
													<button class="mini-btn" type="button" title="下移" onclick={() => moveItem(card.id, i, 1)}>↓</button>
													<button class="mini-btn danger" type="button" title="删除这一组" onclick={() => removeItem(card.id, i)}>✕</button>
												</div>
											{:else}
												<p class="pair-readonly">{item.text || "（还没有文字）"}</p>
											{/if}
										</div>
									</div>
								{:else}
									<p class="empty-hint">还没有内容喵～ 点「＋ 添加图片」或者直接把图片拖进来</p>
								{/each}
							</div>
							{#if c.files.length > 0}
								<p class="panel-hint">其他文件</p>
								<ul class="file-list">
									{#each c.files as f, i}
										<li>
											<button class="file-open" type="button" onclick={() => showFile(f.name, f.size, f.type)}>📎 {f.name}</button>
											<span class="dim">{fmtSize(f.size)}</span>
											{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
										</li>
									{/each}
								</ul>
							{/if}

						{:else if card.type === "image"}
							<p class="panel-hint">导入 PS(.psd) 或图片(.png/.jpg/.webp) · 图片会自动压缩</p>
							<div class="panel-tools">
								{#if compressing === card.id}
									<span class="busy">🌀 正在压图…</span>
								{/if}
								{#if editMode}
									<label class="pick-btn">
										＋ 添加图片 / PSD
										<input type="file" multiple accept=".psd,.png,.jpg,.jpeg,.webp,image/png,image/jpeg,image/webp" onchange={(e) => onFileChange(card.id, e)} />
									</label>
								{/if}
							</div>
							<div class="img-grid">
								{#each c.images as img, i}
									<div class="img-item">
										<button class="img-open" type="button" title="点击查看大图" onclick={() => showImage(img, `${card.name} #${i + 1}`)}>
											<img src={img} alt="" />
										</button>
										{#if editMode}<button class="mini-x abs" type="button" onclick={() => removeImage(card.id, i)}>✕</button>{/if}
									</div>
								{/each}
							</div>
							<ul class="file-list">
								{#each c.files as f, i}
									<li>
										<button class="file-open" type="button" onclick={() => showFile(f.name, f.size, f.type)}>🧩 {f.name}</button>
										<span class="dim">{fmtSize(f.size)}</span>
										{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
									</li>
								{/each}
							</ul>
						{/if}
					{/if}
				</div>
				{#if dragOverId === card.id}
					<div class="drop-veil">松手就放进「{card.name}」喵～</div>
				{/if}
			{/if}
		{/if}
	</div>
</div>

<!-- 【蓝菲修改·点击查看】全屏查看浮层 -->
{#if viewer}
	<div class="viewer" onclick={closeViewer} role="presentation">
		<div class="viewer-box" onclick={(e) => e.stopPropagation()} role="presentation">
			<div class="viewer-head">
				<span class="viewer-title">{viewer.title}</span>
				<button class="panel-btn" type="button" onclick={closeViewer}>✕ 关闭</button>
			</div>
			{#if viewer.kind === "image" && viewer.src}
				<img class="viewer-img" src={viewer.src} alt={viewer.title} />
			{:else if viewer.kind === "video" && viewer.src}
				<video class="viewer-video" src={viewer.src} controls autoplay></video>
			{/if}
			{#if viewer.meta}
				<p class="viewer-meta">{viewer.meta}</p>
			{/if}
		</div>
	</div>
{/if}

<style>
	.tarot-stage {
		position: relative;
		width: 100%;
	}
	.tarot-scene {
		position: absolute;
		inset: 0;
	}
	.tarot-card {
		position: absolute;
		border-radius: 12px;
		overflow: hidden;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
		border: 1px solid rgba(255, 255, 255, 0.4);
		background: #1b1b2f;
		cursor: pointer;
		padding: 0;
		will-change: left, top, transform, opacity;
	}
	.tarot-card.drag-target {
		outline: 3px dashed var(--primary, #6ea8ff);
		outline-offset: -3px;
	}
	.tarot-card img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		display: block;
	}
	.tarot-label {
		position: absolute;
		left: 50%;
		bottom: 12px;
		transform: translateX(-50%);
		color: #fff;
		font-size: 13px;
		font-weight: 700;
		padding: 2px 10px;
		border-radius: 999px;
		background: rgba(0, 0, 0, 0.45);
		white-space: nowrap;
		text-shadow: 0 1px 2px rgba(0, 0, 0, 0.6);
		letter-spacing: 2px;
	}
	.tarot-divider {
		position: absolute;
		z-index: 60;
		width: 2px;
		background: #fff;
		border-radius: 2px;
		box-shadow: 0 0 8px rgba(255, 255, 255, 0.8);
		transition: opacity 0.25s ease 0.28s;
		pointer-events: none;
	}
	.tarot-panel {
		position: absolute;
		z-index: 50;
		border-radius: 16px;
		background: rgba(140, 145, 155, 0.5);
		backdrop-filter: blur(6px);
		-webkit-backdrop-filter: blur(6px);
		box-shadow: 0 8px 28px rgba(0, 0, 0, 0.25);
		transition: opacity 0.3s ease;
		overflow: hidden;
		display: flex;
		flex-direction: column;
	}
	.tarot-panel.pulling {
		box-shadow: 0 0 0 2px var(--primary, #6ea8ff) inset;
	}
	.panel-head {
		display: flex;
		align-items: center;
		gap: 6px;
		padding: 10px 12px;
		border-bottom: 1px solid rgba(255, 255, 255, 0.35);
		flex: 0 0 auto;
		flex-wrap: wrap;
	}
	.panel-title {
		font-weight: 700;
		font-size: 15px;
		margin-right: auto;
	}
	.panel-btn {
		border: none;
		background: rgba(0, 0, 0, 0.18);
		color: inherit;
		border-radius: 8px;
		padding: 4px 10px;
		cursor: pointer;
		font-size: 12px;
	}
	.panel-btn:hover {
		background: rgba(0, 0, 0, 0.3);
	}

	/* 可编辑区域：右侧常驻竖向滚动条 */
	.panel-body {
		flex: 1 1 auto;
		min-height: 0;
		overflow-y: auto;
		overflow-x: hidden;
		padding: 12px 8px 12px 14px;
		scrollbar-gutter: stable;
		overscroll-behavior: contain;
		scrollbar-width: thin;
		scrollbar-color: rgba(0, 0, 0, 0.38) transparent;
	}
	.panel-body::-webkit-scrollbar {
		width: 10px;
	}
	.panel-body::-webkit-scrollbar-track {
		background: rgba(255, 255, 255, 0.22);
		border-radius: 999px;
	}
	.panel-body::-webkit-scrollbar-thumb {
		background: rgba(0, 0, 0, 0.38);
		border-radius: 999px;
		border: 2px solid transparent;
		background-clip: content-box;
	}
	.panel-body::-webkit-scrollbar-thumb:hover {
		background: rgba(0, 0, 0, 0.55);
		background-clip: content-box;
	}

	.panel-hint {
		font-size: 12px;
		opacity: 0.8;
		margin: 0 0 8px;
	}
	.panel-tools {
		display: flex;
		align-items: center;
		gap: 10px;
		flex-wrap: wrap;
		margin-bottom: 10px;
	}
	.panel-tools .panel-hint {
		margin: 0;
	}
	.busy {
		font-size: 12px;
		opacity: 0.85;
	}
	.storage-warn {
		margin: 0 0 10px;
		padding: 6px 10px;
		border-radius: 8px;
		font-size: 12px;
		background: rgba(255, 196, 0, 0.25);
		border: 1px solid rgba(255, 196, 0, 0.5);
	}
	.pick-btn {
		display: inline-flex;
		align-items: center;
		gap: 6px;
		padding: 4px 12px;
		border-radius: 999px;
		background: rgba(0, 0, 0, 0.2);
		font-size: 12px;
		cursor: pointer;
		user-select: none;
		transition: background 0.2s;
	}
	.pick-btn:hover {
		background: rgba(0, 0, 0, 0.32);
	}
	.pick-btn input[type="file"] {
		display: none;
	}

	/* ══════════════════════════════════════════════════════════
	   【蓝菲修改·一对一 + 16:9】左边图片固定 16:9，右边文字同高
	   ══════════════════════════════════════════════════════════ */
	.pair-list {
		display: flex;
		flex-direction: column;
		gap: 10px;
	}
	.pair-row {
		display: flex;
		gap: 10px;
		align-items: stretch;
		padding: 8px;
		border-radius: 12px;
		background: rgba(255, 255, 255, 0.22);
		border: 1px solid rgba(255, 255, 255, 0.3);
	}
	.pair-img {
		position: relative;
		flex: 0 0 auto;
		width: 42%;
		aspect-ratio: 16 / 9;
		height: auto;
		border: none;
		padding: 0;
		border-radius: 10px;
		overflow: hidden;
		background: rgba(0, 0, 0, 0.1);
		cursor: zoom-in;
	}
	.pair-img img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		display: block;
	}
	.pair-no {
		position: absolute;
		left: 4px;
		bottom: 4px;
		font-size: 10px;
		padding: 1px 6px;
		border-radius: 999px;
		background: rgba(0, 0, 0, 0.5);
		color: #fff;
	}
	.pair-text {
		flex: 1 1 auto;
		min-width: 0;
		display: flex;
		flex-direction: column;
		gap: 6px;
	}
	.pair-text textarea {
		flex: 1 1 auto;
		width: 100%;
		min-height: 72px;
		border-radius: 10px;
		border: 1px solid rgba(255, 255, 255, 0.35);
		background: rgba(255, 255, 255, 0.6);
		padding: 6px 9px;
		font: inherit;
		font-size: 13px;
		resize: none;
		color: inherit;
	}
	.pair-readonly {
		margin: 0;
		font-size: 13px;
		white-space: pre-wrap;
		word-break: break-word;
		opacity: 0.9;
	}
	.pair-actions {
		display: flex;
		gap: 6px;
		justify-content: flex-end;
	}
	.mini-btn {
		border: none;
		background: rgba(0, 0, 0, 0.16);
		color: inherit;
		border-radius: 6px;
		width: 22px;
		height: 20px;
		font-size: 11px;
		line-height: 1;
		cursor: pointer;
		padding: 0;
	}
	.mini-btn:hover {
		background: rgba(0, 0, 0, 0.3);
	}
	.mini-btn.danger {
		background: rgba(200, 60, 60, 0.35);
	}
	.empty-hint {
		margin: 6px 0;
		font-size: 12px;
		opacity: 0.75;
	}

	.file-list {
		list-style: none;
		margin: 0;
		padding: 0;
		display: flex;
		flex-wrap: wrap;
		gap: 8px;
	}
	.file-list li {
		display: flex;
		align-items: center;
		gap: 8px;
		background: rgba(0, 0, 0, 0.14);
		border-radius: 8px;
		padding: 4px 10px;
		font-size: 12px;
	}
	.file-open {
		border: none;
		background: none;
		color: inherit;
		font: inherit;
		cursor: pointer;
		padding: 0;
		text-align: left;
	}
	.file-open:hover {
		text-decoration: underline;
	}
	.dim {
		opacity: 0.65;
		font-size: 11px;
	}

	.img-grid {
		display: flex;
		flex-wrap: wrap;
		gap: 8px;
		margin-bottom: 8px;
	}
	.img-item {
		position: relative;
		width: 108px;
		aspect-ratio: 16 / 9;
	}
	.img-open {
		border: none;
		padding: 0;
		width: 100%;
		height: 100%;
		cursor: zoom-in;
		border-radius: 8px;
		overflow: hidden;
		background: none;
	}
	.img-open img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		border-radius: 8px;
		display: block;
	}
	.mini-x {
		border: none;
		background: rgba(0, 0, 0, 0.5);
		color: #fff;
		border-radius: 50%;
		width: 18px;
		height: 18px;
		font-size: 10px;
		cursor: pointer;
		line-height: 1;
		padding: 0;
	}
	.mini-x.abs {
		position: absolute;
		top: 2px;
		right: 2px;
	}

	/* 【蓝菲修改·16:9】视频播放窗口固定 16:9 */
	.video-list {
		display: flex;
		flex-direction: column;
		gap: 12px;
	}
	.video-item {
		display: flex;
		flex-direction: column;
		gap: 4px;
	}
	.video-frame {
		position: relative;
		width: 100%;
		aspect-ratio: 16 / 9;
		padding: 0;
		border: none;
		border-radius: 10px;
		overflow: hidden;
		background: #000;
		cursor: zoom-in;
	}
	.video-frame video {
		width: 100%;
		height: 100%;
		object-fit: contain;
		display: block;
		background: #000;
		pointer-events: none;
	}
	.video-play {
		position: absolute;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		color: #fff;
		font-size: 28px;
		text-shadow: 0 2px 8px rgba(0, 0, 0, 0.7);
		background: rgba(0, 0, 0, 0.18);
	}
	.video-name {
		font-size: 12px;
	}

	/* 【蓝菲修改·拖拽】松手提示 */
	.drop-veil {
		position: absolute;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 13px;
		font-weight: 700;
		color: #fff;
		background: rgba(40, 90, 170, 0.45);
		backdrop-filter: blur(2px);
		pointer-events: none;
		text-shadow: 0 1px 3px rgba(0, 0, 0, 0.5);
	}

	/* 【蓝菲修改·点击查看】全屏浮层 */
	.viewer {
		position: fixed;
		inset: 0;
		z-index: 9999;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 24px;
		background: rgba(0, 0, 0, 0.72);
		backdrop-filter: blur(4px);
	}
	.viewer-box {
		max-width: min(92vw, 1200px);
		max-height: 88vh;
		display: flex;
		flex-direction: column;
		gap: 8px;
		padding: 12px;
		border-radius: 14px;
		background: rgba(30, 32, 40, 0.92);
		color: #f3f4f8;
		box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
		overflow: auto;
	}
	.viewer-head {
		display: flex;
		align-items: center;
		gap: 10px;
	}
	.viewer-title {
		font-weight: 700;
		font-size: 14px;
		margin-right: auto;
	}
	.viewer-img {
		max-width: 100%;
		max-height: 70vh;
		object-fit: contain;
		border-radius: 10px;
		display: block;
		margin: 0 auto;
	}
	.viewer-video {
		width: min(88vw, 1080px);
		aspect-ratio: 16 / 9;
		border-radius: 10px;
		background: #000;
		display: block;
	}
	.viewer-meta {
		margin: 0;
		font-size: 13px;
		line-height: 1.7;
		white-space: pre-wrap;
		opacity: 0.9;
	}
</style>
