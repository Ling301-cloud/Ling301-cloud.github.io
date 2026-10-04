<script lang="ts">
	import { onMount } from "svelte";

	interface Card {
		id: string;
		name: string;
		src: string;
		type: "blender" | "video" | "media" | "image";
	}

	const CARDS: Card[] = [
		{ id: "modeling", name: "建模", src: "/tarot/modeling.webp", type: "blender" },
		{ id: "editing", name: "剪辑", src: "/tarot/editing.webp", type: "video" },
		{ id: "anime", name: "动漫", src: "/tarot/anime.webp", type: "media" },
		{ id: "game", name: "游戏", src: "/tarot/game.webp", type: "media" },
		{ id: "drawing", name: "绘画", src: "/tarot/drawing.webp", type: "image" },
		{ id: "music", name: "音乐", src: "/tarot/music.webp", type: "media" },
	];

	const ROTS = [-7, -4, -1, 1, 4, 7];
	const CARD_TOP = 64;
	const SEL_LEFT = 40;
	const SEL_SCALE = 1.18;
	const SEL_ROT = 30;

	interface Pos {
		left: number;
		top: number;
		rot: number;
		scale: number;
		opacity: number;
		z: number;
	}

	interface Content {
		text: string;
		images: string[];
		files: { name: string; size: number; type: string }[];
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

	const cardH = () => Math.round(cardW * 2.0);
	const overlap = () => Math.round(cardW * 0.36);
	const step = () => cardW - overlap();
	const spreadW = () => cardW + (CARDS.length - 1) * step();
	const stageH = () => CARD_TOP + cardH() + 80;
	const spreadLeft = (i: number) => Math.round((stageW - spreadW()) / 2 + i * step());
	const stackedLeft = () => Math.round((stageW - cardW) / 2);

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
				scale: 1,
				opacity: 0,
				z: i,
			};
		}
		phase = "intro";
		// 0.3s 渐显
		animate(
			300,
			easeOut,
			(t) => {
				for (const c of CARDS) poses[c.id].opacity = t;
			},
			() => {
				// 停顿 0.2s
				setTimeout(() => {
					const fromLeft = stackedLeft();
					const toLeft = Math.round(cardW / 2);
					// 沿弧线向左移动
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
							}
						},
						() => {
							// 从上到下（图层顺序）沿弧线向右摊开
							const order = [...CARDS].reverse();
							order.forEach((card, k) => {
								const idx = CARDS.indexOf(card);
								setTimeout(() => {
									const from = { left: Math.round(cardW / 2), top: CARD_TOP, rot: 0 };
									const to = { left: spreadLeft(idx), top: CARD_TOP, rot: ROTS[idx] };
									animate(520, easeOut, (t) => {
										const arc = Math.sin(Math.PI * t) * 52;
										poses[card.id].left = from.left + (to.left - from.left) * t;
										poses[card.id].top = from.top - arc;
										poses[card.id].rot = to.rot * t;
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
					top: CARD_TOP,
					rot: ROTS[i],
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
					top: CARD_TOP,
					rot: ROTS[idx],
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
					top: CARD_TOP,
					rot: ROTS[prevIdx],
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

	function persist(id: string) {
		try {
			localStorage.setItem("tarot-content-" + id, JSON.stringify(contents[id]));
		} catch {
			/* 存储空间不足 */
		}
	}

	function setText(id: string, v: string) {
		contents[id].text = v;
		persist(id);
	}

	function onFileChange(id: string, e: Event) {
		const input = e.currentTarget as HTMLInputElement;
		importFiles(id, input.files);
		input.value = "";
	}

	async function importFiles(id: string, list: FileList | null) {
		if (!list || !list.length) return;
		const type = CARDS.find((c) => c.id === id)?.type ?? "media";
		const c = contents[id];
		for (const f of Array.from(list)) {
			const isImage = f.type.startsWith("image/") && !f.name.toLowerCase().endsWith(".psd");
			const isPsd = f.name.toLowerCase().endsWith(".psd");
			const isVideo = f.type.startsWith("video/");

			if (type === "image" && isPsd) {
				c.files = [...c.files, { name: f.name, size: f.size, type: f.type || "psd" }];
			} else if (type === "blender" || type === "video") {
				c.files = [...c.files, { name: f.name, size: f.size, type: f.type || "file" }];
				if (isVideo) {
					try {
						videoUrls[f.name] = URL.createObjectURL(f);
					} catch {
						/* 忽略 */
					}
				}
			} else if (isImage) {
				const dataUrl: string = await new Promise((res, rej) => {
					const r = new FileReader();
					r.onload = () => res(r.result as string);
					r.onerror = rej;
					r.readAsDataURL(f);
				});
				if (dataUrl.length > 1_500_000) {
					alert("这张图片太大了（超过 1.5MB），请换一张小点的");
					continue;
				}
				c.images = [...c.images, dataUrl];
			} else {
				c.files = [...c.files, { name: f.name, size: f.size, type: f.type || "file" }];
			}
		}
		persist(id);
	}

	function removeImage(id: string, i: number) {
		contents[id].images.splice(i, 1);
		contents[id].images = [...contents[id].images];
		persist(id);
	}

	function removeFile(id: string, i: number) {
		contents[id].files.splice(i, 1);
		contents[id].files = [...contents[id].files];
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

	onMount(() => {
		for (const c of CARDS) {
			try {
				const raw = localStorage.getItem("tarot-content-" + c.id);
				contents[c.id] = raw ? JSON.parse(raw) : { text: "", images: [], files: [] };
			} catch {
				contents[c.id] = { text: "", images: [], files: [] };
			}
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

<div class="tarot-stage" bind:this={stage} style="height:{stageH()}px">
	<div class="tarot-scene" onclick={onSceneClick}>
		{#each CARDS as card (card.id)}
			{@const p = poses[card.id]}
			{#if p}
				<button
					class="tarot-card"
					type="button"
					style="left:{p.left}px; top:{p.top}px; width:{cardW}px; height:{cardH()}px; transform:rotate({p.rot}deg) scale({p.scale}); opacity:{p.opacity}; z-index:{p.z}; transition:{phase === 'idle' ? 'left .55s cubic-bezier(.22,1,.36,1), top .55s cubic-bezier(.22,1,.36,1), transform .55s cubic-bezier(.22,1,.36,1), opacity .3s' : 'none'};"
					onclick={(e) => onCardClick(card.id, e)}
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
		style="opacity:{selectedId ? 1 : 0}; left:{SEL_LEFT + Math.round(cardW * SEL_SCALE) + 52}px; top:{CARD_TOP}px; height:{Math.round(cardH() * SEL_SCALE)}px; right:16px;"
		onclick={(e) => e.stopPropagation()}
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
					<button class="panel-btn" type="button" onclick={closeCard} title="关闭">✕</button>
				</div>
				<div class="panel-body">
					{#if c}
						{#if card.type === "blender"}
							<p class="panel-hint">把 Blender / 3D 文件拖进来（.blend .obj .fbx .glb .gltf …）</p>
							{#if editMode}
								<input type="file" multiple accept=".blend,.obj,.fbx,.glb,.gltf,.stl,.abc" onchange={(e) => onFileChange(card.id, e)} />
							{/if}
							<ul class="file-list">
								{#each c.files as f, i}
									<li>
										<span>📦 {f.name}</span>
										<span class="dim">{fmtSize(f.size)}</span>
										{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
									</li>
								{/each}
							</ul>

						{:else if card.type === "video"}
							<p class="panel-hint">把视频拖进来（mp4 / webm …）</p>
							{#if editMode}
								<input type="file" multiple accept="video/*" onchange={(e) => onFileChange(card.id, e)} />
							{/if}
							<div class="video-list">
								{#each c.files as f, i}
									<div class="video-item">
										{#if videoUrls[f.name]}
											<video src={videoUrls[f.name]} controls></video>
										{/if}
										<span class="video-name">🎬 {f.name} · {fmtSize(f.size)}</span>
										{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
									</div>
								{/each}
							</div>

						{:else if card.type === "media"}
							<div class="media-grid">
								<div class="media-images">
									<p class="panel-hint">图片区</p>
									{#if editMode}
										<input type="file" multiple accept="image/*" onchange={(e) => onFileChange(card.id, e)} />
									{/if}
									<div class="img-grid">
										{#each c.images as img, i}
											<div class="img-item">
												<img src={img} alt="" />
												{#if editMode}<button class="mini-x abs" type="button" onclick={() => removeImage(card.id, i)}>✕</button>{/if}
											</div>
										{/each}
									</div>
								</div>
								<div class="media-text">
									<p class="panel-hint">文字区</p>
									<textarea
										readonly={!editMode}
										value={c.text}
										oninput={(e) => setText(card.id, e.currentTarget.value)}
										placeholder="在这里写点东西…"
									></textarea>
								</div>
							</div>

						{:else if card.type === "image"}
							<p class="panel-hint">导入 PS(.psd) 或图片(.png/.jpg/.webp)</p>
							{#if editMode}
								<input type="file" multiple accept=".psd,.png,.jpg,.jpeg,.webp,image/png,image/jpeg,image/webp" onchange={(e) => onFileChange(card.id, e)} />
							{/if}
							<div class="img-grid">
								{#each c.images as img, i}
									<div class="img-item">
										<img src={img} alt="" />
										{#if editMode}<button class="mini-x abs" type="button" onclick={() => removeImage(card.id, i)}>✕</button>{/if}
									</div>
								{/each}
							</div>
							<ul class="file-list">
								{#each c.files as f, i}
									<li>
										<span>🧩 {f.name}</span>
										<span class="dim">{fmtSize(f.size)}</span>
										{#if editMode}<button class="mini-x" type="button" onclick={() => removeFile(card.id, i)}>✕</button>{/if}
									</li>
								{/each}
							</ul>
						{/if}
					{/if}
				</div>
			{/if}
		{/if}
	</div>
</div>

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
		width: 2px;
		background: #fff;
		border-radius: 2px;
		box-shadow: 0 0 8px rgba(255, 255, 255, 0.8);
		transition: opacity 0.25s ease 0.28s;
		pointer-events: none;
	}
	.tarot-panel {
		position: absolute;
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
	.panel-head {
		display: flex;
		align-items: center;
		gap: 8px;
		padding: 10px 14px;
		border-bottom: 1px solid rgba(255, 255, 255, 0.35);
	}
	.panel-title {
		font-weight: 700;
		font-size: 15px;
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
	.panel-btn:first-of-type {
		margin-left: auto;
	}
	.panel-body {
		flex: 1;
		overflow: auto;
		padding: 12px 14px;
	}
	.panel-hint {
		font-size: 12px;
		opacity: 0.8;
		margin: 0 0 8px;
	}
	.panel-body input[type="file"] {
		margin-bottom: 10px;
		font-size: 12px;
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
		width: 72px;
		height: 72px;
	}
	.img-item img {
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
	.media-grid {
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 12px;
		height: 100%;
	}
	.media-text {
		min-width: 0;
		display: flex;
		flex-direction: column;
	}
	.media-text textarea {
		flex: 1;
		width: 100%;
		min-height: 120px;
		border-radius: 10px;
		border: 1px solid rgba(255, 255, 255, 0.35);
		background: rgba(255, 255, 255, 0.5);
		padding: 8px 10px;
		font: inherit;
		resize: none;
		color: inherit;
	}
	.video-list {
		display: flex;
		flex-direction: column;
		gap: 10px;
	}
	.video-item video {
		width: 100%;
		max-height: 140px;
		border-radius: 8px;
		display: block;
		margin-bottom: 4px;
		background: #000;
	}
	.video-name {
		font-size: 12px;
	}
</style>
