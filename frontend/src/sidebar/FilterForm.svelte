<script lang="ts">
  import AutocompleteInput from "../AutocompleteInput.svelte";
  import { _ } from "../i18n.ts";
  import { escape_for_regex } from "../lib/regex.ts";
  import { router, set_query_param } from "../router.ts";
  import {
    account_filter,
    fql_filter,
    time_filter,
  } from "../stores/filters.ts";
  import { accounts, links, payees, tags, years } from "../stores/index.ts";
  import { timeFormat } from "d3-time-format";

  let fql_filter_suggestions = $derived([
    ...$tags.map((tag) => `#${tag}`),
    ...$links.map((link) => `^${link}`),
    ...$payees.map((payee) => `payee:"${escape_for_regex(payee)}"`),
  ]);

  function valueExtractor(value: string, input: HTMLInputElement) {
    const match = /\S*$/.exec(
      value.slice(0, input.selectionStart ?? undefined),
    );
    return match?.[0] ?? value;
  }
  function valueSelector(value: string, input: HTMLInputElement) {
    const selectionStart = input.selectionStart ?? 0;
    const match = /\S*$/.exec(input.value.slice(0, selectionStart));
    const matchLength = match?.[0]?.length;
    return matchLength !== undefined
      ? `${input.value.slice(
          0,
          selectionStart - matchLength,
        )}${value}${input.value.slice(selectionStart)}`
      : value;
  }

  let account_filter_value = $state("");
  let fql_filter_value = $state("");
  let time_filter_value = $state("");
  account_filter.subscribe((v) => {
    account_filter_value = v;
  });
  fql_filter.subscribe((v) => {
    fql_filter_value = v;
  });
  time_filter.subscribe((v) => {
    time_filter_value = v;
  });

  /** Set the target we want to navigate to to avoid duplicate navigation. */
  let target: URL | null = null;

  /**
   * Submit the filter form.
   *
   * This is called on all the three possible events (blur, select, enter)
   * and also on the form submit. Having the listener on:enter would
   * theoretically be unnecessary (as the form would also be submitted) but
   * it seems to work around a Safari bug, see #809 and #1528.
   */
  function submit() {
    const url = new URL(router.current);
    set_query_param(url, "account", account_filter_value);
    set_query_param(url, "filter", fql_filter_value);
    set_query_param(url, "time", time_filter_value);
    if (url.href !== router.current.href) {
      target = url;
      setTimeout(() => {
        if (target) {
          router.navigate(target);
          target = null;
        }
      });
    }
  }

  const next_period = () => {
    const params = new URLSearchParams(window.location.search);
    const timeParam = params.get('time');
    
    if (!timeParam) return;

    if (timeParam.match(/^\d{4}-\d{2}$/)) {
      const currentDate = new Date(timeParam + '-01');
      const nextDate = new Date(currentDate.getTime() + (32 * 24 * 60 * 60 * 1000));
      const nextTime = timeFormat("%Y-%m")(nextDate);
      time_filter.set(nextTime);
    } else if (timeParam.match(/^\d{4}$/)) {
      const nextTime = (parseInt(timeParam) + 1).toString();
      time_filter.set(nextTime);
    } else if (timeParam.match(/^\d{4}-Q[1-4]$/)) {
      // 2024-Q1 -> 2024-Q2, 2024-Q4 -> 2025-Q1
      const currentQuarter = parseInt(timeParam.slice(-1));
      const nextQuarter = currentQuarter + 1 > 4 ? 1 : currentQuarter + 1;
      const currentYear = parseInt(timeParam.slice(0, 4));
      const nextYear = currentYear + (nextQuarter === 1 ? 1 : 0);
      const nextTime = `${nextYear}-Q${nextQuarter}`;
      time_filter.set(nextTime);
    }
  }

  const previous_period = () => {
    const params = new URLSearchParams(window.location.search);
    const timeParam = params.get('time');
    
    if (!timeParam) return;
    
    if (timeParam.match(/^\d{4}-\d{2}$/)) {
      const currentDate = new Date(timeParam + '-01');
      const nextDate = new Date(currentDate.getTime() - (5 * 24 * 60 * 60 * 1000));
      const nextTime = timeFormat("%Y-%m")(nextDate);
      time_filter.set(nextTime);
    } else if (timeParam.match(/^\d{4}$/)) {
      const nextTime = (parseInt(timeParam) - 1).toString();
      time_filter.set(nextTime);
    } else if (timeParam.match(/^\d{4}-Q[1-4]$/)) {
      // 2024-Q1 -> 2023-Q4, 2024-Q4 -> 2024-Q3
      const currentQuarter = parseInt(timeParam.slice(-1));
      const previousQuarter = currentQuarter === 1 ? 4 : currentQuarter - 1;
      const currentYear = parseInt(timeParam.slice(0, 4));
      const previousYear = currentYear - (previousQuarter === 4 ? 1 : 0);
      const previousTime = `${previousYear}-Q${previousQuarter}`;
      time_filter.set(previousTime);
    }
  }
</script>

<form
  class="flex-row"
  onsubmit={(ev) => {
    ev.preventDefault();
    submit();
  }}
>
  <button type="button" onclick={previous_period} class="nav-button">←</button>
  <button type="button" onclick={next_period} class="nav-button">→</button>
  <AutocompleteInput
    bind:value={time_filter_value}
    placeholder={_("Time")}
    suggestions={$years}
    key="f t"
    clearButton={true}
    setSize={true}
    onBlur={submit}
    onSelect={submit}
    onEnter={submit}
  />
  <AutocompleteInput
    bind:value={account_filter_value}
    placeholder={_("Account")}
    suggestions={$accounts}
    key="f a"
    clearButton={true}
    setSize={true}
    onBlur={submit}
    onSelect={submit}
    onEnter={submit}
  />
  <AutocompleteInput
    bind:value={fql_filter_value}
    placeholder={_("Filter by tag, payee, …")}
    suggestions={fql_filter_suggestions}
    key="f f"
    clearButton={true}
    setSize={true}
    {valueExtractor}
    {valueSelector}
    onBlur={submit}
    onSelect={submit}
    onEnter={submit}
  />
  <!-- svelte-ignore a11y_consider_explicit_label -->
  <button type="submit"></button>
</form>

<style>
  form {
    color: var(--text-color);

    --placeholder-color: var(--header-placeholder-color);
    --placeholder-background: var(--header-placeholder-background);
    --input-padding: 8px 25px 8px 10px;
  }

  form > :global(span) {
    max-width: 18rem;
  }

  form :global(input) {
    outline: none;
    background-color: var(--background);
    border: 0;
  }

  form :global([type="text"]:focus) {
    background-color: var(--background);
  }

  form :global(.nav-button) {
    padding: 0.5em;
    background-color: var(--placeholder-background);
    border: 0;
    color: var(--header-placeholder-color);
    cursor: pointer;
    font-weight: bold;
  }

  form :global(.nav-button):hover {
    background-color: var(--background);
  }

  [type="submit"] {
    display: none;
  }

  @media print {
    form {
      --input-padding: 8px 10px;
    }

    form :global(input):placeholder-shown {
      display: none;
    }
  }
</style>
