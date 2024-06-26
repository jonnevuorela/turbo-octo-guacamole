<script>
    import { onMount } from "svelte";
    import { invoke } from "@tauri-apps/api";
    import { createEventDispatcher } from "svelte";

    const dispatch = createEventDispatcher();

    const passiveMode = "passive";
    const inputMode = "text_area_visible";
    let currentState = passiveMode;

    let isPanning = false;
    let startPanPosition = { x: 0, y: 0 };
    let panOffset = { x: 0, y: 0 };

    let isDragging = false;
    let mouseOffset = { x: 0, y: 0 };
    let mousePosition = { x: 0, y: 0 };

    let isAnimating = false;

    let isConnecting = false;
    let idConnecting = null;
    let connectingThought = null;
    let trackX = 0;
    let trackY = 0;

    let thoughts = [];
    let selectedThought = null;
    let activeIndex = null;
    let title = "";

    let mind;
    let scale = 1;
    let mouseX = 0;
    let mouseY = 0;

    let targetTranslate = { x: 0, y: 0 };
    let currentTranslate = { x: 0, y: 0 };
    let isMovingToTarget = false;

    const panSpeed = 0.8;
    const zoomSpeed = 0.08;

    onMount(async () => {
        try {
            document.addEventListener("mouseup", function () {
                isDragging = false;
            });
            const response = await invoke("read_json");
            thoughts = JSON.parse(response);
            thoughts.forEach((thought) => {
                if (thought.x === undefined) thought.x = 0;
                if (thought.y === undefined) thought.y = 0;
            });
            console.log(
                `thoughts in mind component from invoke:`,
                JSON.stringify(thoughts),
            );
        } catch (error) {
            console.log("Error invoking thoughts", error);
        }
    });

    function findThoughtIndexById(activeThought) {
        return thoughts.findIndex((thought) => thought.id === activeThought);
    }

    function checkBorders() {
        const mindRect = mind.getBoundingClientRect();
        const viewportWidth = window.innerWidth;
        const viewportHeight = window.innerHeight;

        if (mindRect.left > 0) {
            targetTranslate.x -= mindRect.left;
        }
        if (mindRect.right < viewportWidth) {
            targetTranslate.x += viewportWidth - mindRect.right;
        }
        if (mindRect.top > 0) {
            targetTranslate.y -= mindRect.top;
        }
        if (mindRect.bottom < viewportHeight) {
            targetTranslate.y += viewportHeight - mindRect.bottom;
        }
    }

    function handleWheel(e) {
        e.preventDefault();

        const mind = document.getElementById("mind");
        const style = window.getComputedStyle(mind);
        const matrix = new DOMMatrixReadOnly(style.transform);
        const currentTranslate = {
            x: matrix.m41,
            y: matrix.m42,
        };

        const oldScale = scale;
        scale *= 1 - e.deltaY * zoomSpeed;
        scale = Math.min(Math.max(scale, 0.125), 4);

        if (e.deltaY > 0) {
            targetTranslate.x = (currentTranslate.x * scale) / oldScale;
            targetTranslate.y = (currentTranslate.y * scale) / oldScale;
        } else {
            // Scrolling in
            // Zoom in towards the cursor
            const viewportCenter = {
                x: window.innerWidth / 2,
                y: window.innerHeight / 2,
            };
            const mousePointTo = {
                x: e.clientX - viewportCenter.x - currentTranslate.x,
                y: e.clientY - viewportCenter.y - currentTranslate.y,
            };
            const scaleRatio = oldScale / scale;
            targetTranslate.x =
                currentTranslate.x - mousePointTo.x * (1 - scaleRatio);
            targetTranslate.y =
                currentTranslate.y - mousePointTo.y * (1 - scaleRatio);
        }

        checkBorders();

        if (!isMovingToTarget) {
            isMovingToTarget = true;
            isAnimating = true;
            moveToTarget(zoomSpeed);
        }
    }

    function moveToTarget(speed) {
        const dx = targetTranslate.x - currentTranslate.x;
        const dy = targetTranslate.y - currentTranslate.y;

        currentTranslate.x += dx * speed;
        currentTranslate.y += dy * speed;

        mind.style.transform = `translate(${currentTranslate.x}px, ${currentTranslate.y}px) scale(${scale})`;

        if (Math.abs(dx) < 0.1 && Math.abs(dy) < 0.1) {
            isAnimating = false;
            isMovingToTarget = false;
            console.log("isAnimating", isAnimating);
        } else {
            // Continue the animation.
            window.requestAnimationFrame(() => moveToTarget(speed));
        }
    }

    function handleMouseDown(e) {
        e.preventDefault();
        if (e.target.classList.contains("thought")) {
            handleDragStart(e);
        } else {
            isPanning = true;
            startPanPosition = {
                x: e.clientX,
                y: e.clientY,
            };
        }
    }

    function handleMouseMove(e) {
        const adjustedMousePosition = {
            x: e.clientX / scale,
            y: e.clientY / scale,
        };
        const svgRect = mind.getBoundingClientRect();
        trackX = (e.clientX - svgRect.left) / scale;
        trackY = (e.clientY - svgRect.top) / scale;

        if (isPanning) {
            const dx = e.clientX - startPanPosition.x;
            const dy = e.clientY - startPanPosition.y;
            startPanPosition = { x: e.clientX, y: e.clientY };
            panMind(dx, dy);
        } else if (isDragging) {
            handleDragMove(adjustedMousePosition);
        }
    }

    function handleMouseUp() {
        isPanning = false;
    }

    function panMind(dx, dy) {
        const mind = document.getElementById("mind");
        const style = window.getComputedStyle(mind);
        const matrix = new DOMMatrixReadOnly(style.transform);

        targetTranslate.x = matrix.m41 + dx;
        targetTranslate.y = matrix.m42 + dy;

        checkBorders();

        if (!isMovingToTarget) {
            isMovingToTarget = true;
            moveToTarget(panSpeed);
        }
    }

    // listeners for mouse going out of window
    window.addEventListener("mouseout", (event) => {
        if (!event.relatedTarget && !event.toElement) {
            handleMouseUp();
        }
    });
    window.addEventListener("mousemove", (event) => {
        if (!(event.buttons & 1)) {
            handleMouseUp();
        }
    });
    window.addEventListener(
        "wheel",
        function (e) {
            e.preventDefault();
        },
        { passive: false },
    );

    async function handleDragStart(e) {
        e.stopPropagation();

        // same value as adjustedMousePosition in handleMouseMove, but this needs to be independent
        mouseX = e.clientX / scale;
        mouseY = e.clientY / scale;

        const activeThought = e.target.dataset.thoughtId;

        activeIndex = findThoughtIndexById(+activeThought);

        if (thoughts[activeIndex] != null) {
            isDragging = true;

            thoughts[activeIndex].x = e.target.offsetLeft;
            thoughts[activeIndex].y = e.target.offsetTop;

            thoughts[activeIndex].initialPosition = {
                x: thoughts[activeIndex].x,
                y: thoughts[activeIndex].y,
            };

            mouseOffset = {
                x: Math.round(mouseX - thoughts[activeIndex].x),
                y: Math.round(mouseY - thoughts[activeIndex].y),
            };
        }
    }

    async function handleDragMove(adjustedMousePosition) {
        if (isDragging) {
            mousePosition = {
                x: Number.parseInt(adjustedMousePosition.x),
                y: Number.parseInt(adjustedMousePosition.y),
            };

            // track position
            thoughts[activeIndex].el.style.left =
                `${mousePosition.x - mouseOffset.x}px`;
            thoughts[activeIndex].el.style.top =
                `${mousePosition.y - mouseOffset.y}px`;

            // update thoughts array
            thoughts[activeIndex].x = parseInt(
                thoughts[activeIndex].el.style.left,
            );
            thoughts[activeIndex].y = parseInt(
                thoughts[activeIndex].el.style.top,
            );

            const data = JSON.stringify(thoughts);
            await invoke("write_json", { data });
        }
    }

    function handleDragEnd() {
        isDragging = false;
    }

    function handleDoubleClick(e) {
        const toughtId = e.target.dataset.thoughtId;
        const thought = thoughts.find((thought) => thought.id === +toughtId);

        if (thought) {
            const centerX = window.innerWidth / 2;
            const centerY = window.innerHeight / 2;

            const thoughtCenterX = thought.x + 50;
            const thoughtCenterY = thought.y + 50;

            const translateX = centerX - thought.x * scale;
            const translateY = centerY - thought.y * scale;

            scale = 2;

            mind.style.transformOrigin = `${thoughtCenterX}px ${thoughtCenterY}px`;
            mind.style.transform = `translate(${translateX}px, ${translateY}px) scale(${scale})`;
        }
    }
    function handleClick(thoughtId) {
        let relatedId = thoughtId;
        let thought = thoughts.find((t) => t.id === thoughtId);
        let coordinates = { x: thought.x, y: thought.y };

        console.log("relatedId (from Mind)", relatedId);
        console.log("thoughts coordinates (from Mind)", thought.x, thought.y);
        console.log("------------------------");

        dispatch("add", { relatedId, coordinates });
    }

    /*async function handleClickOk(e) {
        console.log("okButtonClick");
        currentState = passiveMode;

        if (!selectedThought) {
            console.error("No thought selected");
            return;
        }
        console.log("selectedThought.id", selectedThought.id);

        const lastId = await invoke("get_last_id");
        const newId = lastId + 1;

        const jsonOutput = JSON.stringify([
            {
                title,
                id: newId,
                x: selectedThought.x,
                y: selectedThought.y + 100,
                relation_id: selectedThought.id,
            },
        ]);

        await invoke("write_json", { data: jsonOutput })
            .then(() => {
                console.log("Data passed to backend");
            })
            .catch((error) => {
                console.error("Error writing data", error);
            });
            }*/

    function connectStart(thoughtId) {
        isConnecting = true;
        connectingThought = thoughts.find((t) => t.id === thoughtId);
        window.addEventListener("drag", handleMouseMove);
    }

    function connectEnd() {
        isConnecting = false;
        connectingThought = null;
        window.removeEventListener("drag", handleMouseMove);
    }
</script>

<div
    id="mind"
    role="main"
    bind:this={mind}
    on:mousedown={handleMouseDown}
    on:mouseup={handleMouseUp}
    on:mousemove={handleMouseMove}
    on:wheel={handleWheel}
>
    {#each thoughts as thought (thought.id)}
        <div
            role="application"
            bind:this={thought.el}
            class="thought"
            style="left: {thought.x}px; top: {thought.y}px;"
            on:mousedown={handleDragStart}
            on:mouseup={handleDragEnd}
            on:dblclick={handleDoubleClick}
            on:drop={handleDrop(thought.id)}
            class:blurred={currentState === inputMode}
            data-thought-id={thought.id}
        >
            <button id="addRelated" on:click={handleClick(thought.id)}>
                <svg width="8" height="8">
                    <line x1="0" y1="4" x2="8" y2="4" stroke="white" />
                    <line x1="4" y1="0" x2="4" y2="8" stroke="white" />
                </svg>
            </button>
            <button
                id="connect"
                draggable="true"
                on:dragstart={() => connectStart(thought.id)}
                on:dragend={connectEnd}
            />

            <div id="thoughtTitle">
                {thought.title}
            </div>
        </div>
        {#if isConnecting && connectingThought}
            <svg
                class:blurred={currentState === inputMode}
                style="position:absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none;"
            >
                <line
                    x1={connectingThought.x}
                    y1={connectingThought.y}
                    x2={trackX}
                    y2={trackY}
                    stroke="white"
                />
            </svg>
        {/if}
        {#if thought.relation_id}
            <svg
                class:blurred={currentState === inputMode}
                style="position:absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none;"
            >
                <line
                    x1={thought.x + 50}
                    y1={thought.y + 50}
                    x2={thoughts.find((t) => t.id === thought.relation_id).x +
                        50}
                    y2={thoughts.find((t) => t.id === thought.relation_id).y +
                        50}
                    stroke="white"
                />
            </svg>
        {/if}
    {/each}
</div>

<style>
    .blurred {
        filter: blur(5px);
    }
    #inputWindow {
        position: fixed;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background-color: midnightblue;
        border: 0.1em groove black;
        border-radius: 0.5em;
        height: 6em;
        padding: 10px;
        z-index: 3;
    }
    #textInput {
        background-color: transparent;
        border: none;

        color: white;
        font-size: 1em;
        bottom: 0;
        left: 0;
        margin: 0 10px;
        width: 100%;
    }
    #inputOk {
        background-color: transparent;
        border: 1px solid white;
        color: white;
        font-size: 1em;
        bottom: 0;
        right: 0;
        margin: 10px;
        padding: 5px;
        width: fit-content;
        height: fit-content;
    }
    #inputCancel {
        background-color: transparent;
        border: 1px solid white;
        color: white;
        top: 0;
        left: 0;
        font-size: 1em;
        margin: 10px;
        padding: 5px;
        width: fit-content;
        height: fit-content;
    }

    #mind {
        position: absolute;
        background-color: black;
        width: 769vw;
        min-height: 769vh;
        transform-origin: 50% 50%;
        border: 1px solid white;
    }
    #mind:active {
        cursor: grabbing;
    }

    .thought {
        position: absolute;

        font: 1.5em sans-serif;
        width: 4em;
        height: 4em;
        background-color: maroon;
        z-index: 2;
    }
    #thoughtTitle {
        display: flex;
        flex-direction: column;
        background-color: transparent;
        color: white;
        font-size: 1em;
        margin: 0;
        padding: 0;
        width: 100%;
        text-align: center;
    }
    #addRelated {
        background-color: transparent;
        border: none;
        color: white;
        font-size: 1em;
        left: 0;
        top: 0;

        width: 1em;
        height: 1em;
    }
    #connect {
        border: solid 1px black;
        color: white;
        font-size: 1em;
        right: 0;
        top: 0;
        width: 1em;
        height: 1em;
    }
</style>
