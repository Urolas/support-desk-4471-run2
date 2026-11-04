<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. `parser` and `hint` come from causes.ts and count as the
  // desk's CauseTable does. Reports the desk read as a one-day move (desk-notes.json) get a row of
  // their own as well: a day/month swap cannot move a date by one day. They stay counted where the
  // desk filed them, so the rows add up to more than the ticket.
  let { ticket, issues }: { ticket: number; issues: Record<string, string> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rows = $derived(
    [...Map.groupBy(merged, (r) => causeOf(r, tenantOf(r)) ?? 'unplaced').entries(), ['one-day', oneDay] as const]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        const tagged = rs.filter((r) => r.meta.intakeTag === problem).length;
        return { problem, n: rs.length, tagged, tenants: new Set(rs.map((r) => r.tenant)).size, first: days[0], last: days.at(-1) };
      })
      .sort((a, b) => b.n - a.n),
  );
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems ·
  {(reports as Report[]).length - merged.length} repeated rows in the export dropped
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake tag agrees</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td>{r.n}</td>
        <td>{r.tagged} of {r.n}</td>
        <td>{r.tenants}</td>
        <td>{r.first} → {r.last}</td>
        <td>{#if issues[r.problem]}<a href={issues[r.problem]}>#{issues[r.problem].split('/').at(-1)}</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
