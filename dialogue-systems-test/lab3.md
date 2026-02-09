# Lab 3 Setup From Scratch for Linux and WSL (Windows Subsystem for Linux) Users

Instead of yarn (what we think is the problematic part), we will only try and use npm.

Navigate to home folder (for us to easily troubleshoot when necessary):
```bash
cd
```

Node.js and npm should be installed on your system. Please check:
```bash
node -v
```
This should give an output, something like: v22.13.1
```bash
npm -v
```
This should give an output, something like: 11.8.0

You also need an Azure Speech Service KEY from any region, check the original Lab 3 documentation for info on that.

## Step 1: Create a New Vite Project

Create a new Vite project using the vanilla TypeScript template:
```bash
npm create vite@latest
```

- **Project name:** Code
- **Framework:** Select Vanilla
- **Variant:** Select TypeScript
- **Experimental Features:** No

Navigate into the project:
```bash
cd Code
```

## Step 2: Install Dependencies

First, we install the base dependencies that Vite created:
```bash
npm install
```

Then we install the additional packages needed for the lab:
```bash
npm install xstate
npm install speechstate@latest
npm install @statelyai/inspect
```

## Step 3: We Clean Up Default Files

Navigate to the `src` directory:
```bash
cd src
```

Remove the default Vite files that we won't need:
```bash
rm -f *
```

## Step 4: Create Project Files

Now we will create the source files for our lab. You can create these files using your preferred text editor.

### 4.1 Create `azure.ts`

This file will store your Azure credentials.

**File location:** `Code/src/azure.ts`
```typescript
export const KEY = "YOUR_KEY";
```

Replace `YOUR_KEY` with your actual Azure Speech Service key.

### 4.2 Create `types.ts`

This file contains TypeScript type definitions for our dm --> dialogue manager/machine.

**File location:** `Code/src/types.ts`
```typescript
import type { Hypothesis, SpeechStateExternalEvent } from "speechstate";
import type { ActorRef } from "xstate";

export interface DMContext {
  spstRef: ActorRef<any, any>;
  lastResult: Hypothesis[] | null;
}

export type DMEvents = SpeechStateExternalEvent | { type: "CLICK" } | { type: "DONE" };
```

### 4.3 Create `dm.ts`

This is our main dialogue manager/machine file with the state machine logic.

**File location:** `Code/src/dm.ts`
```typescript
import { assign, createActor, setup } from "xstate";
import type { Settings } from "speechstate";
import { speechstate } from "speechstate";
import { createBrowserInspector } from "@statelyai/inspect";
import { KEY } from "./azure";
import type { DMContext, DMEvents } from "./types";

const inspector = createBrowserInspector();

const azureCredentials = {
  endpoint:
    "https://YOUR_REGION.api.cognitive.microsoft.com/sts/v1.0/issuetoken",
  key: KEY,
};

const settings: Settings = {
  azureCredentials: azureCredentials,
  azureRegion: "YOUR_REGION",
  asrDefaultCompleteTimeout: 0,
  asrDefaultNoInputTimeout: 5000,
  locale: "en-US",
  ttsDefaultVoice: "en-US-DavisNeural",
};

interface GrammarEntry {
  person?: string;
  day?: string;
  time?: string;
}

const grammar: { [index: string]: GrammarEntry } = {
  vlad: { person: "Vladislav Maraev" },
  bora: { person: "Bora Kara" },
  tal: { person: "Talha Bedir" },
  tom: { person: "Tom Södahl Bladsjö" },
  monday: { day: "Monday" },
  tuesday: { day: "Tuesday" },
  "10": { time: "10:00" },
  "11": { time: "11:00" },
};

function isInGrammar(utterance: string) {
  return utterance.toLowerCase() in grammar;
}

function getPerson(utterance: string) {
  return (grammar[utterance.toLowerCase()] || {}).person;
}

const dmMachine = setup({
  types: {
    context: {} as DMContext,
    events: {} as DMEvents,
  },
  actions: {
    "spst.speak": ({ context }, params: { utterance: string }) =>
      context.spstRef.send({
        type: "SPEAK",
        value: {
          utterance: params.utterance,
        },
      }),
    "spst.listen": ({ context }) =>
      context.spstRef.send({
        type: "LISTEN",
      }),
  },
}).createMachine({
  context: ({ spawn }) => ({
    spstRef: spawn(speechstate, { input: settings }),
    lastResult: null,
  }),
  id: "DM",
  initial: "Prepare",
  states: {
    Prepare: {
      entry: ({ context }) => context.spstRef.send({ type: "PREPARE" }),
      on: { ASRTTS_READY: "WaitToStart" },
    },
    WaitToStart: {
      on: { CLICK: "Greeting" },
    },
    Greeting: {
      initial: "Prompt",
      on: {
        LISTEN_COMPLETE: [
          {
            target: "CheckGrammar",
            guard: ({ context }) => !!context.lastResult,
          },
          { target: ".NoInput" },
        ],
      },
      states: {
        Prompt: {
          entry: { type: "spst.speak", params: { utterance: `Hello world!` } },
          on: { SPEAK_COMPLETE: "Ask" },
        },
        NoInput: {
          entry: {
            type: "spst.speak",
            params: { utterance: `I can't hear you!` },
          },
          on: { SPEAK_COMPLETE: "Ask" },
        },
        Ask: {
          entry: { type: "spst.listen" },
          on: {
            RECOGNISED: {
              actions: assign(({ event }) => {
                return { lastResult: event.value };
              }),
            },
            ASR_NOINPUT: {
              actions: assign({ lastResult: null }),
            },
          },
        },
      },
    },
    CheckGrammar: {
      entry: {
        type: "spst.speak",
        params: ({ context }) => ({
          utterance: `You just said: ${context.lastResult![0].utterance}. And it ${
            isInGrammar(context.lastResult![0].utterance) ? "is" : "is not"
          } in the grammar.`,
        }),
      },
      on: { SPEAK_COMPLETE: "Done" },
    },
    Done: {
      on: {
        CLICK: "Greeting",
      },
    },
  },
});

const dmActor = createActor(dmMachine, {
  inspect: inspector.inspect,
}).start();

dmActor.subscribe((state) => {
  console.group("State update");
  console.log("State value:", state.value);
  console.log("State context:", state.context);
  console.groupEnd();
});

export function setupButton(element: HTMLButtonElement) {
  element.addEventListener("click", () => {
    dmActor.send({ type: "CLICK" });
  });
  dmActor.subscribe((snapshot) => {
    const meta: { view?: string } = Object.values(
      snapshot.context.spstRef.getSnapshot().getMeta(),
    )[0] || {
      view: undefined,
    };
    element.innerHTML = `${meta.view}`;
  });
}
```

Replace `YOUR_REGION` with your Azure region (e.g., `northeurope`, `swedencentral` ...). Check two places:
1. In the `endpoint` URL
2. In the `azureRegion` setting

### 4.4 Create `main.ts`

Same location with:

```typescript
import "./style.css";
import { setupButton } from "./dm.ts";

document.querySelector<HTMLDivElement>("#app")!.innerHTML = `
  <div>
    <div class="card">
      <button id="counter" type="button"></button>
    </div>
  </div>
`;

setupButton(document.querySelector<HTMLButtonElement>("#counter")!);
```

### 4.5 Create `style.css`

Same location:

```css
:root {
  font-family: Inter, system-ui, Avenir, Helvetica, Arial, sans-serif;
  line-height: 1.5;
  font-weight: 400;
  color-scheme: light dark;
  color: rgba(255, 255, 255, 0.87);
  background-color: #242424;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

a {
  font-weight: 500;
  color: #646cff;
  text-decoration: inherit;
}

a:hover {
  color: #535bf2;
}

body {
  margin: 0;
  display: flex;
  place-items: center;
  min-width: 320px;
  min-height: 100vh;
}

h1 {
  font-size: 3.2em;
  line-height: 1.1;
}

#app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
}

.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}

.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}

.logo.vanilla:hover {
  filter: drop-shadow(0 0 2em #3178c6aa);
}

.card {
  padding: 2em;
}

.read-the-docs {
  color: #888;
}

button {
  border-radius: 8px;
  border: 1px solid transparent;
  padding: 0.6em 1.2em;
  font-size: 1em;
  font-weight: 500;
  font-family: inherit;
  background-color: #1a1a1a;
  cursor: pointer;
  transition: border-color 0.25s;
}

button:hover {
  border-color: #646cff;
}

button:focus,
button:focus-visible {
  outline: 4px auto -webkit-focus-ring-color;
}

@media (prefers-color-scheme: light) {
  :root {
    color: #213547;
    background-color: #ffffff;
  }
  a:hover {
    color: #747bff;
  }
  button {
    background-color: #f9f9f9;
  }
}
```

## Step 5: Check Your File System

Navigate back to the project root:
```bash
cd ..
```

Your `src` directory should now have these files:
```
src/
├── azure.ts
├── dm.ts
├── main.ts
├── style.css
├── types.ts
```

## Step 6: Run the Development Server

```bash
npm run dev
```

O + Enter (to quickly open it in your default browser)

Did it work?

## Error handling:

### Error: `The requested module does not provide an export named 'AnyActorRef'`

If you see this error for some reason Vite carrying over cache from your previous attempts, try and clear Vite's cache:
```bash
rm -rf node_modules/.vite
npm run dev
```

### Package installation issues

If you encounter dependency conflicts:
```bash
rm -rf node_modules package-lock.json
npm install
```
