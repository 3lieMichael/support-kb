<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Problem = 'parser' | 'hint' | 'one-day' | 'unplaced';

  // tickets: problem → link to the ticket filed for it.
  let { ticket, tickets = {} }: { ticket: number; tickets?: Partial<Record<Problem, string>> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  // Counted the desk's way (causes.ts): one-day moves stay under the cause they are filed by.
  const rows = $derived.by(() => {
    const row = (problem: Problem, rs: Report[], note = '') => {
      const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        problem,
        note,
        count: rs.length,
        tenants: new Set(rs.map((r) => r.tenant)).size,
        channels: [...new Set(rs.map((r) => r.channel))].sort().join(', '),
        seen: `${days[0]} → ${days[days.length - 1]}`,
      };
    };
    const oneDay = merged.filter((r) => notes.oneDayMove.includes(r.id));
    return [
      ...groupByCause(merged, tenantOf).map((g) => row(g.cause, g.reports)),
      ...(oneDay.length ? [row('one-day', oneDay, 'also counted under the cause they are filed by')] : []),
    ];
  });
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong> {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>came in by</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr>
        <td><code>{r.problem}</code></td>
        <td>{r.count}{#if r.note}<br /><small>{r.note}</small>{/if}</td>
        <td>{r.tenants}</td>
        <td>{r.channels}</td>
        <td>{r.seen}</td>
        <td>{#if tickets[r.problem]}<a href={tickets[r.problem]}>{tickets[r.problem]?.replace(/^.*\//, '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  small { color: var(--muted); }
</style>
