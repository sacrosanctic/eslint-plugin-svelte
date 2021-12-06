<script>
  import { onDestroy, onMount } from "svelte"
  import ESLintEditor from "../eslint/ESLintEditor.svelte"
  import RulesSettings from "../eslint/RulesSettings.svelte"
  import { deserializeState, serializeState } from "../eslint/scripts/state"
  import {
    DEFAULT_RULES_CONFIG,
    getURL,
    createLinter,
  } from "../eslint/scripts/linter.js"

  const linter = createLinter()

  const DEFAULT_CODE =
    `<!-- Welcome to @ota-meshi/eslint-plugin-svelte -->
<script>
  let a = 1;
  let b = 2;
  // let c = 2;
  let string = \`this string contains some <strong>HTML!!!</strong>\`;
  let user = {
    firstname: 'Ada',
    lastname: 'Lovelace'
  };
    let current = 'foo';
<` +
    `/script>

<input
  type="number"
  bind:value={a}>
<input
    type="number"
  bind:value={b}>
<input
  type="number"
  bind:value={c}>
<p>{a} + {b} + {c} = {a + b + c}</p>

<p>{@html string}</p>

<input bind:value={user.firstname}>
<input bind:value={user.lastname}>

{@debug user}

<h1>Hello {user.firstname}!</h1>

{#if a}
  <div>foo</div>
{:else if b}
  <div>bar</div>
{:else if b}
  <div>baz</div>
{/if}

<button
  type=button
  class={current === 'foo' ? 'selected' : ''}
  on:click="{() => current = 'foo'}"
>foo</button>
`

  const state = deserializeState(
    (typeof window !== "undefined" && window.location.hash.slice(1)) || "",
  )
  let code = state.code || DEFAULT_CODE
  let rules = state.rules || Object.assign({}, DEFAULT_RULES_CONFIG)
  let messages = []
  let time = ""
  let options = {
    filename: "example.svelte",
  }

  $: serializedString = (() => {
    const serializeCode = DEFAULT_CODE === code ? undefined : code
    const serializeRules = equalsRules(DEFAULT_RULES_CONFIG, rules)
      ? undefined
      : rules
    return serializeState({
      code: serializeCode,
      rules: serializeRules,
    })
  })()
  $: {
    if (typeof window !== "undefined") {
      window.location.replace(`#${serializedString}`)
    }
  }
  onMount(() => {
    if (typeof window !== "undefined") {
      window.addEventListener("hashchange", onUrlHashChange)
    }
  })
  onDestroy(() => {
    if (typeof window !== "undefined") {
      window.removeEventListener("hashchange", onUrlHashChange)
    }
  })
  function onLintedResult(evt) {
    messages = evt.detail.messages
    time = `${evt.detail.time}ms`
  }
  function onUrlHashChange() {
    const newSerializedString =
      (typeof window !== "undefined" && window.location.hash.slice(1)) || ""
    if (newSerializedString !== serializedString) {
      const state = deserializeState(newSerializedString)
      code = state.code || DEFAULT_CODE
      rules = state.rules || Object.assign({}, DEFAULT_RULES_CONFIG)
    }
  }

  /** */
  function equalsRules(a, b) {
    const akeys = Object.keys(a).filter((k) => a[k] !== "off")
    const bkeys = Object.keys(b).filter((k) => b[k] !== "off")
    if (akeys.length !== bkeys.length) {
      return false
    }

    for (const k of akeys) {
      if (a[k] !== b[k]) {
        return false
      }
    }
    return true
  }
</script>

<div class="playground-root">
  <div class="playground-tools">
    <span style="margin-left: 16px">{time}</span>
  </div>
  <div class="playground-content">
    <RulesSettings bind:rules class="rules-settings" />
    <div class="editor-content">
      <ESLintEditor
        {linter}
        bind:code
        config={{
          parser: "svelte-eslint-parser",
          parserOptions: {
            ecmaVersion: 2020,
            sourceType: "module",
          },
          rules,
          env: {
            browser: true,
            es2021: true,
          },
        }}
        {options}
        class="eslint-playground"
        on:result={onLintedResult}
      />
      <div class="messages">
        <ol>
          {#each messages as msg, i (`${msg.line}:${msg.column}:${msg.ruleId}@${i}`)}
            <li class="message">
              [{msg.line}:{msg.column}]:
              {msg.message} (<a href={getURL(msg.ruleId)} target="_blank">
                {msg.ruleId}
              </a>)
            </li>
          {/each}
        </ol>
      </div>
    </div>
  </div>
</div>

<style>
  .playground-root {
    height: calc(100vh - 180px);
  }
  .playground-tools {
    height: 24px;
    text-align: end;
  }
  .playground-content {
    display: flex;
    flex-wrap: wrap;
    height: calc(100% - 16px);
    border: 1px solid #cfd4db;
    background-color: #282c34;
    color: #f8c555;
  }

  .playground-content > .editor-content {
    height: 100%;
    flex: 1;
    display: flex;
    flex-direction: column;
    border-left: 1px solid #cfd4db;
    min-width: 1px;
  }

  .playground-content > .editor-content > .messages {
    height: 30%;
    width: 100%;
    overflow: auto;
    box-sizing: border-box;
    border-top: 1px solid #cfd4db;
    padding: 8px;
    font-size: 12px;
  }
</style>