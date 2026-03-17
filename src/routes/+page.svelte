<script lang="ts">
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
  import { VRMLoaderPlugin } from "@pixiv/three-vrm";
  import {
    Settings,
    ChevronDown,
    ChevronRight,
    Box,
    Activity,
    ShieldCheck,
    ExternalLink,
    GripVertical,
    Search,
    X,
    Github,
  } from "@lucide/svelte";

  let canvas: HTMLCanvasElement;
  let viewportContainer: HTMLDivElement;

  // --- ESTADOS SVELTE 5 ---
  let isDraggingFile = $state(false);
  let showWireframe = $state(false);
  let showSkeleton = $state(true);
  let sidebarWidth = $state(320);
  let isResizing = $state(false);

  let searchQuery = $state("");
  let selectedIndex = $state(0);

  let modelStats = $state({
    name: "",
    meshes: 0,
    boneTree: [] as any[],
    flatBones: [] as string[],
  });
  let sections = $state({ stats: true, hierarchy: true, settings: true });

  let scene: THREE.Scene;
  let renderer: THREE.WebGLRenderer;
  let camera: THREE.PerspectiveCamera;
  let controls: OrbitControls;
  let currentModel: THREE.Group | null = null;
  let skeletonHelper = $state<THREE.SkeletonHelper | null>(null);

  // --- LÓGICA DE BÚSQUEDA Y AUTOCOMPLETADO ---
  let suggestions = $derived(
    searchQuery.length > 1
      ? modelStats.flatBones
          .filter((b) => b.toLowerCase().includes(searchQuery.toLowerCase()))
          .slice(0, 5)
      : [],
  );

  let filteredResults = $derived(
    searchQuery
      ? modelStats.flatBones.filter((b) =>
          b.toLowerCase().includes(searchQuery.toLowerCase()),
        )
      : [],
  );

  function handleKeyDown(e: KeyboardEvent) {
    if (suggestions.length === 0) return;
    if (e.key === "ArrowDown") {
      e.preventDefault();
      selectedIndex = (selectedIndex + 1) % suggestions.length;
    } else if (e.key === "ArrowUp") {
      e.preventDefault();
      selectedIndex =
        (selectedIndex - 1 + suggestions.length) % suggestions.length;
    } else if (e.key === "Tab" || e.key === "Enter") {
      e.preventDefault();
      searchQuery = suggestions[selectedIndex];
      selectedIndex = 0;
    }
  }

  // --- THREE.JS ---
  $effect(() => {
    const active = showWireframe;
    if (currentModel) {
      currentModel.traverse((child) => {
        if ((child as THREE.Mesh).isMesh) {
          const mat = (child as THREE.Mesh).material;
          if (Array.isArray(mat)) mat.forEach((m) => (m.wireframe = active));
          else mat.wireframe = active;
        }
      });
    }
  });

  $effect(() => {
    if (skeletonHelper) skeletonHelper.visible = showSkeleton;
  });

  onMount(() => {
    initThree();
    const ro = new ResizeObserver(() => handleResize());
    ro.observe(viewportContainer);
    return () => {
      renderer.dispose();
      ro.disconnect();
    };
  });

  function initThree() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color("#eef2f6");
    camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100);
    camera.position.set(0, 1.2, 2.5);
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    scene.add(new THREE.GridHelper(10, 20, "#cbd5e1", "#e2e8f0"));
    scene.add(new THREE.AmbientLight(0xffffff, 1.2));
    const dl = new THREE.DirectionalLight(0xffffff, 1.5);
    dl.position.set(5, 10, 7.5);
    scene.add(dl);
    handleResize();
    const animate = () => {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    };
    animate();
  }

  function handleResize() {
    if (!viewportContainer || !renderer) return;
    renderer.setSize(
      viewportContainer.clientWidth,
      viewportContainer.clientHeight,
      false,
    );
    camera.aspect =
      viewportContainer.clientWidth / viewportContainer.clientHeight;
    camera.updateProjectionMatrix();
  }

  function buildBoneTree(object: THREE.Object3D): any[] {
    const roots: any[] = [];
    object.traverse((node) => {
      if ((node as THREE.Bone).isBone) {
        const parent = node.parent;
        if (!parent || !(parent as THREE.Bone).isBone)
          roots.push(mapBone(node as THREE.Bone));
      }
    });
    return roots;
  }

  function mapBone(bone: THREE.Bone): any {
    return {
      name: bone.name,
      children: bone.children
        .filter((child) => (child as THREE.Bone).isBone)
        .map((child) => mapBone(child as THREE.Bone)),
    };
  }

  // Reemplaza tu función handleDrop con esta versión "Segura"
  async function handleDrop(e: DragEvent) {
    e.preventDefault();
    isDraggingFile = false;
    const file = e.dataTransfer?.files[0];

    // Aceptamos ambos formatos
    if (
      !file ||
      !(
        file.name.toLowerCase().endsWith(".vrm") ||
        file.name.toLowerCase().endsWith(".glb")
      )
    )
      return;

    const reader = new FileReader();
    reader.onload = (ev) => {
      const loader = new GLTFLoader();
      // Registramos el plugin (esto permite leer la física si existe)
      loader.register((parser) => new VRMLoaderPlugin(parser));

      loader.parse(ev.target?.result as ArrayBuffer, "", (gltf) => {
        if (currentModel) scene.remove(currentModel);
        if (skeletonHelper) scene.remove(skeletonHelper);

        // Detectamos si es VRM o escena estándar
        const vrm = gltf.userData.vrm;
        currentModel = vrm ? vrm.scene : gltf.scene;

        scene.add(currentModel);
        skeletonHelper = new THREE.SkeletonHelper(currentModel);
        scene.add(skeletonHelper);

        // Actualizamos stats para la sidebar
        let meshes = 0;
        let flat: string[] = [];
        currentModel.traverse((n) => {
          if ((n as THREE.Mesh).isMesh) meshes++;
          if ((n as THREE.Bone).isBone) flat.push(n.name);
        });

        modelStats = {
          name: file.name,
          meshes,
          boneTree: buildBoneTree(currentModel),
          flatBones: flat,
        };
      });
    };
    reader.readAsArrayBuffer(file);
  }
</script>

{#snippet renderBone(node, depth)}
  <div class="bone-node" style="padding-left: {depth * 14}px">
    <span class="tree-line">↳</span>
    {node.name || "Unnamed"}
  </div>
  {#each node.children as child}
    {@render renderBone(child, depth + 1)}
  {/each}
{/snippet}

<main class="inspector-app">
  <div
    bind:this={viewportContainer}
    class="viewport"
    class:dragging={isDraggingFile}
    ondragover={(e) => {
      e.preventDefault();
      isDraggingFile = true;
    }}
    ondragleave={(e) => {
      if (!e.currentTarget.contains(e.relatedTarget as Node))
        isDraggingFile = false;
    }}
    ondrop={handleDrop}
  >
    <canvas bind:this={canvas}></canvas>

    {#if !modelStats.name}
      <div class="hero-overlay">
        <div class="icon-blob"><Box size={32} /></div>
        <h2>Structural Inspector</h2>
        <p>Drop your VRM or GLB file to analyze hierarchy</p>
      </div>
    {/if}

    {#if isDraggingFile}
      <div class="drop-guidance">READY TO ANALYZE</div>
    {/if}
  </div>

  <div
    class="resizer"
    onmousedown={(e) => {
      isResizing = true;
      const move = (m: MouseEvent) => {
        sidebarWidth = window.innerWidth - m.clientX;
        handleResize();
      };
      const up = () => {
        isResizing = false;
        window.removeEventListener("mousemove", move);
        window.removeEventListener("mouseup", up);
      };
      window.addEventListener("mousemove", move);
      window.addEventListener("mouseup", up);
    }}
  >
    <div class="grip-container">
      <GripVertical size={14} color={isResizing ? "#6366f1" : "#94a3b8"} />
    </div>
  </div>

  <aside class="sidebar" style="width: {sidebarWidth}px">
    <header class="sb-header">
      <div class="title">
        <Settings size={14} /> <span>File Inspector</span>
      </div>
      <a href="https://andrewm09.github.io" class="exit">Exit</a>
    </header>

    <div class="sb-content">
      {#if modelStats.name}
        <section class="card">
          <button
            class="card-head"
            onclick={() => (sections.settings = !sections.settings)}
          >
            {#if sections.settings}<ChevronDown size={14} />{:else}<ChevronRight
                size={14}
              />{/if}
            Display Settings
          </button>
          {#if sections.settings}
            <div class="card-body">
              <label class="row"
                ><span>Wireframe</span>
                <input type="checkbox" bind:checked={showWireframe} /></label
              >
              <label class="row"
                ><span>Skeleton Helper</span>
                <input type="checkbox" bind:checked={showSkeleton} /></label
              >
            </div>
          {/if}
        </section>

        <section class="card">
          <div class="card-static-head"><Activity size={14} /> File Data</div>
          <div class="card-body">
            <div class="row">
              <span class="lbl">File:</span>
              <span class="val truncate">{modelStats.name}</span>
            </div>
            <div class="row">
              <span class="lbl">Meshes:</span>
              <span class="val">{modelStats.meshes}</span>
            </div>
            <div class="row">
              <span class="lbl">Total Bones:</span>
              <span class="val">{modelStats.flatBones.length}</span>
            </div>
          </div>
        </section>

        <section class="card hierarchy-card">
          <div class="card-static-head">Structural Hierarchy</div>

          <div class="search-section">
            <div class="search-input-wrapper">
              <Search size={14} class="s-icon" />
              <input
                type="text"
                placeholder="Search bone or object..."
                bind:value={searchQuery}
                onkeydown={handleKeyDown}
                class="search-bar"
              />
              {#if searchQuery}
                <button onclick={() => (searchQuery = "")} class="clear-btn"
                  ><X size={12} /></button
                >
              {/if}
            </div>

            {#if suggestions.length > 0}
              <div class="tokenizer-results">
                {#each suggestions as sug, i}
                  <button
                    class="token-item"
                    class:active={i === selectedIndex}
                    onclick={() => {
                      searchQuery = sug;
                      selectedIndex = 0;
                    }}
                  >
                    {sug} <span class="tab-key">Tab</span>
                  </button>
                {/each}
              </div>
            {/if}
          </div>

          <div class="hierarchy-view">
            {#if searchQuery}
              {#each filteredResults as bone}
                <button class="search-pill">
                  <span class="pill-prefix">🔍</span>
                  {bone}
                </button>
              {:else}
                <div class="no-match">
                  <X size={20} />
                  <p>No objects match your search</p>
                </div>
              {/each}
            {:else}
              <div class="bone-tree">
                {#each modelStats.boneTree as root}
                  {@render renderBone(root, 0)}
                {/each}
              </div>
            {/if}
          </div>
        </section>
      {:else}
        <div class="welcome-hint">
          Drop a model to inspect its internal structure
        </div>
      {/if}
    </div>

    <footer class="sb-footer">
      <div class="privacy-notice">
        <ShieldCheck size={12} /> Privacy First: No data is uploaded.
      </div>
      <div class="processing-disclaimer">
        Your models never leave your computer. All processing happens locally in
        your browser.
      </div>
      <p>© 2026 Andres Martin</p>
      <a
        href="https://github.com/andrewm09/andrewm09.github.io/tree/main/src/routes/vrm-inspector"
        target="_blank"
        class="source-link"
      >
        <Github size={12} /> View Source Code
      </a>
    </footer>
  </aside>
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: "Inter", sans-serif;
    background: #eef2f6;
    color: #334155;
    overflow: hidden;
  }
  .inspector-app {
    display: flex;
    width: 100vw;
    height: 100vh;
  }

  .viewport {
    flex: 1;
    position: relative;
  }
  canvas {
    width: 100%;
    height: 100%;
    display: block;
    outline: none;
  }

  /* FEEDBACK VISUAL */
  .hero-overlay {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;
    pointer-events: none;
    opacity: 0.6;
  }
  .icon-blob {
    width: 56px;
    height: 56px;
    background: #fff;
    border-radius: 16px;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0 auto 1rem;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
    color: #6366f1;
  }

  .drop-guidance {
    position: absolute;
    inset: 0;
    background: rgba(99, 102, 241, 0.1);
    border: 3px dashed #6366f1;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #6366f1;
    font-weight: 800;
    font-size: 1.5rem;
    z-index: 100;
    pointer-events: none;
  }

  /* CENTERED RESIZER */
  .resizer {
    width: 6px;
    cursor: col-resize;
    background: #e2e8f0;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 10;
    transition: background 0.2s;
  }
  .resizer:hover {
    background: #cbd5e1;
  }
  .grip-container {
    display: flex;
    align-items: center;
    height: 100%;
  }

  /* SIDEBAR */
  .sidebar {
    background: #fff;
    border-left: 1px solid #e2e8f0;
    display: flex;
    flex-direction: column;
    box-shadow: -10px 0 30px rgba(0, 0, 0, 0.02);
  }
  .sb-header {
    padding: 12px 16px;
    border-bottom: 1px solid #f1f5f9;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-weight: 800;
    font-size: 0.7rem;
    text-transform: uppercase;
    color: #64748b;
  }
  .exit {
    font-size: 0.7rem;
    color: #94a3b8;
    text-decoration: none;
    font-weight: 600;
  }

  .sb-content {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .card {
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    overflow: hidden;
  }
  .card-head,
  .card-static-head {
    width: 100%;
    padding: 10px 14px;
    font-size: 0.7rem;
    font-weight: 700;
    text-transform: uppercase;
    color: #64748b;
    background: #fff;
    border-bottom: 1px solid #f1f5f9;
    display: flex;
    align-items: center;
    gap: 6px;
  }
  .card-head {
    cursor: pointer;
    border: none;
    text-align: left;
  }
  .card-head:hover {
    background: #f8fafc;
  }

  .card-body {
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-size: 0.8rem;
  }
  .lbl {
    color: #64748b;
    font-weight: 500;
  }
  .val {
    font-family: monospace;
    font-weight: 600;
    color: #1e293b;
  }
  .truncate {
    max-width: 150px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* HIERARCHY & SEARCH */
  .hierarchy-card {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 350px;
  }
  .search-section {
    padding: 12px;
    background: #fff;
    border-bottom: 1px solid #f1f5f9;
    position: relative;
  }
  .search-input-wrapper {
    position: relative;
    display: flex;
    align-items: center;
  }
  .s-icon {
    position: absolute;
    left: 10px;
    color: #94a3b8;
  }
  .search-bar {
    width: 100%;
    background: #f1f5f9;
    border: 1px solid #e2e8f0;
    padding: 8px 32px;
    border-radius: 6px;
    font-size: 0.75rem;
    outline: none;
    transition: border-color 0.2s;
  }
  .search-bar:focus {
    border-color: #6366f1;
    background: #fff;
  }
  .clear-btn {
    position: absolute;
    right: 8px;
    border: none;
    background: #cbd5e1;
    color: #fff;
    border-radius: 50%;
    width: 16px;
    height: 16px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
  }

  .tokenizer-results {
    position: absolute;
    top: 100%;
    left: 12px;
    right: 12px;
    background: #fff;
    border: 1px solid #6366f1;
    border-radius: 0 0 6px 6px;
    z-index: 200;
    box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
  }
  .token-item {
    width: 100%;
    padding: 8px 12px;
    border: none;
    background: none;
    text-align: left;
    font-size: 0.7rem;
    cursor: pointer;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .token-item.active {
    background: #6366f1;
    color: #fff;
  }
  .tab-key {
    font-size: 0.6rem;
    opacity: 0.5;
    border: 1px solid currentColor;
    padding: 1px 4px;
    border-radius: 3px;
  }

  .hierarchy-view {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
    background: #fff;
  }

  /* TREE & LIST STYLES */
  .bone-node {
    font-family: monospace;
    font-size: 0.7rem;
    padding: 4px 0;
    color: #475569;
    border-bottom: 1px solid #f8fafc;
  }
  .bone-node:hover {
    background: rgba(99, 102, 241, 0.05);
    color: #6366f1;
    cursor: pointer;
  }
  .tree-line {
    color: #6366f1;
    opacity: 0.3;
    margin-right: 4px;
  }

  .search-pill {
    width: 100%;
    padding: 8px 12px;
    background: #f8fafc;
    border-radius: 6px;
    font-family: monospace;
    font-size: 0.7rem;
    margin-bottom: 4px;
    transition: all 0.2s;
    cursor: pointer;
    border: 1px solid transparent;
    text-align: left;
  }
  .search-pill:hover {
    background: #6366f1;
    color: #fff;
    transform: translateX(4px);
  }
  .pill-prefix {
    opacity: 0.4;
    margin-right: 8px;
  }

  .no-match {
    text-align: center;
    padding: 40px 10px;
    color: #cbd5e1;
  }
  .no-match p {
    font-size: 0.75rem;
    font-style: italic;
    margin-top: 8px;
  }

  .sb-footer {
    padding: 16px;
    border-top: 1px solid #f1f5f9;
    background: #f8fafc;
  }
  .privacy-notice {
    font-size: 0.65rem;
    color: #10b981;
    font-weight: 700;
    display: flex;
    align-items: center;
    gap: 4px;
    margin-bottom: 4px;
  }
  .processing-disclaimer {
    font-size: 0.65rem;
    color: #64748b;
    margin-bottom: 8px;
    line-height: 1.4;
  }
  .sb-footer p {
    font-size: 0.65rem;
    color: #94a3b8;
    margin: 0;
  }

  .welcome-hint {
    font-size: 0.75rem;
    color: #94a3b8;
    text-align: center;
    margin-top: 40px;
    font-style: italic;
  }

  .sb-footer {
    padding: 16px;
    border-top: 1px solid #f1f5f9;
    font-size: 0.65rem;
  }
  .source-link {
    display: flex;
    align-items: center;
    gap: 6px;
    color: #6366f1;
    text-decoration: none;
    margin-top: 8px;
    font-weight: 600;
  }
</style>
