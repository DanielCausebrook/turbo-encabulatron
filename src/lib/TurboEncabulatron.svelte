<script lang="ts">
    import {json5, json5ParseLinter} from 'codemirror-json5';
    import {wrappedLineIndent} from 'codemirror-wrapped-line-indent';
    import {basicSetup} from "codemirror";
    import {linter} from '@codemirror/lint';
    import {EditorView} from "@codemirror/view";
    import {EditorState} from '@codemirror/state';
    import defaultConfig from './default-config.json5?raw';
    import {onMount} from "svelte";
    import JSON5 from "json5";
    import playIcon from '@tabler/icons/filled/player-play.svg';

    let {
        animationDelayMs = 50,
        generationDelayMs = 100,
    } = $props();

    function delayMs(delayMs: number): Promise<void> {
        return new Promise<void>((resolve) => {
            setTimeout(resolve, delayMs);
        })
    }

    class State {
        value = "";
        private onUpdate: (newValue: string) => Promise<void>;

        constructor(onUpdate: (newValue: string) => Promise<void> = async () => {}) {
            this.onUpdate = onUpdate;
        }

        async notifyUpdate(intermediateValue: string): Promise<void> {
            await this.onUpdate(intermediateValue);
        }

        async append(component: Component): Promise<string> {
            let subState = new State(async str => await this.onUpdate(this.value + str));
            await component.evaluate(subState);
            this.value += subState.value;
            return this.value;
        }

        async apply(component: Component): Promise<string> {
            await component.evaluate(this);
            return this.value;
        }
    }

    interface Component {
        evaluate(state: State): Promise<void>;
    }

    class StringComponent implements Component {
        string: string;

        constructor(string: string) {
            this.string = string;
        }

        async evaluate(state: State): Promise<void> {
            state.value += this.string;
            await state.notifyUpdate(state.value);
        }
    }

    class ChainComponent implements Component {
        components: Component[];

        constructor(components: Component[]) {
            this.components = components;
        }

        static fromConfig(data: any): ChainComponent {
            if (Array.isArray(data)) {
                return new ChainComponent(
                    data.map((componentJson: any) => componentFromConfig(componentJson))
                );
            } else {
                throw new Error("Chain Component's data must be an array.");
            }
        }

        async evaluate(state: State): Promise<void> {
            for (const component of this.components) {
                await state.apply(component);
            }
        }
    }

    class RandomComponent implements Component {
        components: Component[];

        constructor(components: Component[]) {
            this.components = components;
        }

        static fromConfig(data: any): RandomComponent {
            if (Array.isArray(data)) {
                return new RandomComponent(
                    data.map((componentJson: any) => componentFromConfig(componentJson))
                );
            } else {
                throw new Error("Random Component's data must be an array.");
            }
        }

        async evaluate(state: State): Promise<void> {
            await state.append(this.components[Math.floor(Math.random() * this.components.length)]);
        }
    }

    class RandomWeightedComponent implements Component {
        private components: {component: Component, probability: number}[];

        constructor(components: {component: Component, weight: number}[]) {
            let totalWeight = 0;
            for (const componentData of components) {
                if (componentData.weight < 0) {
                    throw new Error("Weight must be positive.");
                }
                totalWeight += componentData.weight;
            }
            if (totalWeight <= 0) {
                throw new Error("Total weight must be greater than 0.");
            }
            this.components = components.map(data => ({
                component: data.component,
                probability: data.weight/totalWeight,
            }));
        }

        static fromConfig(data: any): RandomWeightedComponent {
            if (Array.isArray(data)) {
                return new RandomWeightedComponent(
                    data.flatMap((entry: any) => {
                        if (Array.isArray(entry) && entry.length >= 1) {
                            const weight = entry[0];
                            const results = [];
                            for (let i = 1; i < entry.length; i++) {
                                results.push({
                                    component: componentFromConfig(entry[i]),
                                    weight: weight,
                                });
                            }
                            return results;
                        } else {
                            throw new Error("Random Weighted Component's data must contain only arrays of length 2 or higher.");
                        }
                    })
                );
            } else {
                throw new Error("Random Weighted Component's data must be an array.");
            }
        }

        async evaluate(state: State): Promise<void> {
            let randomValue = Math.random();
            for (const componentData of this.components) {
                randomValue -= componentData.probability;
                if (randomValue <= 0) {
                    await state.append(componentData.component);
                    return;
                }
            }
            throw new Error();
        }
    }

    class ReplaceComponent implements Component {
        replacements: {pattern: RegExp, replacement: Component}[];

        constructor(replacements: {pattern: RegExp, replacement: Component}[]) {
            this.replacements = replacements;
        }

        static fromConfig(data: any): ReplaceComponent {
            if (Array.isArray(data)) {
                return new ReplaceComponent(
                    data.map((replacement: any) => {
                        if (
                            Array.isArray(replacement)
                            && replacement.length === 2
                        ) {
                            return {
                                pattern: new RegExp(replacement[0], 'g'),
                                replacement: componentFromConfig(replacement[1]),
                            };
                        } else {
                            throw new Error(`Replace Component's data must contain only 2-length arrays.`);
                        }
                    })
                );
            } else {
                throw new Error("Replace Component's data must be an array.");
            }
        }

        async evaluate(state: State): Promise<void> {
            for (const replacementData of this.replacements) {
                let replacementStr = await new State().apply(replacementData.replacement);
                let newValue = state.value.replace(replacementData.pattern, replacementStr);
                if (state.value !== newValue) {
                    state.value = newValue;
                    await state.notifyUpdate(state.value);
                }
            }
        }
    }

    class CapitaliseComponent implements Component {
        async evaluate(state: State): Promise<void> {
            let newValue = state.value.replace(/(?:^|\s|["'([{])+\S/g, match => match.toUpperCase());
            if (newValue !== state.value) {
                state.value = newValue;
                await state.notifyUpdate(state.value);
            }
        }
    }

    function componentFromConfig(componentJson: any): Component {
        if (
            typeof componentJson === 'object'
            && componentJson.hasOwnProperty('type')
        ) {
            const getData = () => {
                if (componentJson.hasOwnProperty('data')) {
                    return componentJson.data;
                } else {
                    throw new Error(`Component "${componentJson.type}" must have property "data".`);
                }
            };
            switch(componentJson.type) {
                case 'chain': return ChainComponent.fromConfig(getData());
                case 'random': return RandomComponent.fromConfig(getData());
                case 'random-weighted': return RandomWeightedComponent.fromConfig(getData());
                case 'replace': return ReplaceComponent.fromConfig(getData());
                case 'capitalise': return new CapitaliseComponent();
                default:
                    throw new Error(`Unknown component type "${componentJson.type}"`);
            }
        } else if (typeof componentJson === 'string') {
            return new StringComponent(componentJson);
        } else {
            throw new Error("Component must be object or string.");
        }
    }

    let configStr: string = $state("");
    let loadedComponent: Component|null = null;


    let outputElem: HTMLElement;
    let newMarker: HTMLElement;
    const maxCapacity = 500;

    function createOutputBlocks(count: number): HTMLElement[] {
        outputElem.removeChild(newMarker);

        let toRemove = outputElem.children.length + count - maxCapacity;
        while (toRemove-- > 0) {
            outputElem.lastChild?.remove();
        }

        outputElem.prepend(newMarker);

        const result: HTMLElement[] = [];
        for (let i = 0; i < count; i++) {
            let newElem = document.createElement('div');
            outputElem.prepend(newElem);
            result.push(newElem);
        }

        outputElem.scrollTop = outputElem.scrollHeight;

        return result;
    }

    async function generate(count: number) {
        let loadedComponentCache = loadedComponent;
        if (loadedComponentCache === null) {
            return;
        }

        createOutputBlocks(count).forEach((block, i) => {
            let state = new State(async str => {
                block.innerText = str;
                await delayMs(animationDelayMs);
            });
            delayMs(i*generationDelayMs).then(() => state.apply(loadedComponentCache));
        });
    }

    let showConfig: boolean = $state(false);
    let editorWrapper: HTMLElement;
    let editorView: EditorView;

    async function loadConfig() {
        let oldComponent = loadedComponent;
        loadedComponent = null;

        let block = createOutputBlocks(1)[0];
        block.classList.add('system');

        block.innerText = 'Loading program';
        await delayMs(200);
        block.innerText += '.';
        await delayMs(200);
        block.innerText += '.';
        await delayMs(200);
        block.innerText += '.';
        await delayMs(300);
        let configStr = editorView.state.doc.toString();
        let config;
        try {
            config = JSON5.parse(configStr);
        } catch (e) {
            block.innerText = 'Loading program... ERR\nProgram is not valid JSON5.';
            loadedComponent = oldComponent;
            return;
        }
        let newComponent;
        try {
            newComponent = componentFromConfig(config);
        } catch (e) {
            if (e instanceof Error) {
                block.innerText = 'Loading program... ERR\n' + e.message;
                loadedComponent = oldComponent;
                return;
            } else {
                loadedComponent = oldComponent;
                throw e;
            }
        }
        block.innerText = 'Loading program... OK';
        await delayMs(400);
        loadedComponent = newComponent;
        generate(5);
    }

    onMount(async () => {
        configStr = defaultConfig;

        editorView = new EditorView({
            parent: editorWrapper,
            state: EditorState.create({
                doc: configStr,
                extensions: [
                    basicSetup,
                    EditorView.lineWrapping,
                    wrappedLineIndent,
                    json5(),
                    linter(json5ParseLinter()),
                ],
            }),
        });

        loadConfig();
    })

    // CSS backgrounds
    // background-image: /* bg10 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAGklEQVQY02NgwA3+//+PwcIPSFc3FE1mYAAAGhod428P9TcAAAAASUVORK5CYII=);
    // background-image: /* bg20 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAIAAAACDbGyAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAFklEQVQI12NgQAL///9HotAAMZIMDABWFg7ySLsWPAAAAABJRU5ErkJggg==);
    // background-image: /* bg30 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAOUlEQVQY02P4//8/AxJA42KX+A8G6BIQBrI6JkxjGBkZ0c3DdAFUhCinobiFoMlwBhN2t+A3HM4GAGNdR8MIuJkVAAAAAElFTkSuQmCC);
    // background-image: /* bg70 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAARklEQVQY042OwQ4AIAhCwf//Z3OjnJmHPMHEJwTg7jhD8rJp6iK0zqwaCeUUNTxTGRvYXybvq1rGW45qNAJCWOvSas7wBCzyOkf8owvCOAAAAABJRU5ErkJggg==);
    // background-image: /* bg80 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAIAAAACDbGyAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAJUlEQVQI12P8//8/AwwwMjIyQSgIHyjHBKHgooxwPgQwIUsCAQB9YA8GvWxUKQAAAABJRU5ErkJggg==);
    // background-image: /* bg90 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAOklEQVQY032MQRIAIAgCg///mTo0joXqiZFlIWk1B4CRvD5LRio5+iJzHMy37sy/3Dnkrx8H8yMvuQ2ihh4Q750yJgAAAABJRU5ErkJggg==);
</script>
<div class="widget">
    <div class="machine-border">
        <div class="machine">
            <div class="header">
                <h2>TURBO-ENCABULATRON 5000</h2>
            </div>
            <div class="output-border">
                <div class="output" bind:this={outputElem}>
                    <div class="new-marker" bind:this={newMarker}></div>
                </div>
            </div>
            <div class="controls">
                <div class="generate">
                    <div class="label">GENERATE:</div>
                    <div class="buttons">
                        <button type="button" onclick={() => generate(1)}><span>1</span></button>
                        <button type="button" onclick={() => generate(5)}><span>5</span></button>
                        <button type="button" onclick={() => generate(10)}><span>10</span></button>
                    </div>
                </div>
                <div class="config">
                    <button type="button" onclick={() => showConfig = !showConfig}><span>PROG</span></button>
                </div>
            </div>
        </div>
    </div>
    <div class="config-pane" class:hide={!showConfig}>
        <div class="header">
            <h3>PROGRAM DATA</h3>
            <button type="button" class="icon" onclick={loadConfig}>
                <img src={playIcon} alt="Load" />
            </button>
        </div>
        <div class="editor" bind:this={editorWrapper}></div>
    </div>
</div>
<style>
    :root {
        scrollbar-color: black white;
        position: relative;
        z-index: 0;
    }
    button {
        position: relative;
        background: none;
        color: inherit;
        padding: 5px 10px;
        border: 0;
        font: inherit;
        cursor: pointer;
        outline: inherit;
        z-index: 0;

        &:not(.icon) {
            border: 1px solid black;
            --bevel-width: 3px;

            &:hover {
                &::before {
                    display: block;
                    background-image: /* bg90 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAOklEQVQY032MQRIAIAgCg///mTo0joXqiZFlIWk1B4CRvD5LRio5+iJzHMy37sy/3Dnkrx8H8yMvuQ2ihh4Q750yJgAAAABJRU5ErkJggg==);
                }
                &::after {
                    background-image: /* bg30 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAOUlEQVQY02P4//8/AxJA42KX+A8G6BIQBrI6JkxjGBkZ0c3DdAFUhCinobiFoMlwBhN2t+A3HM4GAGNdR8MIuJkVAAAAAElFTkSuQmCC);
                }
            }

            &:active {
                &::before {
                    display: block;
                    background-image: /* bg70 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAARklEQVQY042OwQ4AIAhCwf//Z3OjnJmHPMHEJwTg7jhD8rJp6iK0zqwaCeUUNTxTGRvYXybvq1rGW45qNAJCWOvSas7wBCzyOkf8owvCOAAAAABJRU5ErkJggg==);
                }
                &::after {
                    display: none;
                }
                > span {
                    transform: translate(1px, 1px);
                }
            }

            &::before, &::after {
                position: absolute;
                top: 0;
                bottom: 0;
                left: 0;
                right: 0;
                image-rendering: pixelated;
                content: '';
            }

            &::before {
                display: none;
                clip-path: content-box polygon(
                        0% 0%,
                        100% 0%,
                        calc(100% - var(--bevel-width)) var(--bevel-width),
                        var(--bevel-width) var(--bevel-width),
                        var(--bevel-width) calc(100% - var(--bevel-width)),
                        0% 100%
                );
            }

            &::after {
                background-image: /* bg70 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAARklEQVQY042OwQ4AIAhCwf//Z3OjnJmHPMHEJwTg7jhD8rJp6iK0zqwaCeUUNTxTGRvYXybvq1rGW45qNAJCWOvSas7wBCzyOkf8owvCOAAAAABJRU5ErkJggg==);
                clip-path: content-box polygon(
                        100% 100%,
                        0% 100%,
                        var(--bevel-width) calc(100% - var(--bevel-width)),
                        calc(100% - var(--bevel-width)) calc(100% - var(--bevel-width)),
                        calc(100% - var(--bevel-width)) var(--bevel-width),
                        100% 0%
                );
            }

            > span {
                display: block;
            }
        }

        &.icon {
            line-height: 0;

            &:hover {
                &::before {
                    position: absolute;
                    left: 1px;
                    right: 1px;
                    top: -1px;
                    bottom: -1px;
                    border: 0 dotted black;
                    border-top-width: 2px;
                    border-bottom-width: 2px;
                    content: '';
                }
            }
            &:active {
                &::before {
                    top: 1px;
                    bottom: 1px;
                }
            }
        }
    }

    .widget {
        --machine-shadow-offset: 8px;
        display: flex;
        flex-flow: column nowrap;
        position: relative;
        min-width: 300px;
        min-height: 0;
        padding: var(--machine-shadow-offset);
        padding-top: 0;
        z-index: 0;
    }

    .machine-border {
        position: relative;
        padding: 3px;
        background-image: /* bg10 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAGklEQVQY02NgwA3+//+PwcIPSFc3FE1mYAAAGhod428P9TcAAAAASUVORK5CYII=);
        image-rendering: pixelated;
        border: 1px solid white;

        &::before {
            position: absolute;
            top: 0;
            bottom: 0;
            left: 0;
            right: 0;
            background-image: /* bg70 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAARklEQVQY042OwQ4AIAhCwf//Z3OjnJmHPMHEJwTg7jhD8rJp6iK0zqwaCeUUNTxTGRvYXybvq1rGW45qNAJCWOvSas7wBCzyOkf8owvCOAAAAABJRU5ErkJggg==);
            image-rendering: pixelated;
            transform: translate(var(--machine-shadow-offset), var(--machine-shadow-offset));
            z-index: -1;
            content: '';
        }
    }

    .machine {
        display: flex;
        flex-flow: column nowrap;
        align-items: stretch;
        gap: 10px;
        padding: 15px;
        background: black;

        > .header {
            display: flex;
            flex-flow: row nowrap;
            align-items: stretch;
            padding: 5px 10px;
            transform: rotate(-1.5deg);

            > h2 {
                flex: 0 1 auto;
                margin: 0;
                font-style: oblique;
                background: black;
                border-top: 1px solid white;
                border-bottom: 1px solid white;
                color: white;
                text-align: center;
            }

            &::before {
                flex: 1 1 auto;
                border-top: 1px solid white;
                content: '';
            }
            &::after {
                flex: 1 1 auto;
                border-bottom: 1px solid white;
                content: '';
            }
        }

        .output-border {
            padding: 4px;
            background-image: /* bg20 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAIAAAACDbGyAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAFklEQVQI12NgQAL///9HotAAMZIMDABWFg7ySLsWPAAAAABJRU5ErkJggg==);
            image-rendering: pixelated;
        }

        .output {
            flex: 1 1 auto;
            display: flex;
            flex-flow: column-reverse nowrap;
            align-items: stretch;
            height: 300px;
            overflow-y: scroll;
            contain: strict;
            background-image: /* bg80 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAIAAAACDbGyAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAJUlEQVQI12P8//8/AwwwMjIyQSgIHyjHBKHgooxwPgQwIUsCAQB9YA8GvWxUKQAAAABJRU5ErkJggg==);
            image-rendering: pixelated;

            > :global(:not(.new-marker)) {
                flex: 0 0 min-content;
                position: relative;
                margin: 2px 4px;
                padding: 2px 4px;
                background: white;
                min-height: 1lh;

                &.system {
                    border: 0 solid black;
                    border-right-width: 3px;
                    border-left-width: 3px;
                    padding: 4px 8px;
                    margin: 6px 4px;
                }
            }

            > .new-marker {
                margin: 2px 4px;
                border-top: 1px dotted black;
            }
        }
        .controls {
            display: flex;
            flex-flow: row nowrap;
            align-items: center;
            justify-content: space-between;
            gap: 10px;
            background: black;
            color: white;

            > .generate {
                flex: 3 1 auto;
                display: flex;
                flex-flow: row wrap;
                align-items: baseline;
                justify-content: center;
                gap: 10px;

                .label {
                    font-weight: bold;
                }
                .buttons {
                    display: flex;
                    flex-flow: row nowrap;
                    gap: 3px;

                    button {
                        width: 50px;
                    }
                }
            }
            .config {
                flex: 1 1 auto;
                display: flex;
                flex-flow: row nowrap;
                justify-content: flex-end;
                button {
                    width: 60px;
                }
            }
            button {
                flex: none;
                background: white;
                color: black;
            }
        }
    }
    .config-pane {
        flex: 0 1 80vh;
        display: flex;
        flex-flow: column nowrap;
        gap: 10px;
        min-height: 250px;
        position: relative;
        background-image: /* bg80 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAIAAAACDbGyAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAJUlEQVQI12P8//8/AwwwMjIyQSgIHyjHBKHgooxwPgQwIUsCAQB9YA8GvWxUKQAAAABJRU5ErkJggg==);
        image-rendering: pixelated;
        margin: -4px 10px 0;
        z-index: 1;
        padding: 10px;

        &.hide {
            display: none;
        }

        &::before {
            position: absolute;
            top: 0;
            height: 3px;
            left: 0;
            right: 0;
            background-image: /* bg70 */ url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAIAAAACUFjqAAAACXBIWXMAAC4jAAAuIwF4pT92AAAARklEQVQY042OwQ4AIAhCwf//Z3OjnJmHPMHEJwTg7jhD8rJp6iK0zqwaCeUUNTxTGRvYXybvq1rGW45qNAJCWOvSas7wBCzyOkf8owvCOAAAAABJRU5ErkJggg==);
            background-position: 0 3px;
            image-rendering: pixelated;
            content: '';
        }

        .header {
            display: flex;
            flex-flow: row nowrap;
            align-items: center;
            gap: 15px;
            padding: 3px 6px;

            h3 {
                flex: 1 1 auto;
                margin: 0;
                font-size: 135%;
            }
        }

        .editor {
            flex: 1 1 auto;
            background: white;
            overflow: clip;
            contain: strict;

            :global(.cm-editor) {
                height: 100%;
                overflow: hidden;
            }
            :global(.cm-scroller) {
                height: 100%;
                overflow: auto;
                font-family: inherit;
            }
        }
    }
</style>